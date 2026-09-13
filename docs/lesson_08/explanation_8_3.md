# Bazel Fundamentals - Level 8.3: The `cc_common` API (Explanations)

## 1. The Solution

Here is the completed implementation for `my_rules.bzl`:

```python
load("@bazel_tools//tools/cpp:toolchain_utils.bzl", "find_cpp_toolchain")
load("@rules_cc//cc:defs.bzl", "CcInfo")

def _certified_cc_library_impl(ctx):
    # 1. Grab the C++ toolchain and configure its features.
    cc_toolchain = find_cpp_toolchain(ctx)

    feature_config = cc_common.configure_features(
        ctx = ctx,
        cc_toolchain = cc_toolchain,
        requested_features = ctx.features,
        unsupported_features = ctx.disabled_features,
    )

    # 2. Compile the C++ sources.
    comp_context, comp_outputs = cc_common.compile(
        actions = ctx.actions,
        feature_configuration = feature_config,
        cc_toolchain = cc_toolchain,
        srcs = ctx.files.srcs,
        public_hdrs = ctx.files.hdrs,
        name = ctx.label.name,
    )

    # 3. Create the static library and linking context.
    link_context, linking_outputs = (
        cc_common.create_linking_context_from_compilation_outputs(
            actions = ctx.actions,
            feature_configuration = feature_config,
            cc_toolchain = cc_toolchain,
            compilation_outputs = comp_outputs,
            name = ctx.label.name,
        )
    )

    # 4. Expose the C++ information to downstream targets.
    return [
        CcInfo(
            compilation_context = comp_context,
            linking_context = link_context,
        ),
    ]


certified_cc_library = rule(
    implementation = _certified_cc_library_impl,
    attrs = {
        "srcs": attr.label_list(
            allow_files = [".cpp"],
        ),
        "hdrs": attr.label_list(
            allow_files = [".h"],
        ),
    },
    fragments = ["cpp"],
    toolchains = [
        "@bazel_tools//tools/cpp:toolchain_type",
    ],
)
```

The rule follows a simple pipeline:

```text
telemetry.cpp + telemetry.h
             │
             ▼
     cc_common.compile()
             │
             ├── compilation_context
             └── compilation_outputs
                     │
                     ▼
     create_linking_context_from_compilation_outputs()
                     │
                     ├── linking_context
                     └── linking_outputs
                             │
                             ▼
                           CcInfo
                             │
                             ▼
                         cc_binary
```

---

## 2. The Modern C++ Pipeline

The important idea is that the rule uses Bazel's C++ APIs directly rather than manually invoking a compiler with `ctx.actions.run()` or `ctx.actions.run_shell()`.

### `cc_common.compile()`

`cc_common.compile()` compiles the supplied sources using the selected C++ toolchain. It returns a `compilation_context`, containing information such as exported headers, and `compilation_outputs`, containing the generated object files.

In this example, `telemetry.cpp` is compiled into an object file while `telemetry.h` is exported through the compilation context.

```text
telemetry.cpp
      │
      ▼
cc_common.compile()
      │
      ├── compilation_context
      └── telemetry.o
```

### `create_linking_context_from_compilation_outputs()`

The object files then need to become a library that downstream C++ targets can consume. `cc_common.create_linking_context_from_compilation_outputs()` performs that library step and returns both the linking context and the resulting library artifacts.

The `link_context` is the important part for propagation: downstream C++ rules use it to understand what needs to be linked, rather than receiving raw `.o` files.

### What about `cc_common.link()`?

`cc_common.link()` is not interchangeable with `create_linking_context_from_compilation_outputs()`.

`cc_common.link()` returns a `CcLinkingOutputs` value, not a `CcLinkingContext`, so code such as this is incorrect:

```python
link_result = cc_common.link(...)

linking_context = link_result.linking_context
```

For a library rule such as this one, use:

```python
cc_common.create_linking_context_from_compilation_outputs(...)
```

This distinction is particularly important when reading older Bazel examples, because the C++ Starlark APIs have evolved over time.

---

## 3. `CcInfo`: Making the Rule Look Like a C++ Library

The final step is to combine the compilation and linking information into a `CcInfo` provider:

```python
CcInfo(
    compilation_context = comp_context,
    linking_context = link_context,
)
```

`CcInfo` is the standard interface through which C++ targets communicate compilation and linking information.

A downstream rule therefore doesn't need to know whether the information came from a native `cc_library` or our custom `certified_cc_library`; both expose the same `CcInfo` interface, which is why a normal `cc_binary` can consume the custom rule through `deps`.

That is what makes the custom rule behave like a normal C++ library.

---

## 4. `ctx.actions`: The Action Factory

Bazel separates the analysis phase, where Starlark describes the build, from the execution phase, where Bazel runs the registered actions. `ctx.actions` is the rule's action factory, providing APIs such as `declare_file()`, `write()`, and `run()` for registering that work.

Passing `actions = ctx.actions` to `cc_common.compile()` or `cc_common.create_linking_context_from_compilation_outputs()` gives the C++ API the action factory it needs to register the corresponding compilation or linking actions for your target.

---

## 5. `public_hdrs` vs. `private_hdrs`

C++ libraries need to distinguish between their public interface and implementation details.

`public_hdrs` are part of the target's public compilation interface. They are included in the compilation context and therefore become available to downstream C++ targets. In this exercise, declaring `telemetry.h` as a public header allows `main.cpp` to include it.

`private_hdrs` are intended for implementation details. They can be used when compiling the library itself but are not exported as part of its public compilation interface.

For example:

```text
telemetry_lib
   │
   ├── telemetry.h          ← public
   └── telemetry_internal.h ← private
```

The library can include both headers, while consumers should only depend on the public interface.

---

## 6. Why `CcInfo` Matters

Creating `telemetry.o` and `libtelemetry.a` is not enough for Bazel's C++ dependency graph. Bazel also needs metadata describing which headers consumers can use, which include paths they need, which libraries should be linked, and how those libraries participate in the dependency graph.

`CcInfo` carries this information.

The complete flow is therefore:

```text
source files
     │
     ▼
cc_common.compile()
     │
     ├── compilation_context
     └── compilation_outputs
                │
                ▼
create_linking_context_from_compilation_outputs()
                │
                └── linking_context
                         │
                         ▼
                       CcInfo
                         │
                         ▼
                     cc_binary
```

The key lesson is that a custom C++ rule should use Bazel's C++ APIs to create the normal C++ artifacts and then expose the resulting compilation and linking information through `CcInfo`.

---

## 7. Common Mistakes

### Mistake 1: Using `cc_common.link()` and expecting a linking context

This is incorrect:

```python
link_result = cc_common.link(...)

CcInfo(
    compilation_context = comp_context,
    linking_context = link_result.linking_context,
)
```

`cc_common.link()` returns `CcLinkingOutputs`, not a `CcLinkingContext`.

For this library rule, use:

```python
link_context, linking_outputs = (
    cc_common.create_linking_context_from_compilation_outputs(...)
)
```

### Mistake 2: Unpacking `cc_common.link()` into two values

This is also incorrect:

```python
link_context, link_outputs = cc_common.link(...)
```

The result of `cc_common.link()` is a single `CcLinkingOutputs` value.

The library API used in this exercise returns the two values:

```python
link_context, linking_outputs = (
    cc_common.create_linking_context_from_compilation_outputs(...)
)
```

### Mistake 3: Extra nesting around `ctx.files`

Don't write:

```python
srcs = (ctx.files.srcs,)
```

That creates a tuple containing the entire collection rather than a collection of individual `File` objects.

Use:

```python
srcs = ctx.files.srcs
```

If a particular API requires a tuple in the Bazel version you're using, use:

```python
srcs = tuple(ctx.files.srcs)
```

The important difference is that `ctx.files.srcs` contains individual files, while `(ctx.files.srcs,)` introduces an unnecessary level of nesting.

### Mistake 4: Forgetting to load `CcInfo`

If the rule uses `CcInfo(...)`, it must load the provider:

```python
load("@rules_cc//cc:defs.bzl", "CcInfo")
```

### Mistake 5: Returning only `DefaultInfo`

Returning only `DefaultInfo` does not make a target behave like a C++ library. A downstream `cc_binary` needs the C++ provider information:

```python
CcInfo(
    compilation_context = comp_context,
    linking_context = link_context,
)
```

---

## 8. The Mental Model

The entire rule can be reduced to four steps:

```text
1. Find the C++ toolchain
          │
          ▼
2. Compile source files
   cc_common.compile()
          │
          ▼
3. Create library + linking context
   create_linking_context_from_compilation_outputs()
          │
          ▼
4. Package C++ information
   CcInfo(...)
          │
          ▼
   downstream cc_binary
```

The custom rule isn't replacing Bazel's C++ machinery. It is using Bazel's C++ machinery directly and exposing the result through the standard `CcInfo` provider.

That is the core `cc_common` pattern to take away from this exercise.