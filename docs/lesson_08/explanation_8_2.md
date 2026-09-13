# Bazel Fundamentals - Level 8.2: Toolchain Resolution (Explanations)

## 1. The Solution

Here is the completed implementation for `my_rules.bzl`:

```python
load("@bazel_tools//tools/cpp:toolchain_utils.bzl", "find_cpp_toolchain")
load("@rules_cc//cc:action_names.bzl", "ACTION_NAMES")

def _toolchain_inspector_impl(ctx):
    # 1. Find the C++ toolchain selected for this configuration.
    cc_toolchain = find_cpp_toolchain(ctx)

    # 2. Configure the C++ features for this target.
    feature_config = cc_common.configure_features(
        ctx = ctx,
        cc_toolchain = cc_toolchain,
        requested_features = ctx.features,
        unsupported_features = ctx.disabled_features,
    )

    # 3. Ask the configured toolchain which tool performs the C++ compile action.
    compiler_path = cc_common.get_tool_for_action(
        feature_configuration = feature_config,
        action_name = ACTION_NAMES.cpp_compile,
    )

    # 4. Write the result to a report file.
    out_file = ctx.actions.declare_file(ctx.label.name + "_report.txt")
    ctx.actions.write(
        output = out_file,
        content = "The C++ compiler for this configuration is: " + compiler_path + "\n",
    )

    return [DefaultInfo(files = depset([out_file]))]

toolchain_inspector = rule(
    implementation = _toolchain_inspector_impl,
    fragments = ["cpp"],
    toolchains = ["@bazel_tools//tools/cpp:toolchain_type"],
)
```

The fundamental principle here is that the rule **never** defines a hardcoded path like `compiler = "/usr/bin/g++"`. Instead, it asks Bazel: *"For the C++ toolchain and feature configuration selected for this build, what tool performs the C++ compile action?"*

---

## 2. DevInfra Theory: Why Hard-Coding the Compiler Is Wrong

A tempting implementation would be using `ctx.actions.run_shell(command = "/usr/bin/g++ my_file.cpp")`. This creates a major portability problem because it assumes the machine has `g++` installed, it's located exactly at `/usr/bin/g++`, and that this specific compiler is appropriate for the target hardware.

Modern Bazel uses **platforms, constraints, and toolchain resolution** to select compatible toolchains. It distinguishes between:
*   **Execution Platform:** Where the build tools (the compiler) execute.
*   **Target Platform:** Where the resulting compiled software runs.

By participating in Bazel's toolchain model, your rule seamlessly handles cross-compilation without changing a single line of Starlark code.

---

## 3. The `cc_common` API Step-by-Step

### Step 1: Declaring the Toolchain
By adding `toolchains = ["@bazel_tools//tools/cpp:toolchain_type"]` to the rule definition, you declare that the rule requires a C++ toolchain. Bazel resolves an appropriate toolchain before the execution phase begins.

### Step 2: `find_cpp_toolchain(ctx)`
This helper retrieves the `CcToolchainInfo` object representing the selected toolchain. It contains metadata about the compiler, linker, supported features, and action configurations. The rule does not discover the compiler itself; it asks Bazel for the *already-selected* toolchain.

### Step 3: `cc_common.configure_features`
A toolchain is more than just a compiler binary. It defines features that affect optimization, debugging, PIC, and LTO. 
By passing `requested_features = ctx.features` and `unsupported_features = ctx.disabled_features`, your custom rule respects the feature configuration provided by the user (e.g., passing `--features=asan` via the command line) rather than silently ignoring it. 

### Step 4: `ACTION_NAMES` and `get_tool_for_action`
`ACTION_NAMES.cpp_compile` provides the well-known name for the C++ compilation action. We pass this into `cc_common.get_tool_for_action()`. 
**Why not use `cc_toolchain.compiler_path`?** Because toolchains are *action-oriented*. The question isn't *"What compiler does this toolchain have?"* but rather *"What tool is configured for this specific action?"*

---

## 4. Execution vs. Analysis: Generating the Report

The compiler path is obtained during the **Analysis Phase**. However, the Starlark implementation does not directly write files to disk. Instead, we register an action for the **Execution Phase**.

Because we only need to write a known text string to a file, we use `ctx.actions.write()`. A useful rule of thumb:
*   Need to create a file containing known text? Use `ctx.actions.write()`.
*   Need to execute an external program to generate a file? Use `ctx.actions.run()`.

Avoid introducing a shell command when no shell is required.

---

## 5. Sandboxing and Compiler Wrappers

When you inspect the output report, the path returned might look like `external/.../cc_wrapper.sh`.

The path returned by the API is a **toolchain path**, not necessarily a physical binary on your host machine. Bazel C++ toolchains often use a wrapper script to sanitize the environment (stripping out toxic system variables) before forwarding the command to the actual compiler (`clang++` or `gcc`). 

If you see a wrapper script, it means the toolchain intentionally uses that wrapper as the configured tool for the action to preserve hermeticity.

---

## 6. Common Mistakes

1.  **Hard-coding paths:** Bypassing `get_tool_for_action()` and hard-coding `/usr/bin/g++` destroys portability.
2.  **Forgetting to declare the toolchain:** If you forget `toolchains = [...]` in the rule definition, you cannot reliably retrieve the C++ toolchain from the context.
3.  **Forgetting `configure_features`:** Do not jump directly from `find_cpp_toolchain` to assuming the tool path. Features alter flags and action configurations.
4.  **Hard-coding action names:** Using string literals like `"c++-compile"` instead of `ACTION_NAMES.cpp_compile` is prone to typos and breaks easily.
5.  **Assuming `--cpu` alone defines the toolchain:** Treat `--cpu` as just one configuration input. Actual toolchain selection involves target/execution platforms and registered constraints.
6.  **Assuming `get_tool_for_action()` returns the physical compiler:** It returns the tool *configured for the action*, which is often a sandbox-aware wrapper script.
7.  **Assuming features always change the executable:** A feature like `--features=asan` might just inject new instrumentation flags into the action configuration without changing the underlying compiler binary at all.

---

## 7. The Complete Mental Model

The entire progression from simple to toolchain-aware rules looks like this:

`Rule Requirements` → `find_cpp_toolchain()` → `configure_features()` → `get_tool_for_action()` → `Action Execution`

The central DevInfra principle to take away from this exercise is: **Don't make custom build rules guess which compiler to use. Ask Bazel's toolchain system.** 

Once a custom rule participates in Bazel's toolchain model, the exact same Starlark implementation will work flawlessly across local laptops, CI servers, cross-compilation environments, and remote execution clusters.