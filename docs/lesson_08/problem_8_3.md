# Bazel Fundamentals - Level 8.3: The cc_common API (Problem Statement)

## The Scenario
Your team is building an Engine Telemetry system. For regulatory reasons, the core telemetry filter must be compiled using a highly restrictive, custom Bazel rule (`certified_cc_library`) rather than the standard `cc_library`. 

You need to write a custom rule that accepts a `.cpp` and `.h` file, invokes Bazel's native C++ compiler, links it into a static library, and passes it to a standard `cc_binary` seamlessly.

## DevInfra Concepts: The `cc_common` Pipeline

To replicate a native C++ build, your custom rule must perform a two-step pipeline using the `cc_common` API:

1. **`cc_common.compile`**
   This function takes your source files, headers, and the feature configuration. It automatically handles all the `-I` flags, `#include` tracking, and compiler invocations. 
   It returns two things: a `compilation_context` (for downstream targets that need your headers) and `compilation_outputs` (the compiled `.o` object files).

2. **`cc_common.create_linking_context_from_compilation_outputs`**
   C++ targets don't just pass raw `.o` files to each other; they archive them into static libraries (`.a`). 
   This function takes the `compilation_outputs` from the previous step and invokes the archiver. It returns a `linking_context` and `linking_outputs`, perfectly bundled for propagation. *(Note: The older `cc_common.link` API is now strictly reserved for terminal actions like executables, not intermediate static libraries).*

3. **The `CcInfo` Assembly**
   Finally, you combine the `compilation_context` and `linking_context` into a single `CcInfo` provider, completely disguising your custom rule as a standard C++ library!

---

## The Task

### 1. The C++ Source Files
Create a header and source file for our telemetry filter.

**`telemetry.h`**
```cpp
#pragma once
int calculate_telemetry(int input_signal);
```

**`telemetry.cpp`**
```cpp
#include "telemetry.h"
int calculate_telemetry(int input_signal) { 
    return input_signal * 42; 
}
```

**`main.cpp`**
```cpp
#include <iostream>
#include "telemetry.h"

int main() {
    std::cout << "Raw Signal: 10" << std::endl;
    std::cout << "Filtered Telemetry: " << calculate_telemetry(10) << std::endl;
    return 0;
}
```

### 2. The Rule (`my_rules.bzl`)
Add this rule and complete the `TODO`s. Notice how we reuse the toolchain logic from Level 8.2!

```python
load("@bazel_tools//tools/cpp:toolchain_utils.bzl", "find_cpp_toolchain")

def _certified_cc_library_impl(ctx):
    # 1. Boilerplate: Grab the toolchain and configure features
    cc_toolchain = find_cpp_toolchain(ctx)
    feature_config = cc_common.configure_features(
        ctx = ctx,
        cc_toolchain = cc_toolchain,
        requested_features = ctx.features,
        unsupported_features = ctx.disabled_features,
    )

    # 2. Compile the C++ code
    # TODO: Call cc_common.compile()
    # Pass: actions = ctx.actions, feature_configuration = feature_config, cc_toolchain = cc_toolchain
    # Pass: srcs = ctx.files.srcs, public_hdrs = ctx.files.hdrs
    # Pass: name = ctx.label.name
    # Assign the result to: (comp_context, comp_outputs)
    
    # 3. Archive the object files into a static library and create linking context
    # TODO: Call cc_common.create_linking_context_from_compilation_outputs()
    # Pass: actions = ctx.actions, feature_configuration = feature_config, cc_toolchain = cc_toolchain
    # Pass: compilation_outputs = comp_outputs
    # Pass: name = ctx.label.name
    # Assign the result to: (link_context, linking_outputs)

    # 4. Return the assembled CcInfo
    # TODO: Return a list containing a CcInfo provider.
    # Pass: compilation_context = comp_context, linking_context = link_context
    pass

certified_cc_library = rule(
    implementation = _certified_cc_library_impl,
    attrs = {
        "srcs": attr.label_list(allow_files = [".cpp"]),
        "hdrs": attr.label_list(allow_files = [".h"]),
    },
    fragments = ["cpp"],
    toolchains = ["@bazel_tools//tools/cpp:toolchain_type"],
)
```

### 3. The Targets (`BUILD`)
Add the targets to your `BUILD` file. Watch how the standard `cc_binary` consumes your custom rule perfectly.

```python
load("//:my_rules.bzl", "certified_cc_library")

# Your custom rule compiling the library
certified_cc_library(
    name = "telemetry_lib",
    srcs = ["telemetry.cpp"],
    hdrs = ["telemetry.h"],
)

# Standard native C++ binary
cc_binary(
    name = "telemetry_app",
    srcs = ["main.cpp"],
    deps = [":telemetry_lib"],
)
```

### 4. Verification
Run the C++ binary!

```bash
bazel run //:telemetry_app
```

**Success Criteria:** The build should compile without any "missing header" or "undefined reference" errors, and the terminal should print `Filtered Telemetry: 420`. You have successfully built a native compiler rule from scratch!