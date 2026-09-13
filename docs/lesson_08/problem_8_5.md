# Bazel Fundamentals - Level 8.5: Custom Toolchain Configuration (Problem Statement)

## The Scenario
Your team is preparing to port the Engine Telemetry system to a specialized Electronic Control Unit (ECU). The default Bazel C++ toolchain does not know how to compile code for this specific automotive hardware. 

To bridge this gap, you must define a custom C++ toolchain from scratch. You will write a configuration rule that maps Bazel's internal C++ actions to specific compiler paths on your machine, wrap it in a native `cc_toolchain`, and register it so Bazel can use it to build your application.

## DevInfra Concepts: The Toolchain Registration Pipeline

Defining a toolchain requires traversing three layers of abstraction:

1.  **The Config Rule (`cc_toolchain_config_info`)**
    You must write a custom Starlark rule that returns this specific provider. It tells Bazel exactly where the physical tools (compiler, linker, archiver) live on the host machine and which built-in directories are safe to include.
2.  **The Toolchain Core (`cc_toolchain`)**
    This is a native Bazel rule that takes your Starlark configuration and attaches it to empty `filegroup`s (which represent the sandbox files needed for compilation).
3.  **The Registration (`toolchain`)**
    This native Bazel rule tells the global platform system: *"I have a toolchain of type C++, and it should be used when building for this specific target architecture."*

---

## The Task

### 1. The Application Source
Create a simple application to verify the toolchain works.

**`main.cpp`**
```cpp
#include <iostream>

int main() {
    std::cout << "[ECU Toolchain] Telemetry application compiled successfully." << std::endl;
    return 0;
}
```

### 2. The Configuration Rule (`ecu_toolchain_config.bzl`)
Create a new file and complete the `TODO`.

```python
load("@rules_cc//cc/common:cc_common.bzl", "cc_common")
load("@rules_cc//cc:cc_toolchain_config_lib.bzl", "tool_path")

def _ecu_toolchain_config_impl(ctx):
    # Map Bazel's required tool names to the physical tools on your Mac.
    # (In a real cross-compilation scenario, these would point to your embedded toolchain).
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

    # TODO: Return cc_common.create_cc_toolchain_config_info()
    # Pass the following arguments:
    # ctx = ctx
    # features = []
    # action_configs = []
    # artifact_name_patterns = []
    # cxx_builtin_include_directories = [
    #     "/usr/local/include",
    #     "/usr/include",
    #     "/Library/Developer/CommandLineTools",
    # ]
    # toolchain_identifier = "ecu-custom-toolchain"
    # host_system_name = "local"
    # target_system_name = "local"
    # target_cpu = "darwin"
    # target_libc = "unknown"
    # compiler = "clang"
    # abi_version = "unknown"
    # abi_libc_version = "unknown"
    # tool_paths = tool_paths
    pass

ecu_toolchain_config = rule(
    implementation = _ecu_toolchain_config_impl,
    provides = [CcToolchainConfigInfo],
)
```

### 3. The Targets (`BUILD`)
Add the toolchain layers to your `BUILD` file.

```python
load("//:ecu_toolchain_config.bzl", "ecu_toolchain_config")

# 1. Instantiate your custom configuration
ecu_toolchain_config(
    name = "ecu_config",
)

# 2. Attach the configuration to a C++ toolchain core
cc_toolchain(
    name = "ecu_cc_toolchain",
    toolchain_identifier = "ecu-custom-toolchain",
    toolchain_config = ":ecu_config",
    all_files = ":empty",
    compiler_files = ":empty",
    dwp_files = ":empty",
    linker_files = ":empty",
    objcopy_files = ":empty",
    strip_files = ":empty",
    supports_param_files = 0,
)

# We use an empty filegroup because we are using local system binaries (/usr/bin/...) 
# rather than downloading a hermetic compiler into the Bazel sandbox.
filegroup(name = "empty")

# 3. Register the toolchain with Bazel's platform system
toolchain(
    name = "ecu_toolchain",
    exec_compatible_with = [],
    target_compatible_with = [],
    toolchain = ":ecu_cc_toolchain",
    toolchain_type = "@bazel_tools//tools/cpp:toolchain_type",
)

# 4. The application
cc_binary(
    name = "ecu_app",
    srcs = ["main.cpp"],
)
```

### 4. Verification
Run the application, but explicitly force Bazel to override its default C++ toolchain and use yours instead!

```bash
bazel run //:ecu_app --extra_toolchains=//:ecu_toolchain
```

**Success Criteria:** The build succeeds without missing headers or toolchain resolution errors, and the terminal prints `[ECU Toolchain] Telemetry application compiled successfully.`. You have completely taken over Bazel's C++ compilation engine.