# Bazel Fundamentals - Level 6.2: The Consumer (Explanations)

## 1. The Solution

Here is the completed code for the consumer rule in `my_rules.bzl`:

```python
def _metadata_report_impl(ctx):
    # 1. Grab the dependency target
    dep_target = ctx.attr.dep
    
    # 2. Extract the data safely
    if TeamMetadataInfo not in dep_target:
        fail("The dependency must provide TeamMetadataInfo")
        
    provider_data = dep_target[TeamMetadataInfo]
    owner_str = provider_data.owner
    version_str = provider_data.version
    
    # 3. Generate the report
    out_file = ctx.actions.declare_file(ctx.label.name + "_report.txt")
    
    ctx.actions.write(
        output = out_file,
        content = "Audit Report: Target is owned by {} at version {}".format(owner_str, version_str),
    )
    
    return [DefaultInfo(files = depset([out_file]))]

metadata_report = rule(
    implementation = _metadata_report_impl,
    attrs = {
        # Using attr.label means this target depends on another target in the graph
        "dep": attr.label(mandatory = True),
    },
)
```

---

## 2. DevInfra Theory: Safe Graph Traversal

In Level 6.1, we talked about how a rule's implementation function cannot see inside other rules. The **only** way they communicate is through the Provider objects returned at the end of the Analysis Phase.

When you use `dep_target = ctx.attr.dep`, you are not getting the Python source code of the dependency. You are getting a `Target` object, which is basically just a container holding all the Providers that rule returned.

### The `fail()` Function
```python
if TeamMetadataInfo not in dep_target:
    fail("The dependency must provide TeamMetadataInfo")
```
Because Starlark is dynamically typed, a user could accidentally pass a generic `sh_library` target into your `dep` attribute. If you blindly tried to extract `dep_target[TeamMetadataInfo]`, the build would crash with a cryptic stack trace. 

Using the `in` operator combined with `fail()` allows you to catch architecture errors gracefully during the Analysis Phase. The `fail()` function immediately stops the build and prints your custom error message to the developer's terminal. 

### Provider Extraction
```python
provider_data = dep_target[TeamMetadataInfo]
```
This is the exact moment the graph connects. You are pulling the custom struct out of the dependency's return list and loading it into the parent rule's local variables. From there, you can use those strings to configure bash actions, write files, or pass them even higher up the graph.

This is exactly how a `cc_binary` (the consumer) extracts the `#include` paths from a `cc_library` (the exporter).