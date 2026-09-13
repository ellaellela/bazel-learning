# Bazel Fundamentals - Level 8.5: Custom Toolchain Configuration (Explanations)

## 1. The Solution

Here is the completed implementation for `ecu_toolchain_config.bzl`:

```python
load("@rules_cc//cc/common:cc_common.bzl", "cc_common")
load("@rules_cc//cc:cc_toolchain_config_lib.bzl", "tool_path")

def _ecu_toolchain_config_impl(ctx):
    tool_paths = [
        tool_path(name = "gcc", path = "/usr/bin/clang++"),
        tool_path(name = "ld", path = "/usr/bin/ld"),
        tool_path(name = "ar", path = "/usr/bin/ar"),
        tool_path(name = "cpp", path = "/usr/bin/cpp"),
        tool_path(name = "gcov", path = "/usr/bin/gcov"),
        tool_path(name = "nm", path = "/usr/bin/nm"),
        tool_path(name = "objdump", path = "/usr/bin/objdump"),
        tool_path(name = "strip", path = "/usr/bin/strip"),
    ]

    return cc_common.create_cc_toolchain_config_info(
        ctx = ctx,
        features = [],
        action_configs = [],
        artifact_name_patterns = [],
        cxx_builtin_include_directories = [
            "/usr/local/include",
            "/usr/include",
            "/Library/Developer/CommandLineTools",
        ],
        toolchain_identifier = "ecu-custom-toolchain",
        host_system_name = "local",
        target_system_name = "local",
        target_cpu = "darwin",
        target_libc = "unknown",
        compiler = "clang",
        abi_version = "unknown",
        abi_libc_version = "unknown",
        tool_paths = tool_paths,
    )

ecu_toolchain_config = rule(
    implementation = _ecu_toolchain_config_impl,
)
```

---

## 2. DevInfra Theory: The Three Layers of a Toolchain

To completely override Bazel's C++ compilation behavior, you must configure three distinct layers. Missing any one of these will cause toolchain resolution to fall back to the default or fail entirely.

*   **Layer 1: `CcToolchainConfigInfo` (The Brains):** Defines paths to the physical binaries, compiler flags, and safe system include directories. It encapsulates *how* compilation happens.
*   **Layer 2: `cc_toolchain` (The Sandbox Files):** Pairs your configuration (Layer 1) with the physical files needed to run it in the Bazel sandbox.
*   **Layer 3: `toolchain` (The Registration):** Registers your `cc_toolchain` with Bazel's global platform ecosystem, defining *when* this toolchain should be selected based on target constraints.

---

## 3. DevInfra Deep Dive: The Sandbox and the Empty Filegroup

Bazel executes every compilation action inside a strict, isolated environment called the **sandbox**. By default, the sandbox is completely empty. If you do not explicitly tell Bazel to bring a file into the sandbox, the compiler cannot see it.

This is where Layer 2 and the `empty` filegroup come into play.

### The Role of `cc_toolchain` (Layer 2)
Your Starlark configuration (Layer 1) only contains *strings* (e.g., "the compiler is located at `/usr/bin/clang++`"). But strings cannot compile code. 
    
The native `cc_toolchain` rule bridges those strings to the physical sandbox. It has mandatory attributes like `compiler_files`, `linker_files`, and `all_files`. These attributes define the physical files Bazel must copy or symlink into the sandbox before running the compilation action.
*   **Layer 1 (The Config):** The recipe (paths and flags).
*   **Layer 2 (`cc_toolchain`):** The physical ingredients (the actual compiler binary, linker, and standard headers).

### Why `filegroup(name = "empty")`?
The `cc_toolchain` API strictly requires you to provide dependencies for `compiler_files` and `linker_files`. It will fail the analysis phase if you leave them blank.

However, in this exercise, we are doing a **non-hermetic** build. We configured the toolchain to use the host system's compiler (`/usr/bin/clang++`). Because `/usr/bin` is a protected host path, Bazel's local execution strategy automatically allows read access to it. We do not need Bazel to physically drag the `clang++` executable into the workspace sandbox.

Therefore, we pass an empty filegroup. It acts as a dummy target that satisfies the strict API requirements of `cc_toolchain` ("Yes, I gave you a target for your files!") without needlessly copying files.

### The Hermetic Alternative
If you were doing a true **hermetic** cross-compilation (e.g., compiling for the ECU using a specific compiler downloaded from the network), your `BUILD` file would look very different. You would pass the actual downloaded binaries so Bazel mounts them into the sandbox:

```python
# A filegroup containing the downloaded compiler binaries
filegroup(
    name = "downloaded_ecu_compiler_files",
    srcs = glob(["external/ecu_toolchain/bin/**"]),
)

cc_toolchain(
    name = "ecu_cc_toolchain",
    toolchain_config = ":ecu_config",
    # Bazel will now physically mount these binaries into the sandbox!
    compiler_files = ":downloaded_ecu_compiler_files", 
    linker_files = ":downloaded_ecu_compiler_files",
    # ...
)
```

---

## 4. Deep Dive: `tool_path` and Action Mapping

Bazel's internal action names for standard compilation tools are deeply rooted in legacy GCC naming conventions. Even if you are configuring an LLVM/Clang toolchain (or a proprietary embedded compiler), you must map your compiler's physical path to Bazel's expected logical names (`gcc` for the compiler, `ar` for the archiver, `ld` for the linker). 

---

## 5. Deep Dive: `cxx_builtin_include_directories`

By default, Bazel executes C++ compilation in a strict sandbox. If the compiler tries to `#include <iostream>`, Bazel will intercept the read request to the host system's `/usr/include` directory and block it. The `cxx_builtin_include_directories` list acts as a whitelist, telling Bazel's sandbox which absolute host paths are safe for the compiler to read from.

---

## 6. DevInfra Deep Dive: The `provides` Guardrail and Bazel 9

In older Bazel tutorials, you will often see custom rules explicitly declare what they output using the `provides` parameter:

```python
ecu_toolchain_config = rule(
    implementation = _ecu_toolchain_config_impl,
    provides = [CcToolchainConfigInfo], # Strict-typing guardrail
)
```

The `provides` list acts as a contract. It tells Bazel to throw an immediate error if the implementation function ever forgets to return that specific provider. 

However, in modern Bazel (specifically during the Bazel 9 Starlarkification migration), you should **omit the `provides` parameter** when writing C++ toolchain configs. Here is why:

### 1. The Private Provider Problem
While the factory function used to create the config (`cc_common.create_cc_toolchain_config_info`) is a fully supported, public API, the actual `CcToolchainConfigInfo` provider object itself was moved deep into the private, internal files of the `rules_cc` repository. 

Because it is not attached to the `cc_common` struct, attempting to explicitly declare it in the `provides` list would force you to load a private path (e.g., `load("@rules_cc//cc/private/...", "CcToolchainConfigInfo")`). Depending on private internal paths is a massive DevInfra anti-pattern, as they can break on any minor point-release.

### 2. Duck Typing to the Rescue
By omitting the `provides` parameter entirely, you instruct Bazel to use **duck typing** during the analysis phase.

When the native `cc_toolchain` target inspects your `ecu_config` target, it doesn't care about a strict upfront contract. It just looks at the output and asks, *"Does this target possess a `CcToolchainConfigInfo` provider?"* 

Because you used the public factory function:
```python
return cc_common.create_cc_toolchain_config_info(...)
```
The factory securely reaches into those private files, constructs the correct provider, and attaches it to your target. Bazel sees the provider, accepts the duck type, and the build succeeds. 

**The Takeaway:** When dealing with bleeding-edge Bazel migrations, trust the public factory functions and let duck typing handle the provider propagation.

---

## 7. Common API Mistakes to Avoid

Building a custom toolchain exposes you to Bazel's strictest validation checks. Here are the most common ways this configuration fails in production:

*   **Identifier Mismatches (The Silent Killer):** 
    The `toolchain_identifier` string defined in `create_cc_toolchain_config_info` **must exactly match** the `toolchain_identifier` attribute on the native `cc_toolchain` rule. If they differ by even a single character, Bazel will crash during the analysis phase with a cryptic resolution error.
*   **Incomplete `tool_paths` Definitions:** 
    You must provide a `tool_path` mapping for *every* standard action (`gcc`, `ld`, `ar`, `cpp`, `gcov`, `nm`, `objdump`, `strip`). If your custom embedded toolchain doesn't use one of these tools (like `gcov` for coverage), you cannot simply omit it. You must map it to `/bin/false` or `/usr/bin/false` so it intentionally fails only if Bazel actually attempts to invoke it.
*   **Missing Sandbox Files (`all_files`, `compiler_files`):** 
    In this exercise, we used local system binaries (`/usr/bin/...`), so we passed an `:empty` filegroup. However, in a true hermetic cross-compilation setup, you download the compiler via a workspace rule. If you forget to pass those downloaded filegroups into `compiler_files` and `linker_files`, Bazel will sandbox the action but forget to put the actual compiler inside the sandbox, resulting in a "command not found" execution error.
*   **Forgetting to Register the Toolchain:** 
    Simply defining a `toolchain` rule in a BUILD file doesn't make Bazel use it. Bazel will default to its auto-configured host toolchain. You must force Bazel to evaluate yours by passing `--extra_toolchains=//:ecu_toolchain` on the command line, or by permanently registering it in your `MODULE.bazel` / `WORKSPACE` file via `register_toolchains("//:all")`.
*   **Confusing Host vs. Target Constraints:** 
    When configuring `host_system_name` and `target_system_name`, remember cross-compilation. The *host* is the machine running Bazel (where the compiler executable lives). The *target* is the machine where the compiled binary will eventually run. Mixing these up will cause the platform resolution engine to ignore your toolchain entirely.

---

## 8. The Complete Mental Model

To master C++ in Bazel, you must understand how the toolchain system stacks together. The pipeline for completely owning the compilation process looks like this:

**1. The Starlark Configuration (`ecu_toolchain_config.bzl`)**
*   *Action:* Map logical names (`gcc`) to physical paths (`/usr/bin/clang++`).
*   *Action:* Whitelist system include directories.
*   *Output:* Returns `CcToolchainConfigInfo`.

**2. The Core Rule (`cc_toolchain`)**
*   *Action:* Binds the Starlark configuration to the physical sandbox files (the actual compiler binaries).
*   *Output:* A C++ toolchain target ready for execution.

**3. The Platform Registration (`toolchain`)**
*   *Action:* Wraps the core rule in a globally recognizable interface.
*   *Constraints:* Says, *"Use this toolchain ONLY IF the execution platform is Mac AND the target platform is an ARM ECU."*

**4. The Invocation**
*   *Action:* `bazel build //:ecu_app --extra_toolchains=//:ecu_toolchain`
*   *Result:* Bazel intercepts the default C++ compilation pipeline and redirects it through your custom rules.

### The Section 8 Capstone
You have now mapped the entire depth of Bazel's C++ infrastructure:
*   **Level 8.1:** How to inject headers into the dependency graph (`CcInfo` / `compilation_context`).
*   **Level 8.2:** How to dynamically query Bazel for the correct compiler path based on features.
*   **Level 8.3:** How to write custom rules that compile source code using `cc_common.compile`.
*   **Level 8.4:** How to inject proprietary, pre-compiled binaries into the graph (`LibraryToLink`).
*   **Level 8.5:** How to completely replace the underlying compiler Bazel uses to do all of the above.

You are no longer just writing Bazel targets; you are engineering the build system itself.