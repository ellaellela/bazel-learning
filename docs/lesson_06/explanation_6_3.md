# Bazel Fundamentals - Level 6.3: Transitive Aggregation (Explanations)

## 1. The Solution

Here is the completed code for `my_rules.bzl`:

```python
SbomInfo = provider(
    doc = "Holds a depset of target names",
    fields = ["target_names"] 
)

def _sbom_collector_impl(ctx):
    # 1. Collect the transitive depsets from our dependencies
    transitive_sets = []
    for dep in ctx.attr.deps:
        if SbomInfo in dep:
            # Extract the depset and append it to our list of pointers
            transitive_sets.append(dep[SbomInfo].target_names)
            
    # 2. Create OUR depset
    my_depset = depset(
        direct = [ctx.label.name],      # Our own data
        transitive = transitive_sets,   # Pointers to downstream data
    )
    
    # 3. Flatten the depset to write it to a file
    out_file = ctx.actions.declare_file(ctx.label.name + "_sbom.txt")
    
    # .to_list() forces Bazel to resolve all pointers and flatten the graph
    flat_list = my_depset.to_list()
    
    ctx.actions.write(
        output = out_file,
        content = "\n".join(flat_list),
    )
    
    # 4. Return the provider
    return [
        DefaultInfo(files = depset([out_file])),
        SbomInfo(target_names = my_depset) 
    ]

sbom_collector = rule(
    implementation = _sbom_collector_impl,
    attrs = {
        "deps": attr.label_list(), 
    },
)
```

---

## 2. DevInfra Theory: Why Depsets are Magic

Imagine `lib_c` depends on `lib_a` and `lib_b`. 
Then, `final_binary` depends on `lib_c`. 

If we used standard Python lists, the memory would look like this:
1. `lib_a` creates a list: `["lib_a"]`
2. `lib_b` creates a list: `["lib_b"]`
3. `lib_c` copies both lists into its own: `["lib_c", "lib_a", "lib_b"]`
4. `final_binary` copies `lib_c`'s list: `["final_binary", "lib_c", "lib_a", "lib_b"]`

Notice how `"lib_a"` was copied into memory **three separate times**? In a graph with 100,000 targets, this exponential copying consumes gigabytes of RAM and crashes the JVM.

### The Depset Pointer System

With depsets, Bazel does not copy data. It just makes pointers.
1. `lib_a` creates a depset: `(direct: ["lib_a"])`
2. `lib_b` creates a depset: `(direct: ["lib_b"])`
3. `lib_c` creates a depset: `(direct: ["lib_c"], transitive: [pointer_to_a, pointer_to_b])`
4. `final_binary` creates a depset: `(direct: ["final_binary"], transitive: [pointer_to_c])`

The string `"lib_a"` only exists in memory **once**. 

### `to_list()` (The Expensive Operation)
Because depsets are graphs, you cannot iterate over them directly (`for item in my_depset:` will crash). 

When you actually need the raw strings (e.g., to write them to a file or pass them to a bash script), you must call `.to_list()`. This tells Bazel to traverse the pointers and flatten it into a standard list. 

**The Golden Rule of DevInfra:** Keep your data in depsets as long as possible. Only call `.to_list()` at the very top of the graph (like in the final binary or a report generation step) to avoid blowing up memory during the Analysis Phase.

---

## 3. Depset Deep Dive: Flexibility & Superpowers

As you design more complex graphs, you will use different "shapes" of depsets. Neither `direct` nor `transitive` are mandatory. You mix and match them based on the target's role:

1. **The Leaf Node (Only `direct`):** The target has no dependencies.
   `depset(direct = [ctx.label.name])`
2. **The Aggregator (Only `transitive`):** The target generates no files/data itself, it only groups other targets (like an `alias` or umbrella rule).
   `depset(transitive = [dep.target_names for dep in ctx.attr.deps])`
3. **The Standard Node (Both):** The target generates its own data AND passes along dependencies.
4. **The Null Node (Empty):** Used as a safe fallback when a target has no data.
   `depset()`

### The Hidden Powers of `.to_list()`

When you finally call `.to_list()` at the top of the graph to flatten the pointers, Bazel does two massive favors for you behind the scenes:

**1. Automatic Deduplication (The Diamond Problem)**
In a real graph, Target C might depend on A and B, but B *also* depends on A. The pointer to A exists multiple times. When you call `.to_list()`, Bazel automatically deduplicates the output. The resulting Python list will only contain A exactly once. You never have to write custom `if item not in list:` logic.

**2. Topological Ordering (The C++ Linker Saver)**
For C++ linkers, the order of flags and object files is strictly enforced (dependencies must come *after* the targets that need them, or vice versa depending on the linker). By default, `.to_list()` flattens the graph using a postorder traversal. However, you can explicitly tell the depset how it should be sorted when flattened:

```python
my_depset = depset(
    direct = ["my_flag"],
    transitive = [...],
    order = "topological" # Guarantees strict dependency ordering
)
```