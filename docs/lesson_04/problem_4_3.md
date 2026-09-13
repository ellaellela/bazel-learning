### Bonus Boss Fight: The Macro Wrapper

**The Scenario:** 
Your `inject_header` rule works perfectly, but the developer experience is currently poor. If a team has 500 JSON files they need to minify and bundle, they do not want to manually write 500 `minify_json` targets in their `BUILD` file before calling `inject_header`. 

**The Task:**
Bridge the gap between the **Loading Phase** and the **Analysis Phase**. 
Write a Starlark **Macro** (a standard Python function) that automates the creation of the rules.

1.  **The Macro Signature:** 
    Create a function in `my_rules.bzl` called `corporate_minify_bundle(name, header, srcs, out_name)`.
2.  **The Behavior:**
    *   It should iterate over the `srcs` list using a standard `for` loop.
    *   For every file, it should instantiate a `minify_json` rule dynamically.
    *   It should collect the names of all those dynamically generated targets.
    *   Finally, it should instantiate a single `inject_header` rule, passing it the collected targets.

---

#### The Implementation

Add this macro to your `my_rules.bzl` file (macros usually go at the top or bottom of the file, outside of any rule definitions):

```python
def corporate_minify_bundle(name, header, srcs, out_name):
    """A macro that minifies a list of files and injects a legal header."""
    
    # We will store the names of the generated targets here
    minified_targets = []
    
    # 1. Loop over every raw source file the user provided
    for src_file in srcs:
        # Create a unique target name (e.g., "data.json" -> "data_json_min")
        target_name = src_file.replace(".", "_") + "_min"
        
        # Instantiate your custom rule for this specific file
        minify_json(
            name = target_name,
            src = src_file,
            out_name = target_name + ".out",
        )
        
        # Add the generated label to our list
        minified_targets.append(":" + target_name)
    
    # 2. Instantiate the injector rule, passing the list of dynamically generated targets!
    inject_header(
        name = name,
        header = header,
        srcs = minified_targets,
        out_name = out_name,
    )
```

#### The `BUILD` File (The Developer Experience)

Now, your users have a perfectly clean API. They just load the macro and pass their raw files. Replace your explicit targets with this:

```python
load("//:my_rules.bzl", "corporate_minify_bundle")

corporate_minify_bundle(
    name = "my_massive_bundle",
    header = "legal.txt",
    srcs = [
        "data.json",
        "new_data.json",
    ],
    out_name = "corporate_minified_many.json",
)
```

---

### The Final Takeaway: Macro vs. Rule Architecture

When you run `bazel build //:my_massive_bundle`, you are witnessing the complete Bazel lifecycle:

1.  **The Loading Phase:** Your macro executes like a Python script. It silently writes a `minify_json` target for `data.json`, a `minify_json` target for `new_data.json`, and an `inject_header` target that depends on both.
2.  **The Analysis Phase:** Bazel reads those generated rules, analyzes their `ctx.actions`, and builds the Action Graph.
3.  **The Execution Phase:** Bazel spins up sandboxes, runs the independent minifications in parallel, and then runs the final injection script. 

This pattern—using a Loading Phase Macro to stamp out multiple Analysis Phase Rules—is the foundational architecture of almost every major language ruleset (like `rules_cc` or `rules_python`) in the Bazel ecosystem.
