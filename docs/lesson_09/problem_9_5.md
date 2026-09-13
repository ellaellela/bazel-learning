# Bazel Fundamentals - Level 9.5: Native Graph Pruning (Problem Statement)

## The Scenario
In Level 9.3, we used the "Professional Filter" inside our Starlark function:
```python
if CcInfo not in target:
    return []
```
This is logically safe, but it is **computationally expensive**. In a graph with 10,000 targets where only 1,000 are C++, Bazel still has to wake up the Starlark interpreter 10,000 times, run the function, hit the `if` statement, and return the empty list. 

You need to optimize the Analysis Phase by telling Bazel's internal Java engine to aggressively prune the graph *before* it ever invokes your Starlark code.

## DevInfra Concepts: `required_providers`

When you declare an Aspect, you can provide a list called `required_providers`. 

```python
my_aspect = aspect(
    implementation = _my_impl,
    attr_aspects = ["deps"],
    required_providers = [CcInfo] # Native Graph Pruning!
)
```

By defining this, you create a strict contract. As Bazel walks down the graph (`attr_aspects = ["deps"]`), it looks at the native providers of the dependencies. If a dependency does *not* advertise `CcInfo`, Bazel instantly aborts the traversal down that branch. It will not call your implementation function, and it will not check that dependency's children.

---

## The Task

We are going to prove that Starlark is bypassed entirely. Create a new file called `pruning.bzl`.

### 1. The Aspect (`pruning.bzl`)
Copy this scaffolding and complete the `TODO`s. We will use a `print()` statement as our tripwire. 

```python
def _pruning_aspect_impl(target, ctx):
    # Notice we removed the `if CcInfo not in target` check!
    # The print statement is our tripwire. If Starlark runs, this prints.
    print("Executing Starlark for:", target.label)
    return []

optimized_aspect = aspect(
    implementation = _pruning_aspect_impl,
    attr_aspects = ["deps"],
    # TODO: Add the required_providers parameter here.
    # Tell Bazel it should only evaluate targets that possess CcInfo.
)
```

### 2. The Mixed Graph (`BUILD`)
Let's create a graph with a mix of C++ and non-C++ targets. Add this to your `BUILD` file.

```python
# A simple text file target (Returns DefaultInfo, but NOT CcInfo)
filegroup(
    name = "docs",
    srcs = ["readme.md"], 
)

# A C++ library
cc_library(
    name = "core_logic",
    srcs = ["math.cpp"],
)

# A C++ binary that depends on both!
cc_binary(
    name = "mixed_app",
    srcs = ["main.cpp"],
    data = [":docs"],    # Non-C++ dependency
    deps = [":core_logic"], # C++ dependency
)
```
*(Note: Create a dummy `readme.md` file in your directory so the `filegroup` is valid).*

### 3. Verification
Run the optimized aspect against the top-level binary:

```bash
bazel build //:mixed_app \
  --aspects=//:my_aspects.bzl%optimized_aspect
```

**Success Criteria:**
Look closely at the terminal output. You should see Bazel print `"Executing Starlark for:"` for `//:mixed_app` and `//:core_logic`. 
You should **not** see it print for `//:docs`. This proves that Bazel's native engine saw that `docs` lacked `CcInfo` and pruned it from the execution tree before Starlark was ever invoked!