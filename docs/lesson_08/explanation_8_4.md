# Bazel Fundamentals - Level 8.4: Wrapping Precompiled Libraries (Explanations)

## 1. The Solution

Here is the completed implementation for `my_rules.bzl`:

```python
load("@bazel_tools//tools/cpp:toolchain_utils.bzl", "find_cpp_toolchain")
load("@rules_cc//cc/common:cc_common.bzl", "cc_common")
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
    comp_ctx = cc_common.create_compilation_context(
        headers = depset([ctx.file.hdr]),
        includes = depset([ctx.file.hdr.dirname]),
    )
    
    # 3. Create the LibraryToLink (wrap the .a file)
    lib_to_link = cc_common.create_library_to_link(
        actions = ctx.actions,
        feature_configuration = feature_config,
        cc_toolchain = cc_toolchain,
        static_library = ctx.file.static_lib,
    )
    
    # 4. Create Linker Input & Context
    linker_input = cc_common.create_linker_input(
        owner = ctx.label,
        libraries = depset([lib_to_link]),
    )
    
    link_ctx = cc_common.create_linking_context(
        linker_inputs = depset([linker_input])
    )

    # 5. Return the assembled CcInfo
    return [
        DefaultInfo(files = depset([ctx.file.static_lib, ctx.file.hdr])),
        CcInfo(
            compilation_context = comp_ctx,
            linking_context = link_ctx,
        )
    ]

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

---

## 2. DevInfra Theory: The Linking Context Pipeline

When you compile C++ from source (Level 8.3), Bazel automatically tracks the object files and constructs the linking artifacts. However, when a vendor hands you a precompiled binary blob (`.a` or `.so`), Bazel's C++ machinery has no idea what it is or how to link it. You must manually construct the chain of metadata that the linker expects.

This requires a strictly typed three-step pipeline:

*   **Step 1: `LibraryToLink`** 
    Raw files mean nothing to the linker. `cc_common.create_library_to_link` inspects the `.a` file against the current `cc_toolchain` and `feature_configuration`. It generates an internal `LibraryToLink` object that confirms this binary is compatible with the current build environment (e.g., verifying you aren't trying to link a Windows `.lib` using a Linux toolchain).
*   **Step 2: `LinkerInput`** 
    Linkers process a sequence of inputs. `cc_common.create_linker_input` wraps your library (or a depset of multiple libraries) and associates them with an `owner` (`ctx.label`). This ownership ensures Bazel can generate accurate linker command lines, apply correct rpaths, and trace errors back to the specific `prebuilt_cc_library` target.
*   **Step 3: `CcLinkingContext`**
    Finally, the `LinkerInput` is bundled into the `CcLinkingContext`. This is the exact provider structure that `CcInfo` requires to successfully propagate linking flags up the dependency graph to the final `cc_binary`.

---

## 3. Comparison with Previous Levels

*   **Level 8.1 (Headers Only):** We created a `compilation_context` but left out the `linking_context` entirely because header files do not require a linker step.
*   **Level 8.3 (Compile from Source):** We used `cc_common.create_linking_context_from_compilation_outputs()`, which automatically ran the archiver and generated the linking context for us.
*   **Level 8.4 (Precompiled):** Because we bypassed compilation entirely, we must manually build the linking context from the raw `static_library` file using the three-step wrapper API.

---

## 4. Common API Mistakes to Avoid

*   **Passing the file directly:** You cannot pass a raw `.a` file (or `ctx.file.static_lib`) directly into `CcLinkingContext` or `CcInfo`. It must be promoted through `LibraryToLink` and `LinkerInput` first.
*   **Forgetting `owner` in `LinkerInput`:** Failing to pass `owner = ctx.label` will crash the analysis phase. Bazel strictly requires an owner for tracking cross-references during complex linker invocations.
*   **Assuming Dynamic Linking works identically:** This specific code hardcodes the wrapper for a `static_library`. If the vendor provided a shared object (`.so` or `.dylib`), you must pass it to `dynamic_library` in `create_library_to_link` instead, which triggers entirely different `rpath` and dynamic linker logic.
*   **Omitting DefaultInfo:** Even though `CcInfo` handles the C++ graph, returning `DefaultInfo` ensures the physical files propagate natively through Bazel's execution sandbox. Without it, remote execution environments might drop the `.a` file because it isn't formally registered as an output.

---

## 5. The Complete Mental Model

Wrapping a precompiled library combines everything learned in Section 8 into a single, comprehensive structure:

1.  **Define the Interface (Headers):** `create_compilation_context()` handles the `.h` file so downstream targets can `#include` it.
2.  **Define the Implementation (Binary):** `create_library_to_link()` -> `create_linker_input()` -> `create_linking_context()` handles the `.a` file so downstream targets can resolve the symbols.
3.  **Disguise the Package:** Wrap both contexts in `CcInfo` so a native `cc_binary` cannot tell the difference between your proprietary vendor blob and a standard, open-source Bazel `cc_library`.