# Bazel Fundamentals - Level 7.3: The Execution Transition (Explanations)

## 1. The Solution

Here is the completed code for `my_rules.bzl`:

```python
def _firmware_builder_impl(ctx):
    out_file = ctx.actions.declare_file(ctx.label.name + ".bin")
    
    ctx.actions.run(
        executable = ctx.executable.compiler,
        outputs = [out_file],
        arguments = [out_file.path]
    )
    
    return [DefaultInfo(files = depset([out_file]))]

firmware_builder = rule(
    implementation = _firmware_builder_impl,
    attrs = {
        "compiler": attr.label(
            default = "//:codegen_tool",
            executable = True,
            cfg = "exec",  # The DevInfra Magic
        ),
    },
)
```

---

## 2. DevInfra Theory: The Parallel Universes

When you run `bazel build //:infotainment_firmware --cpu=arm`, Bazel creates a universal state for the build called the **Target Configuration**. It tells every rule in the graph: *"You are compiling for an ARM processor."*

If you just added the `compiler` attribute normally, Bazel would say: 
*"Okay, I need to compile `codegen_tool.py` into an executable so I can run it. The global state says `--cpu=arm`, so I will compile this Python tool using ARM libraries."*

But there is a massive problem: **Bazel is running on your laptop**, not on the car. When `ctx.actions.run` tries to spin up the sandbox and execute that ARM-compiled Python tool on your Intel/AMD/Apple chip, your operating system will instantly throw an "Exec format error" because the binary is fundamentally incompatible with your CPU.

### The `cfg = "exec"` Fork

By adding `cfg = "exec"`, you are creating a fork in the dependency graph. You are explicitly telling Bazel: 

*"Leave the Target Configuration. Transition this specific edge of the graph into the Execution Configuration."*

Bazel maintains two parallel universes simultaneously:
1. **The Target Universe:** Contains the final firmware, compiled for the car (`--cpu=arm`).
2. **The Exec Universe:** Contains the Python code generator, compiled for the machine executing the build (your laptop's CPU).

Because the tool is compiled in the Exec universe, it runs perfectly in the sandbox, generates the header files, and feeds them back into the Target universe so the firmware can finish building.

---

## 3. Demystifying the `cquery` Output

You ran this command to peek into Bazel's brain:
```bash
bazel cquery "deps(//:infotainment_firmware)" --cpu=arm --transitions=lite
```

If you look at the terminal output, you saw something like this:
```text
//:infotainment_firmware (9a7b5c...)
  -> //:codegen_tool (3f2e1d...)  [transition (target) -> (exec)]
```

Those alphanumeric strings (`9a7b5c...` and `3f2e1d...`) are **Configuration Hashes**. 
Bazel creates a unique SHA-256 hash representing the exact state of the universe (CPU, OS, compiler flags). 

Notice that `infotainment_firmware` and `codegen_tool` have *different hashes*. That `[transition (target) -> (exec)]` tag proves that Bazel actively changed the laws of physics for that one dependency, allowing a cross-platform automotive build to succeed flawlessly on a standard developer laptop.

---

## 3. Deep Dive: Other `cfg` Options (The Graph Benders)

While `"exec"` is the tool you use to branch out of the target configuration, Bazel provides other ways to manipulate the graph's environment.

### 1. `cfg = "target"` (The Default)
If you omit the `cfg` parameter entirely, Bazel implicitly sets it to `"target"`. 
* **What it does:** It passes the exact same configuration (CPU, OS, compiler flags) down to the dependency.
* **When to use it:** For standard libraries and data dependencies. If your firmware needs a math library, both the firmware and the math library must be compiled for the same ARM chip.

### 2. `cfg = "host"` (The Legacy Phantom)
In older codebases, you will frequently see `cfg = "host"`.
* **What it does:** Functionally, it does the exact same thing as `"exec"` (compiles the tool for the machine running the build).
* **The DevInfra History:** The Bazel team deprecated "host" because it assumed the machine coordinating the build (where Bazel runs) is the exact same machine executing the compiler. In massive enterprise setups, Bazel runs on a Mac laptop, but sends the heavy compiling to a Linux server farm (Remote Execution). `"exec"` correctly targets the execution environment, while `"host"` incorrectly targets the user's laptop.
* **The Verdict:** If you see `"host"`, treat it as `"exec"`. Always use `"exec"` in new code.

### 3. Custom Starlark Transitions (God Mode)
Instead of passing a pre-defined string, Bazel allows you to pass a **custom Python-like function** into `cfg`. This lets you intercept the graph and forcefully rewrite compiler flags midway through the build.

Imagine building an operating system where the main kernel is 64-bit, but the bootloader *must* be compiled in 32-bit. You can write a transition that forces the CPU flag to change just for that one dependency:

```python
# 1. Define the custom transition logic
def _bootloader_transition_impl(settings, attr):
    # Intercept the current CPU flag and force it to 32-bit (x86)
    return {"//command_line_option:cpu": "k8"}

bootloader_transition = transition(
    implementation = _bootloader_transition_impl,
    inputs = [],
    outputs = ["//command_line_option:cpu"],
)

# 2. Apply it to the attribute
my_rule = rule(
    implementation = _my_rule_impl,
    attrs = {
        "bootloader": attr.label(
            default = "//os:bootloader",
            cfg = bootloader_transition, # Passing the custom function!
        ),
    }
)
```