# Bazel Fundamentals - Level 9.4: IDE Integration (Problem Statement)

## The Scenario
Your C++ developers are complaining. They are using VS Code with AI tools like Continue and Roo Code, but their editors are completely covered in red squiggly lines. The IDEs do not know where the headers are, and they don't know what preprocessor `#define` flags Bazel is using.

You need to write a DevInfra tool that extracts the C++ compilation context (`CcInfo`) from every target in the graph, formats it into a JSON array, and outputs a single, master `compile_commands.json` file at the root of the workspace.

## DevInfra Concepts: The Rule-Aspect Handshake

Up until now, you injected Aspects directly from the command line (`--aspects=...`). But for a deliverable file like `compile_commands.json`, we want developers to just type a normal build command: `bazel build //:compdb`. 

To do this, you will combine a Rule and an Aspect.
1. **The Aspect:** Traverses the graph, extracts `CcInfo`, formats a JSON object for each `.cpp` file, and bubbles the files up (exactly like Level 9.3).
2. **The Rule:** Acts as the consumer. It defines an attribute that explicitly invokes the Aspect on a target. It then extracts the aggregated files from the Aspect's custom provider, and executes a single bash action to stitch them all together.

```python
my_rule = rule(
    implementation = _my_rule_impl,
    attrs = {
        # The Rule-Aspect Handshake! 
        # When Bazel analyzes whatever is passed into "target", 
        # it will automatically apply `my_aspect` to it first.
        "target": attr.label(aspects = [my_aspect]) 
    }
)
```

---

## The Task

We will create a new file called `compdb.bzl`. 

### 1. The Aspect & Rule (`compdb.bzl`)
Copy this scaffolding and complete the `TODO`s. I have written the complex JSON string formatting and the bash concatenation for you, so you can focus entirely on the Starlark wiring.

```python
CompDbInfo = provider(fields = ["fragments"])

# ==========================================
# 1. THE ASPECT (The Harvester)
# ==========================================
def _compdb_aspect_impl(target, ctx):
    if CcInfo not in target:
        return []

    fragments = []
    cc_ctx = target[CcInfo].compilation_context
    
    # Extract the flags from Bazel's C++ brain
    defines = ["-D" + d for d in cc_ctx.defines.to_list()]
    flags_str = " ".join(defines)

    # If the target has source files, generate a compile command for each one
    if hasattr(ctx.rule.attr, "srcs"):
        for src in ctx.rule.files.srcs:
            if src.extension in ["c", "cc", "cpp", "cxx"]:
                json_block = """
                {{
                    "directory": "__EXEC_ROOT__",
                    "command": "gcc {flags} -c {file}",
                    "file": "{file}"
                }}""".format(flags = flags_str, file = src.path)
                fragments.append(json_block)

    out_file = ctx.actions.declare_file(target.label.name + "_compdb.txt")
    
    # Write this target's fragments (joined by commas)
    ctx.actions.write(out_file, ",".join(fragments))

    # Aggregation (Exactly like 9.3)
    transitive_depsets = []
    if hasattr(ctx.rule.attr, "deps"):
        transitive_depsets = [
            dep[CompDbInfo].fragments 
            for dep in ctx.rule.attr.deps 
            if CompDbInfo in dep
        ]
        
    # TODO: Create a depset of files containing `out_file` and `transitive_depsets`.
    # Assign it to a variable named `all_fragments`.
    
    # TODO: Return your CompDbInfo provider, passing `all_fragments` to the `fragments` field.

compdb_aspect = aspect(
    implementation = _compdb_aspect_impl,
    attr_aspects = ["deps"],
)

# ==========================================
# 2. THE RULE (The Stitcher)
# ==========================================
def _compdb_rule_impl(ctx):
    out_file = ctx.actions.declare_file("compile_commands.json")
    
    # TODO: Extract the depset of files from the target.
    # Look at ctx.attr.target. Reach into it, grab the CompDbInfo provider, and get the `fragments` field.
    # Assign it to `fragment_depset`.
    
    # We use an Args object (from Level 7!) to pass thousands of files to Bash safely
    args = ctx.actions.args()
    args.add_all(fragment_depset)
    
    # The bash script to wrap the fragments in [ ] and clean up trailing commas
    bash_script = """
    echo "[" > {out}
    cat $@ >> {out}
    sed -i.bak 's/,,/,/g' {out}  # Clean up empty commas
    echo "]" >> {out}
    """.format(out = out_file.path)

    ctx.actions.run_shell(
        inputs = fragment_depset,
        outputs = [out_file],
        command = bash_script,
        arguments = [args]
    )
    
    return [DefaultInfo(files = depset([out_file]))]

compdb_generator = rule(
    implementation = _compdb_rule_impl,
    attrs = {
        # TODO: Define the "target" attribute.
        # It should be an attr.label(). 
        # Inside the label, use the `aspects = [...]` parameter to attach your `compdb_aspect`!
    }
)
```

### 2. The Targets (`BUILD`)
To test this, let's add some actual C++ source files to our targets so the aspect has something to format. Create dummy files `math.cpp`, `network.cpp`, and `main.cpp` in your directory (they can be completely empty!).

Update your `BUILD` file:
```python
load("//:compdb.bzl", "compdb_generator")

cc_library(
    name = "math_lib",
    srcs = ["math.cpp"],
    defines = ["MATH_OPTIMIZED=1"], # A flag for the aspect to find!
)

cc_library(
    name = "network_lib",
    srcs = ["network.cpp"],
    defines = ["USE_TCP=1"],
)

cc_binary(
    name = "app_server",
    srcs = ["main.cpp"],
    deps = [":math_lib", ":network_lib"],
)

# The DevInfra Tool Target
compdb_generator(
    name = "gen_compdb",
    target = ":app_server", # Pointing it at the top of the graph
)
```

### 3. Verification
Run the standard build command against your custom generator rule:
```bash
bazel build //:gen_compdb
```

Then, inspect the final artifact:
```bash
cat bazel-bin/compile_commands.json
```

**Success Criteria:**
You should see a perfectly formatted JSON array containing three objects (`math.cpp`, `network.cpp`, and `main.cpp`). Crucially, look at the `command` strings! You should see your `-D` flags successfully extracted from the `cc_library` attributes and injected into the GCC command line!