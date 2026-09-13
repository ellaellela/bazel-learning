# Bazel Fundamentals - Level 7.1: The Custom Compiler (Explanations)

## 1. The Solution

Here is the completed code for `my_rules.bzl`:

```python
def _hermetic_minify_impl(ctx):
    out_file = ctx.actions.declare_file(ctx.attr.out_name)
    
    # Execute the custom tool natively
    ctx.actions.run(
        executable = ctx.executable.compiler,
        inputs = [ctx.file.src],
        outputs = [out_file],
        # Notice we are just passing the paths as strings. 
        # The Python script's sys.argv will read these!
        arguments = [ctx.file.src.path, out_file.path]
    )
    
    return [DefaultInfo(files = depset([out_file]))]

hermetic_minify = rule(
    implementation = _hermetic_minify_impl,
    attrs = {
        "src": attr.label(allow_single_file = True, mandatory = True),
        "out_name": attr.string(mandatory = True),
        "compiler": attr.label(
            default = "//:json_compiler",
            executable = True,
            cfg = "exec",
        ),
    },
)
```

---

## 2. DevInfra Theory: Why `ctx.actions.run` is King

Up until now, your actions used `ctx.actions.run_shell(command = "tr -d ' \n' ...")`. 

While this works for simple scripts, it breaks the golden rule of **Hermeticity**. If Developer A is on Linux, Developer B is on a Mac, and Developer C is on Windows, the `tr` command might behave differently—or not exist at all. Suddenly, the exact same Bazel commit passes on Linux but fails on Windows. 

By writing a custom `py_binary` tool and using `ctx.actions.run`, you achieved two massive architectural wins:

### Win 1: True Portability
You are no longer relying on the host operating system's installed tools. You are providing the tool's source code (`my_compiler.py`) directly in the workspace. Because Bazel manages the Python runtime, this build will execute identically on Mac, Linux, and Windows.

### Win 2: The Implicit Build Graph
Take a look at your target in the `BUILD` file:
```python
hermetic_minify(
    name = "safe_minify",
    src = "data.json",
    ...
)
```
Notice that you *never explicitly mentioned* the Python tool in the `BUILD` file. 

Because you set `default = "//:json_compiler"` in the rule's `attrs`, Bazel automatically wired up the dependency graph. Before Bazel even attempts to run `safe_minify`, it analyzes the graph, sees the dependency, and **builds the Python tool first**. 

When it's time to execute `safe_minify`, `ctx.executable.compiler` contains the exact sandboxed path to the freshly compiled tool.

---

## 3. Foreshadowing: The `cfg = "exec"` Magic

You had to add `cfg = "exec"` to your compiler attribute. 

Without this, Bazel would assume the `py_binary` tool needs to be compiled for the exact same CPU architecture as the final JSON file. This sounds fine right now, but imagine if `hermetic_minify` was actually a rule compiling C++ for the microchip in a car's engine (ARM architecture). 

If you didn't have `cfg = "exec"`, Bazel would try to compile your Python tool to run *on the car's engine* instead of *on your laptop*. We will explore this mind-bending concept (Execution Transitions) deeply in Level 7.3!

---

## 4. Deep Dive: The `executable = True` Flag

When you declare an attribute to bring a custom tool into your rule, flagging it with `executable = True` is mandatory. Here is a brief summary of exactly what this flag does behind the scenes:

*   **Type Safety (The Guardrail):** It forces Bazel to verify that the target is actually a runnable binary (like `py_binary` or `cc_binary`). If a developer accidentally passes a `py_library` or a `.txt` file into the attribute, Bazel instantly stops the build with a type-checking error.
*   **Unlocks the Executable Object:** `ctx.actions.run` strictly requires a verified "executable file object." Setting this flag tells Bazel to populate the special `ctx.executable` struct (e.g., `ctx.executable.compiler`). You cannot use a standard `ctx.file` for a run action.
*   **The Invisible Toolchain (DevInfra Pattern):** By combining `executable = True`, `cfg = "exec"`, and a `default = "//..."` label, you create a foolproof system. You guarantee the tool is runnable, compiled for the host architecture, and completely hidden from the end-user, who just uses your rule without knowing the tool even exists.