# Bazel Fundamentals - Level 9.7: The Polyglot Traverse (Explanations)

## 1. The Solution

Here is the completed Starlark code for the Polyglot Aspect:

```python
load("@rules_python//python:defs.bzl", "PyInfo")

MapInfo = provider(fields = ["files"])

# ==========================================
# 1. THE ASPECT (The Polyglot Harvester)
# ==========================================
def _polyglot_aspect_impl(target, ctx):
    node_labels = []
    
    # 1. Dynamic Interrogation for Python
    if PyInfo in target:
        node_labels.append("PYTHON_NODE: " + target.label.name)
        
    # 2. Dynamic Interrogation for C++
    if CcInfo in target:
        node_labels.append("C++ NODE: " + target.label.name)
        
    if not node_labels:
        return []

    out_file = ctx.actions.declare_file(target.label.name + "_map.txt")
    ctx.actions.write(out_file, "\n".join(node_labels) + "\n")

    transitive_depsets = []
    
    # 3. Multi-Attribute Traversal
    for attr_name in ["deps", "data"]:
        if hasattr(ctx.rule.attr, attr_name):
            for dep in getattr(ctx.rule.attr, attr_name):
                if MapInfo in dep:
                    transitive_depsets.append(dep[MapInfo].files)
                    
    all_files = depset(direct = [out_file], transitive = transitive_depsets)
    return [MapInfo(files = all_files)]

polyglot_aspect = aspect(
    implementation = _polyglot_aspect_impl,
    # Traverse BOTH standard dependencies and runtime data payloads
    attr_aspects = ["deps", "data"], 
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

---

## 2. DevInfra Theory: Multi-Attribute Traversal

Up until now, our Aspects only traversed `attr_aspects = ["deps"]`. This works well for single-language monolithic libraries, but it fails in modern polyglot architecture. 

When a Python rule needs to execute a compiled C++ binary or load a C++ shared object, it rarely links it via `deps`. Instead, it includes it as a runtime payload via the `data` attribute. By configuring the Aspect to traverse `attr_aspects = ["deps", "data"]`, the Aspect effectively bridges the gap between different execution environments, tracing the graph identically regardless of how the edge was declared.

---

## 3. Deep Dive: Dynamic Interrogation vs. Native Pruning

In Level 9.5, we used `required_providers = [CcInfo]` to optimize the graph. However, we intentionally omitted `required_providers` in Level 9.7. Why?

`required_providers` operates as a strict boolean `AND` condition at the Java engine level. If we had defined `required_providers = [CcInfo, PyInfo]`, Bazel would have aggressively pruned *every single target* unless that target miraculously returned both C++ and Python providers simultaneously. 

When building Polyglot Aspects, you must fall back to **Dynamic Interrogation** within Starlark (`if PyInfo in target:`). This allows the Aspect to remain flexible, adapting its payload extraction logic dynamically based on the exact language identity of the node it is currently visiting.

---

## 4. Deep Dive: The OR Condition (`required_providers` List of Lists)

It is a common misconception that `required_providers` only supports strict `AND` conditions. Bazel actually supports `OR` conditions natively inside the `required_providers` attribute using a **List of Lists** syntax.

*   **The AND Condition (Flat List):** 
    `required_providers = [CcInfo, PyInfo]`
    Bazel requires the target to possess *both* `CcInfo` AND `PyInfo`. 

*   **The OR Condition (List of Lists):**
    `required_providers = [[CcInfo], [PyInfo]]`
    Bazel evaluates the outer list as an `OR` and the inner lists as `AND`s. This tells the Java engine: *"Prune this branch UNLESS the target possesses `CcInfo` OR `PyInfo`."*

**Why did we still need Dynamic Interrogation?**
Using `required_providers = [[CcInfo], [PyInfo]]` is highly recommended for Polyglot Aspects because it offloads the initial pruning of completely unrelated targets (like Java libraries or text files) to the native Java engine. 

However, you *still* must use Dynamic Interrogation (`if CcInfo in target:`) inside your Starlark `implementation` function. The native pruner only guarantees the target has *at least one* of the required providers—Starlark still has to explicitly check *which one* it actually got so it can extract the correct payload!

---

## 5. Deep Dive: Externalized Rules (Bzlmod)

Modern Bazel has been aggressively modularized. Core languages like Python and C++ have been ripped out of the monolithic binary into external modules (e.g., `rules_python`, `rules_cc`). 

This creates a new strictness in DevInfra tooling. An Aspect can no longer blindly assume `PyInfo` exists in the global namespace. You must explicitly fetch the provider from the external toolchain (`load("@rules_python//python:defs.bzl", "PyInfo")`). Furthermore, executable rules like `py_binary` enforce strict entry points. If the filename does not perfectly match the target name, Bazel demands explicit configuration via the `main` attribute (e.g., `main = "model.py"`) to ensure the execution graph is deterministic.