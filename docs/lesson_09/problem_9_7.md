# Bazel Fundamentals - Level 9.7: The Polyglot Traverse (Problem Statement)

## The Scenario
Your engineering organization is building a machine learning application. The top-level training scripts are written in Python, but for performance, they depend on core tensor libraries written in C++. 

The architecture team wants an automated tool that traces the entire dependency tree and explicitly labels which nodes are Python and which are C++. You need to write a Polyglot Aspect that bridges the language divide.

## DevInfra Concepts: Multi-Attribute Traversal & Dynamic Interrogation

Up until now, you have only traversed the `deps` attribute. But Bazel doesn't typically link C++ directly into Python `deps`. Instead, C++ shared objects and data files are often passed to Python rules via the `data` attribute. 

To bridge the language gap, your Aspect must do two things:
1. **Multi-Attribute Traversal:** It must declare `attr_aspects = ["deps", "data"]` to ensure it jumps across both types of dependency edges.
2. **Dynamic Interrogation:** Instead of instantly returning an empty list if a specific provider is missing, it must independently check for the presence of `PyInfo` and `CcInfo` in the same target, allowing it to adapt to whatever language node it lands on.

---

## The Task

Create a new file called `polyglot.bzl`.

### 1. The Aspect & Rule (`polyglot.bzl`)
Copy this scaffolding and complete the `TODO`s. 

```python
MapInfo = provider(fields = ["files"])

# ==========================================
# 1. THE ASPECT (The Polyglot Harvester)
# ==========================================
def _polyglot_aspect_impl(target, ctx):
    node_labels = []
    
    # TODO: Interrogate the Target!
    # 1. Check if PyInfo is in the target. 
    #    If it is, append "PYTHON NODE: " + target.label.name to `node_labels`.
    
    # 2. Check if CcInfo is in the target.
    #    If it is, append "C++ NODE: " + target.label.name to `node_labels`.
    
    # If the target is neither (e.g., a simple text file), prune this branch.
    if not node_labels:
        return []

    # Write the discovered language identity to a file
    out_file = ctx.actions.declare_file(target.label.name + "_map.txt")
    ctx.actions.write(out_file, "\n".join(node_labels) + "\n")

    transitive_depsets = []
    
    # Notice we must loop through multiple attributes now!
    for attr_name in ["deps", "data"]:
        if hasattr(ctx.rule.attr, attr_name):
            for dep in getattr(ctx.rule.attr, attr_name):
                if MapInfo in dep:
                    transitive_depsets.append(dep[MapInfo].files)
                    
    all_files = depset(direct = [out_file], transitive = transitive_depsets)
    return [MapInfo(files = all_files)]

polyglot_aspect = aspect(
    implementation = _polyglot_aspect_impl,
    # TODO: Tell Bazel to traverse BOTH "deps" and "data" edges!
    attr_aspects = [], 
)

# ==========================================
# 2. THE RULE (The Architecture Mapper)
# ==========================================
def _mapper_rule_impl(ctx):
    out_file = ctx.actions.declare_file(ctx.label.name + "_architecture.txt")
    fragment_depset = ctx.attr.target[MapInfo].files
    
    args = ctx.actions.args()
    args.add_all(fragment_depset)
    
    ctx.actions.run_shell(
        inputs = fragment_depset,
        outputs = [out_file],
        command = "cat $@ > " + out_file.path,
        arguments = [args]
    )
    
    return [DefaultInfo(files = depset([out_file]))]

architecture_mapper = rule(
    implementation = _mapper_rule_impl,
    attrs = {
        "target": attr.label(aspects = [polyglot_aspect]),
    }
)
```

### 2. The Cross-Language Graph (`BUILD`)
To test this, let's create a graph where Python code depends on C++ logic. (You can just use `touch wrapper.py model.py` in your terminal to create the dummy files—Bazel just needs the files to exist).

Update your `BUILD` file:
```python
load("//:polyglot.bzl", "architecture_mapper")

# 1. The Core C++ Engine
cc_library(
    name = "tensor_compute",
    srcs = ["math.cpp"],
)

# 2. The Python Bindings (Depends on C++ via data)
py_library(
    name = "tensor_wrapper",
    srcs = ["wrapper.py"],
    data = [":tensor_compute"], 
)

# 3. The Top-Level ML Application (Depends on Python wrapper via deps)
py_binary(
    name = "ml_trainer",
    srcs = ["model.py"],
    deps = [":tensor_wrapper"],
)

# 4. Our DevInfra Tool
architecture_mapper(
    name = "map_ml_app",
    target = ":ml_trainer",
)
```

### 3. Verification
Run the build targeting your architecture mapper:

```bash
bazel build //:map_ml_app
```

Then, inspect the final artifact:
```bash
cat bazel-bin/map_ml_app_architecture.txt
```

**Success Criteria:**
You should see a text file that successfully identifies `ml_trainer` and `tensor_wrapper` as Python nodes, and `tensor_compute` as a C++ node. Your Aspect successfully crossed the language boundary by jumping from a `deps` edge to a `data` edge!