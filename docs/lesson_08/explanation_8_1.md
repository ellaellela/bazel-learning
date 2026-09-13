# Bazel Fundamentals - Level 8.1: The CcInfo Provider (Explanations)

## 1. The Solution

Here is the completed implementation for `my_rules.bzl`:

```python
load("@rules_cc//cc:defs.bzl", "CcInfo")

def _hardware_version_header_impl(ctx):
    # 1. Generate the header file.
    out_file = ctx.actions.declare_file("hw_version.h")

    ctx.actions.write(
        output = out_file,
        content = "#define HW_VERSION " + str(ctx.attr.version) + "\n",
    )

    # 2. Create the C++ compilation context.
    # headers tells Bazel which header files are provided.
    # includes tells Bazel which directories should be added to the compiler's include search path.
    comp_ctx = cc_common.create_compilation_context(
        headers = depset([out_file]),
        includes = depset([out_file.dirname]),
    )

    # 3. Wrap the compilation context in CcInfo.
    # This allows native C++ rules such as cc_binary to consume this custom rule as a C++ dependency.
    my_cc_info = CcInfo(
        compilation_context = comp_ctx,
    )

    # 4. Return both the generated file and the C++ provider.
    return [
        DefaultInfo(
            files = depset([out_file]),
        ),
        my_cc_info,
    ]

hardware_version_header = rule(
    implementation = _hardware_version_header_impl,
    attrs = {
        "version": attr.int(mandatory = True),
    },
)
```

There are two important outputs from this rule serving different purposes: `DefaultInfo` (which provides `hw_version.h`) and `CcInfo` (which provides the `compilation_context` containing `headers` and `includes`).

---

## 2. Why `CcInfo` Is Necessary

The custom rule generates a real file: `hw_version.h`. However, simply generating the file does not automatically make it available to the C++ compiler.

For example, this alone is not enough:
```python
return [DefaultInfo(files = depset([out_file]))]
```
The file exists, but the C++ compilation machinery does not know that the file is part of the C++ dependency's compilation interface. A native C++ rule expects C++-specific information to be provided through `CcInfo`. 

Therefore the custom rule creates `CcInfo(compilation_context = comp_ctx)` and returns it. This gives the dependency a C++ compilation interface that a `cc_binary` can consume.

Conceptually: `custom rule` → `CcInfo` → `compilation_context (headers & includes)` → `cc_binary`.

---

## 3. `CcInfo` and the C++ Dependency Graph

Consider the BUILD file:

```python
hardware_version_header(
    name = "ecu_version",
    version = 42,
)

cc_binary(
    name = "firmware_app",
    srcs = ["main.cpp"],
    deps = [":ecu_version"],
)
```

When Bazel prepares the compilation of `main.cpp`, the C++ rules collect the compilation information exported by `ecu_version` (the header file and its containing directory). This is what allows `#include "hw_version.h"` to work. Without `CcInfo`, the custom rule would not be participating in the C++ compilation dependency graph in the required way.

---

## 4. Deep Dive: `compilation_context`

The C++ information provided by `CcInfo` is divided into different types of information. For this exercise, we only need the **compilation** side.

```python
comp_ctx = cc_common.create_compilation_context(
    headers = depset([out_file]),
    includes = depset([out_file.dirname]),
)
```

The compilation context describes information needed when compiling C++ source files. In this example, there are two important pieces: `headers` and `includes`.

---

## 5. `headers`: Which Header Is Provided?

The part `headers = depset([out_file])` tells Bazel: *This C++ dependency provides `hw_version.h` as a header.*

The `headers` collection contains actual `File` objects. This is different from telling the compiler where to search for the file. That is the job of `includes`.

---

## 6. `includes`: Where Should the Compiler Search?

The part `includes = depset([out_file.dirname])` tells Bazel which directories should be added to the C++ compiler's include search path.

Suppose the generated file ends up conceptually at `bazel-out/.../bin/hw_version.h`. Then `out_file.dirname` refers to `bazel-out/.../bin`. The C++ compiler can then effectively receive something equivalent to `-Ibazel-out/.../bin`.

When `main.cpp` contains `#include "hw_version.h"`, the compiler searches that directory and finds the generated header. Both pieces are useful: `headers` defines the C++ header, while `includes` tells the compiler where to search.

---

## 7. Why `headers` Alone Isn't Enough

It is tempting to think this should be sufficient: `comp_ctx = cc_common.create_compilation_context(headers = depset([out_file]))`. 

But the compiler still needs an appropriate include search path to resolve `#include "hw_version.h"`. The `headers` collection describes the header that belongs to the C++ compilation context. The `includes` collection describes directories that should be exposed as include search paths. For this exercise, we must provide both pieces of information.

---

## 8. Why We Use `out_file.dirname`

The header is generated dynamically via `out_file = ctx.actions.declare_file("hw_version.h")`.

We should not hard-code a path such as `bazel-out/...` because Bazel determines the actual output location. Instead, we ask the `File` object where it lives using `out_file.dirname`. This makes the rule independent of the exact output tree used by Bazel.

We use `headers = depset([out_file])` for the file itself, and `includes = depset([out_file.dirname])` for its containing directory.

---

## 9. Why `depset`?

The C++ APIs expect collections of files and paths that can participate efficiently in Bazel's dependency graph. 

For example, `depset([out_file])` creates a depset containing the generated header, and `depset([out_file.dirname])` creates a depset containing its include directory. Using depsets allows information to be merged efficiently as dependencies become larger. This becomes particularly important when a target has many transitive dependencies.

---

## 10. Why Return `DefaultInfo` as Well?

The rule returns two different interfaces:

*   **`DefaultInfo`**: Describes the rule's normal/default output files. In this case, `DefaultInfo(files = depset([out_file]))` says: *`hw_version.h` is an output of this target.* This is useful for ordinary Bazel file propagation and for tools that inspect a target's default outputs.
*   **`CcInfo`**: Says: *This target provides C++ compilation information.* That information includes the generated header and the include directory through the `compilation_context`.

Returning both is a good pattern for a rule that generates a file specifically intended for use by C++ code.

---

## 11. There Is No `linking_context` Here

Notice that the rule creates `CcInfo(compilation_context = comp_ctx)` but does not provide a `linking_context`. That's intentional.

This rule generates only a header (`hw_version.h`). It does not generate `.o`, `.a`, `.so`, or any other linkable C++ artifact. Therefore there is nothing for the linker to consume. The rule only needs to provide compilation information, unlike a C++ library rule which requires both.

---

## 12. The Action Factory: `ctx.actions`

The first part of the rule uses Bazel's action API:

*   **`declare_file()`**: `ctx.actions.declare_file("hw_version.h")` declares an output file that this rule will create. It does not immediately create the file on disk during the analysis phase.
*   **`write()`**: `ctx.actions.write(...)` registers an action that will write the specified content during the execution phase. For `version = 42`, the generated file contains `#define HW_VERSION 42`.

---

## 13. The Generated Header Is a Real Bazel Artifact

An important concept is that `hw_version.h` isn't just a string stored in Starlark. It is a real Bazel output.

Bazel tracks that the file is generated, which action generates it, which targets depend on it, and when it needs to be regenerated. This allows Bazel's dependency analysis and caching mechanisms to work correctly. If the version changes to `43`, Bazel can identify the affected actions.

---

## 14. Common Mistakes

*   **Mistake 1: Returning only `DefaultInfo`.** The header is an output, but the C++ dependency does not expose the necessary C++ compilation context. Return `CcInfo` as well.
*   **Mistake 2: Forgetting `includes`.** The compiler needs an include search directory. Use `includes = depset([out_file.dirname])`.
*   **Mistake 3: Putting the header itself in `includes`.** Don't do `includes = depset([out_file])`. `includes` represents **directories**, not header files.
*   **Mistake 4: Hard-coding the output directory.** Don't assume the generated header lives in a particular Bazel output directory. Use `out_file.dirname` so Bazel can determine the correct location.
*   **Mistake 5: Forgetting to load `CcInfo`.** Ensure you have `load("@rules_cc//cc:defs.bzl", "CcInfo")` at the top of your file.
*   **Mistake 6: Trying to provide a linking context.** This rule doesn't compile or link anything. There is no need to construct a `linking_context`.

---

## 15. The Complete Mental Model

The entire rule can be understood as five steps:
1.  Generate a file (`hw_version.h`).
2.  Describe it as a C++ header via `compilation_context` (using `headers` and `includes`).
3.  Wrap it in `CcInfo`.
4.  Return `CcInfo` to the dependency graph.
5.  Downstream `cc_binary` can now compile `#include "hw_version.h"`.

The key distinction is:
*   `DefaultInfo`: "Here are this target's output files."
*   `CcInfo`: "Here is how this target participates in the C++ compilation/linking graph."