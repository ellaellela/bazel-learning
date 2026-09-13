# Bazel Fundamentals - Level 8.4: Wrapping Precompiled Libraries (Problem Statement)

## The Scenario
Your team has purchased a proprietary, closed-source optimization engine from a third-party vendor. They did not provide the C++ source code. Instead, they gave you two files: a single public header (`vendor_math.h`) and a precompiled static library (`libvendor_math.a`).

You need to write a custom Bazel rule (`prebuilt_cc_library`) that ingests these precompiled artifacts and exposes them as a standard `CcInfo` provider, allowing your internal `cc_binary` to link against the vendor's code effortlessly.

## DevInfra Concepts: The Linking Context Pipeline

In Level 8.1, you learned how to create a `compilation_context` for headers. In Level 8.3, you learned how to compile code from scratch. Now, you must bridge the gap by manually constructing the linking pipeline for an artifact you didn't compile.

To wrap a precompiled library, you use three chained APIs:

1.  **`cc_common.create_library_to_link`**: 
    This API takes a raw precompiled file (like `.a` or `.so`) and wraps it in Bazel's internal `LibraryToLink` object. It requires the C++ toolchain and feature configuration to ensure the library is compatible with the current build environment.
2.  **`cc_common.create_linker_input`**: 
    Linkers need more than just the library file; they need to know *who* owns it and what flags it requires. This API wraps a `depset` of `LibraryToLink` objects into a `LinkerInput` object.
3.  **`cc_common.create_linking_context`**: 
    Finally, you wrap a `depset` of `LinkerInput` objects into the `CcLinkingContext`, which is exactly what `CcInfo` requires!

---

## The Task

### 1. Simulate the Vendor Artifacts
First, let's create the dummy "vendor" files. Open your terminal in your workspace and run these commands to compile a static library manually (simulating the vendor's build process):

```bash
# 1. Create the vendor header
cat << 'EOF' > vendor_math.h
#pragma once
int proprietary_multiply(int a, int b);
EOF

# 2. Create the vendor source (which you pretend you don't have)
cat << 'EOF' > vendor_math.cpp
#include "vendor_math.h"
int proprietary_multiply(int a, int b) { return a * b * 100; }
EOF

# 3. Compile it into a static library manually
g++ -c vendor_math.cpp -o vendor_math.o
ar rcs libvendor_math.a vendor_math.o

# 4. Clean up the source and object file to simulate a closed-source delivery
rm vendor_math.cpp vendor_math.o
```

### 2. Create the App Source
Now create your team's application that will consume this library:

**`main.cpp`**
```cpp
#include <iostream>
#include "vendor_math.h"

int main() {
    std::cout << "Vendor Output (3 x 4): " << proprietary_multiply(3, 4) << std::endl;
    return 0;
}
```

### 3. The Rule (`my_rules.bzl`)
Add this rule and complete the `TODO`s to manually construct the `CcInfo`.

```python
load("@bazel_tools//tools/cpp:toolchain_utils.bzl", "find_cpp_toolchain")
load("@rules_cc//cc:defs.bzl", "CcInfo")

def _prebuilt_cc_library_impl(ctx):
    # 1. Boilerplate: Toolchain & Features
    cc_toolchain = find_cpp_toolchain(ctx)
    feature_config = cc_common.configure_features(
        ctx = ctx,
        cc_toolchain = cc_toolchain,
        requested_features = ctx.features,
        unsupported_features = ctx.disabled_features,
    )

    # 2. Create the Compilation Context (for the header)
    # TODO: Use cc_common.create_compilation_context
    # Pass: headers = depset([ctx.file.hdr]), includes = depset([ctx.file.hdr.dirname])
    
    # 3. Create the LibraryToLink (wrap the .a file)
    # TODO: Use cc_common.create_library_to_link
    # Pass: actions = ctx.actions, feature_configuration = feature_config, cc_toolchain = cc_toolchain
    # Pass: static_library = ctx.file.static_lib
    
    # 4. Create Linker Input & Context
    # TODO: Use cc_common.create_linker_input (owner = ctx.label, libraries = depset([library_to_link_from_step_3]))
    # TODO: Use cc_common.create_linking_context (linker_inputs = depset([linker_input_from_above]))

    # 5. Return the assembled CcInfo
    # TODO: Return [CcInfo(compilation_context=..., linking_context=...)]
    pass

prebuilt_cc_library = rule(
    implementation = _prebuilt_cc_library_impl,
    attrs = {
        "hdr": attr.label(allow_single_file = [".h"]),
        "static_lib": attr.label(allow_single_file = [".a"]),
    },
    fragments = ["cpp"],
    toolchains = ["@bazel_tools//tools/cpp:toolchain_type"],
)
```

### 4. The Targets (`BUILD`)
Add the targets to your `BUILD` file:

```python
load("//:my_rules.bzl", "prebuilt_cc_library")

# Wrap the vendor's precompiled library
prebuilt_cc_library(
    name = "vendor_math",
    hdr = "vendor_math.h",
    static_lib = "libvendor_math.a",
)

# Standard native C++ binary
cc_binary(
    name = "app",
    srcs = ["main.cpp"],
    deps = [":vendor_math"],
)
```

### 5. Verification
Run the C++ binary!

```bash
bazel run //:app
```

**Success Criteria:** The build should succeed, link the external static library automatically, and print `Vendor Output (3 x 4): 1200`.