# Bazel Fundamentals - Level 6.1: The Schema & The Exporter (Problem Statement)

## The Scenario
In a massive enterprise monorepo, targets need to communicate abstract data to one another, not just pass files. Your DevOps team wants to track which department owns which build targets and what version they are on. 

Instead of hardcoding this into bash scripts or text files, you will create a structured Bazel **Provider** to attach this metadata directly to the dependency graph.

## DevInfra Concepts: The Provider API

1. **Defining the Schema (`provider`)**
   In Starlark, you define a custom provider globally in your `.bzl` file. It acts like a strongly-typed dictionary or struct.
   ```python
   TeamMetadataInfo = provider(
       doc = "Holds ownership and versioning data for a target",
       fields = ["owner", "version"]
   )
   ```

2. **Instantiating the Provider**
   Inside your rule's implementation function, you instantiate the provider by calling it like a constructor and passing values to the fields you defined.
   ```python
   my_data = TeamMetadataInfo(owner = "backend_team", version = "1.0.4")
   ```

3. **Returning Multiple Providers**
   Rules can return a list containing multiple providers. This is how you pass both files (`DefaultInfo`) and custom data (`TeamMetadataInfo`) simultaneously.
   ```python
   return [
       DefaultInfo(files = depset([out_file])),
       my_data
   ]
   ```

---

## The Task

### 1. The Rule (`my_rules.bzl`)
Add the following skeleton to your `my_rules.bzl` file and fill in the `TODO`s.

```python
# 1. Define the schema globally
TeamMetadataInfo = provider(
    doc = "Holds ownership and versioning data for a target",
    fields = ["owner", "version"]
)

def _metadata_exporter_impl(ctx):
    # Declare a dummy output file just so the rule has something to build
    out_file = ctx.actions.declare_file(ctx.label.name + ".txt")
    ctx.actions.write(
        output = out_file,
        content = "This target is owned by: " + ctx.attr.owner,
    )
    
    # 2. Instantiate your custom provider using ctx.attr
    # TODO: Create a TeamMetadataInfo object using ctx.attr.owner and ctx.attr.version
    
    # 3. Return BOTH DefaultInfo and your custom provider in a list
    # TODO: Return the list
    pass

metadata_exporter = rule(
    implementation = _metadata_exporter_impl,
    attrs = {
        "owner": attr.string(mandatory = True),
        "version": attr.string(default = "1.0.0"),
    },
)
```

### 2. The Target (`BUILD`)
Add this target to your `BUILD` file:

```python
load("//:my_rules.bzl", "metadata_exporter")

metadata_exporter(
    name = "core_network_lib",
    owner = "infrastructure_team",
    version = "2.1.0",
)
```

### 3. Verification
Providers exist purely in the Analysis Phase graph; they do not show up as files on your hard drive. To peek into the graph and prove your provider is attached to the target, use Bazel's `cquery` (Configured Query) command:

```bash
bazel cquery //:core_network_lib --output=starlark --starlark:expr="providers(target)"
```

**Success Criteria:** When you run the `cquery` command, your terminal should print out a Starlark dictionary showing both `DefaultInfo` and your custom `TeamMetadataInfo` with the correct string values.