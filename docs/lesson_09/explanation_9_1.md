# Bazel Fundamentals - Level 9.1: The Observer (Explanations)

## 1. The Solution

Here is the completed code for `my_aspects.bzl`:

```python
def _target_printer_impl(target, ctx):
    # Print the absolute Bazel label of the target
    print("Observing target:", target.label)
    
    # Aspects must return a list of providers (empty for now)
    return []

target_printer = aspect(
    implementation = _target_printer_impl,
    # This is the engine of the aspect. It tells Bazel to recursively 
    # attach this aspect to any target listed in the "deps" attribute.
    attr_aspects = ["deps"], 
)
```

---

## 2. DevInfra Theory: How `attr_aspects` Works

When you run `bazel build //:app_server --aspects=//:my_aspects.bzl%target_printer`, you kick off a chain reaction during the Analysis Phase.

1. **The Entry Point:** Bazel attaches `target_printer` to `//:app_server`. The implementation function runs, printing the label.
2. **The Inspection:** Bazel looks at the `aspect()` definition and sees `attr_aspects = ["deps"]`. 
3. **The Propagation:** Bazel looks at the `app_server` target and checks if it has a `deps` attribute. It does: `[":math_lib", ":network_lib"]`.
4. **The Cloning:** Bazel creates a clone of the `target_printer` aspect and attaches it to `math_lib` and `network_lib`.
5. **Recursion:** The process repeats. If `math_lib` had its own `deps`, the aspect would clone itself again and keep walking down until it hit the bottom of the graph (the leaf nodes).

### The Power of Non-Invasion
Think about the architectural implications of this. You just audited a C++ dependency graph without modifying the `cc_library` rules, without changing the `BUILD` files, and without the C++ developers even knowing your DevInfra tool exists. 

This is how enterprise security teams enforce compliance. They write Aspects that traverse the graph on CI/CD servers, ensuring every single dependency is scanned before a release.

---

## 3. Deep Dive: Implicit Dependencies (The Hidden Graph)

When you ran the Aspect, you likely saw output that you didn't explicitly write in your `BUILD` file, such as `@@rules_cc+//:empty_lib` or `@@rules_cc+//:link_extra_lib`.

Where did these come from? 

### The `BUILD` File vs. The True Graph
When a C++ developer writes a `cc_binary`, they only list the direct business logic dependencies. But behind the scenes, Bazel's native C++ rules automatically inject hidden dependencies required to make the compile work (like standard library links, wrapper scripts, and toolchain configurations). 

These are called **Implicit Dependencies**. 

### Aspect Vision
Standard tools like `bazel query` (by default) only show you the explicit graph—what the developer physically typed in the `BUILD` file. 

Aspects, however, possess "True Sight". Because they operate during the Analysis Phase, they see the fully resolved, complete dependency graph just before it is handed off to the execution sandbox. 

If you want an Aspect to *ignore* these hidden toolchain targets and only focus on the user's code, you have to write Starlark logic to filter them out. You do this by checking the target's providers or filtering based on the workspace name:

```python
def _target_printer_impl(target, ctx):
    # Filter out external repositories (like rules_cc)
    if target.label.workspace_name != "":
        return []
        
    print("Observing target:", target.label)
    return []
```

### The Professional Filter: Checking for Providers

Filtering by workspace name (`target.label.workspace_name != ""`) is a quick trick, but the most robust DevInfra pattern is filtering by **Providers**. This relies on the target's *capabilities* rather than its directory location.

You want to tell the Aspect: *"I don't care where this target lives; I only care if it actually compiles C++."* 

In Starlark, the `target` object acts like a dictionary of its providers. You can use the standard `in` operator to verify if a target possesses a specific provider, like `CcInfo`:

```python
def _target_printer_impl(target, ctx):
    # The Provider Filter: If it's not C++, silently ignore it
    if CcInfo not in target:
        return []
        
    # If we reach this line, we are guaranteed it's a C++ target
    print("Observing C++ target:", target.label)
    return []
```

Because implicit dependencies like `cc_wrapper.sh` are shell scripts, they return `DefaultInfo` but do **not** return a `CcInfo` provider. By adding this single `if` statement, your Aspect will automatically filter out wrapper scripts, text files, and toolchain definitions, leaving you with a perfectly clean audit of only the true C++ libraries.