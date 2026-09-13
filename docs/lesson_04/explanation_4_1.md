# Bazel Fundamentals - Level 4: The Graph Builder

## 1. The Core Concept: Target Chaining

In Bazel, rules do not care *where* an input file came from. 
A file could be a static source file committed to git (`data.json`), or it could be a dynamically generated output from another rule (`//:my_minifier`). 

Because all rules communicate by returning **Providers** (like `DefaultInfo`), you can seamlessly chain rules together. Bazel automatically builds the Directed Acyclic Graph (DAG) and ensures actions execute in the correct order.

---

## 2. New API Concepts

### Accepting Multiple Files (`label_list`)
To allow a rule to take an array of dependencies (both raw files and other targets), you use `label_list`:

```python
"srcs": attr.label_list(allow_files = True, mandatory = True)
```

### Accessing the Files (`ctx.files`)
When you use a single `label`, you access the `File` object via `ctx.file`. 
When you use a `label_list`, Bazel aggregates all the files from all the provided targets. You access them as a Starlark list via `ctx.files`.

```python
# Returns a list of Bazel File objects
all_my_inputs = ctx.files.srcs 
```

### Starlark List Comprehensions
To pass a list of files into a bash string, you must extract their sandbox paths. Starlark supports Python-style list comprehensions and string joins:

```python
path_string = " ".join([f.path for f in ctx.files.srcs])
```

---

## 3. The Implementation

### `my_rules.bzl`

```python
def _minify_many_impl(ctx):
    # 1. Declare the output file
    out_file = ctx.actions.declare_file(ctx.attr.out_name)
    
    # 2. Extract paths and glue them together with spaces
    in_paths = " ".join([f.path for f in ctx.files.srcs])
    
    # 3. Register the shell action
    ctx.actions.run_shell(
        # CRITICAL: Pass the raw list of File objects so Bazel mounts them
        inputs = ctx.files.srcs,
        outputs = [out_file],
        command = "cat {inputs} | tr -d ' \n' > {out}".format(
            inputs = in_paths,
            out = out_file.path,
        ),
    )
    
    # 4. Return the DefaultInfo provider
    return [DefaultInfo(files = depset([out_file]))]

minify_many = rule(
    implementation = _minify_many_impl,
    attrs = {
        "srcs": attr.label_list(allow_files = True, mandatory = True),
        "out_name": attr.string(mandatory = True),
    },
)
```

### `BUILD`

```python
load("//:my_rules.bzl", "minify_many")

minify_many(
    name = "combo_minifier",
    srcs = [
        "data.json",       # A static source file
        "//:my_minifier",  # A dynamic output from another rule!
    ],
    out_name = "combo.json",
)
```

---

## 4. Debugging the Chain

You can prove Bazel understands this dependency chain using `aquery`. 

```bash
bazel aquery //:combo_minifier
```

If you look closely at the output, you will see Bazel lists the *output* of `//:my_minifier` in the `Inputs` section of `//:combo_minifier`. Bazel will completely orchestrate the sandboxing and execution order based purely on this graph.