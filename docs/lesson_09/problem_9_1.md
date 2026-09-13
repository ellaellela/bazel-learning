# Bazel Fundamentals - Level 9.1: The Observer (Problem Statement)

## The Scenario
Your company has a massive, 10,000-target C++ repository. The security team needs a way to audit the dependency tree of specific binaries. 

They ask you: *"Can you give us a tool that prints out the label of every single `cc_library` and `cc_binary` that makes up a specific application?"*

You cannot go into the `BUILD` files and change 10,000 `cc_library` targets. You cannot change the rule definitions, because `cc_library` is native to Bazel. Instead, you will write an **Aspect** that attaches to the top of the graph and propagates downward, printing the name of every target it touches.

## DevInfra Concepts: The Aspect API

1. **The Aspect Implementation Function**
   Unlike a rule implementation which only takes `(ctx)`, an aspect implementation takes `(target, ctx)`.
   *   `target`: Represents the underlying rule the aspect is currently attached to. You use this to read the providers the rule generated (e.g., `target[CcInfo]`).
   *   `ctx`: The context of the aspect itself. You use this to declare files or run actions (which we will do in Level 9.2).
   
   ```python
   def _my_aspect_impl(target, ctx):
       # Do something with the target
       return [] # Aspects must return a list of providers (empty for now)
   ```

2. **The `aspect()` Definition**
   Just like `rule()`, you define an aspect using the `aspect()` function.
   The superpower of an aspect is the `attr_aspects` parameter. This tells Bazel: *"If the target I am attached to has a `deps` attribute, automatically attach a copy of this aspect to every target listed in that attribute."*
   
   ```python
   my_aspect = aspect(
       implementation = _my_aspect_impl,
       attr_aspects = ["deps"], # The magic graph-traversal parameter
   )
   ```

3. **Command-Line Injection**
   You don't need a `BUILD` file to run an aspect. You can inject it directly from the terminal using the `--aspects` flag. You provide the `.bzl` file path, a `%`, and the name of the aspect variable.

---

## The Task

### 1. The Targets (`BUILD`)
Let's create a standard C++ dependency tree in your `BUILD` file. Notice there is absolutely no mention of your custom DevInfra tooling here. This is a pure C++ graph.

```python
cc_library(
    name = "math_lib",
    srcs = [], # Empty for this test
)

cc_library(
    name = "network_lib",
    srcs = [],
)

cc_binary(
    name = "app_server",
    srcs = [],
    deps = [":math_lib", ":network_lib"],
)
```

### 2. The Aspect (`my_aspects.bzl`)
Create a new file called `my_aspects.bzl`. (We usually keep aspects separate from rules). Complete the `TODO`s.

```python
def _target_printer_impl(target, ctx):
    # 1. Print the label of the target we are currently visiting.
    # target.label gives you the exact Bazel path (e.g., //:math_lib)
    # TODO: Use the Starlark print() function to print a message like: "Observing target: [LABEL]"
    
    # 2. Return an empty list of providers. 
    # (In Level 9.2, we will return real data here).
    return []

# TODO: Define the aspect using the aspect() function.
# 1. Assign it to a variable named `target_printer`.
# 2. Set the implementation function.
# 3. Tell it to propagate down the "deps" attribute using `attr_aspects`.
```

### 3. Verification
Do **not** use `bazel build //:app_server`. Instead, run this command to inject your aspect into the build graph during the Analysis Phase:

```bash
bazel build //:app_server --aspects=//:my_aspects.bzl%target_printer
```

**Success Criteria:** 
During the Analysis Phase, your terminal should print the `print()` statements from Starlark. Because of `attr_aspects = ["deps"]`, you should see it observe `//:app_server`, and then seamlessly travel down the graph to observe `//:math_lib` and `//:network_lib`!