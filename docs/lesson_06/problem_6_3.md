# Bazel Fundamentals - Level 6.3: Transitive Aggregation (Problem Statement)

## The Scenario
Your DevOps team wants to generate a "Software Bill of Materials" (SBOM). They need a rule at the top of the graph to print out the names of *every single target* beneath it in the dependency tree.

If you use standard Python lists to collect this data, the build system will crash on large graphs due to O(N^2) memory copying. You must use Bazel's `depset` to aggregate the target names efficiently using pointers.

## The Task

You will write a single rule called `sbom_collector`. This rule will:
1.  Read its own name.
2.  Look at its dependencies (`deps`).
3.  Extract the `depset` from each dependency's provider.
4.  Create a *new* `depset` that contains its own name (direct) and points to the depsets of its dependencies (transitive).
5.  Return that new `depset` in the custom provider.
6.  Generate a text file that flattens the `depset` into a final list.

### 1. The Rule (`my_rules.bzl`)
Add this to `my_rules.bzl`:

```python
# 1. The Schema
SbomInfo = provider(
    doc = "Holds a depset of target names",
    fields = ["target_names"] 
)

def _sbom_collector_impl(ctx):
    # 2. Collect the transitive depsets from our dependencies
    transitive_sets = []
    for dep in ctx.attr.deps:
        if SbomInfo in dep:
            # TODO: Extract the 'target_names' depset from the dependency's SbomInfo
            # TODO: Append that depset to the 'transitive_sets' list
            pass
            
    # 3. Create OUR depset
    # direct = [ctx.label.name] (Our own name)
    # transitive = transitive_sets (Pointers to our dependencies)
    # TODO: Create the new depset and assign it to a variable called 'my_depset'
    
    # 4. Flatten the depset to write it to a file
    # You can't write a depset directly to a file. You must call .to_list() on it to 
    # force Bazel to resolve all the pointers into a flat Python list.
    out_file = ctx.actions.declare_file(ctx.label.name + "_sbom.txt")
    
    # TODO: Call my_depset.to_list() and join the strings with a newline ("\n".join(...))
    # TODO: Use ctx.actions.write to write that string to out_file
    
    # 5. Return the provider
    return [
        DefaultInfo(files = depset([out_file])),
        SbomInfo(target_names = my_depset) # Pass our depset up the graph!
    ]

sbom_collector = rule(
    implementation = _sbom_collector_impl,
    attrs = {
        # Note: attr.label_list because a target can have MULTIPLE dependencies
        "deps": attr.label_list(), 
    },
)
```

### 2. The Target (`BUILD`)
Add this tree to your `BUILD` file to test the transitive collection:

```python
load("//:my_rules.bzl", "sbom_collector")

# The bottom of the graph (No dependencies)
sbom_collector(name = "lib_a")
sbom_collector(name = "lib_b")

# The middle of the graph
sbom_collector(
    name = "lib_c",
    deps = [":lib_a", ":lib_b"]
)

# The top of the graph (The final binary)
sbom_collector(
    name = "final_binary",
    deps = [":lib_c"]
)
```

### 3. Verification
Build the top of the graph and read the SBOM file!

```bash
bazel build //:final_binary
cat bazel-bin/final_binary_sbom.txt
```

**Success Criteria:** The text file for `final_binary` should contain `final_binary`, `lib_c`, `lib_a`, and `lib_b`. Because of the `depset` pointers, `final_binary` was able to see the names of `lib_a` and `lib_b` even though it only depended directly on `lib_c`!