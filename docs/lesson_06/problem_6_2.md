# Bazel Fundamentals - Level 6.2: The Consumer (Problem Statement)

## The Scenario
Now that `core_network_lib` is broadcasting its ownership data up the graph, you need a rule that can actually read it. 

Your DevOps lead asked you to write a `metadata_report` rule. This rule will take a single dependency, read the custom provider from that dependency, and generate a text file that acts as an audit report.

## DevInfra Concepts: Consuming the Provider

When Target B depends on Target A via an attribute (like `dep = "//:core_network_lib"`), Target B's implementation function can look at the providers Target A returned.

1. **Accessing the Target Object**
   You access the dependency just like any other attribute: `target = ctx.attr.dep`.

2. **Checking for the Provider**
   Because Bazel is dynamically typed (in Starlark), you should always check if the dependency actually *has* the provider before trying to read it. You do this using the `in` keyword:
   ```python
   if TeamMetadataInfo in target:
       # It's safe to read
   ```

3. **Extracting the Data**
   You extract the provider using bracket notation, and then access your custom fields via standard dot notation:
   ```python
   provider_data = target[TeamMetadataInfo]
   print(provider_data.owner)
   print(provider_data.version)
   ```

---

## The Task

### 1. The Rule (`my_rules.bzl`)
Add this new rule to the bottom of your `my_rules.bzl` file and fill in the `TODO`s.

```python
def _metadata_report_impl(ctx):
    # 1. Grab the dependency target
    dep_target = ctx.attr.dep
    
    # 2. Extract the data
    # TODO: Check if TeamMetadataInfo is in the dep_target.
    # TODO: If it is, extract the 'owner' and 'version' into local variables.
    # TODO: If it isn't, fail the build by calling: fail("Dependency must provide TeamMetadataInfo")
    
    # 3. Generate the report
    out_file = ctx.actions.declare_file(ctx.label.name + "_report.txt")
    
    # TODO: Write an action (ctx.actions.write) that creates a string formatted like:
    # "Audit Report: Target is owned by [OWNER] at version [VERSION]"
    # and saves it to out_file.
    
    return [DefaultInfo(files = depset([out_file]))]

metadata_report = rule(
    implementation = _metadata_report_impl,
    attrs = {
        # Notice we use attr.label() to accept a dependency target
        "dep": attr.label(mandatory = True),
    },
)
```

### 2. The Target (`BUILD`)
Add your new consumer to the `BUILD` file. It will depend directly on the rule you wrote in Level 6.1!

```python
load("//:my_rules.bzl", "metadata_exporter", "metadata_report")

# (Keep your existing metadata_exporter target)
metadata_exporter(
    name = "core_network_lib",
    owner = "infrastructure_team",
    version = "2.1.0",
)

# Add the new consumer target
metadata_report(
    name = "network_audit",
    dep = "//:core_network_lib",
)
```

### 3. Verification
Build the report and check the output!

```bash
bazel build //:network_audit
cat bazel-bin/network_audit_report.txt
```

**Success Criteria:** The text file should successfully print out `"Audit Report: Target is owned by infrastructure_team at version 2.1.0"`.