# Bazel Fundamentals - Level 3: The Shell Transformer

## 1. The Core Concept: The Execution Sandbox

In Level 2, we registered a `FileWrite` action. Bazel handled the execution internally. 
In Level 3, we execute an arbitrary bash command using `ctx.actions.run_shell`. 

When Bazel reaches the **Execution Phase**, it does not run your bash command in your source directory. It creates a **Sandbox**—a temporary, highly restricted, empty directory tree. 

If you do not explicitly tell Bazel to mount an input file into that sandbox, the bash command will fail with a "File not found" error, even if the file exists in your source code repository.

---

## 2. New API Concepts

### Input Files vs. Output Strings
When a rule generates a file, the user passes a string (e.g., `out_name = "minified.json"`). 
When a rule takes an *existing* file as input, it must use a **Label**.

```python
# In the rule attributes:
"src": attr.label(allow_single_file = True)
```

A `Label` tells Bazel that this is a node in the dependency graph. During the Analysis phase, Bazel resolves that string (like `"data.json"`) into a `File` object, which you access via `ctx.file.src`.

### Dynamic Command Strings
Starlark does not support f-strings (`f"..."`). To inject file paths into your bash commands, use `.format()` or `%s` substitution.

You must use the `.path` property of the `File` object (e.g., `my_file.path`) to get the exact location Bazel mapped it to inside the sandbox.

---

## 3. The Implementation

### `my_rules.bzl`

```python
def _minify_json_impl(ctx):
    # 1. Get the input file object (Resolved from the Label attribute)
    in_file = ctx.file.src
    
    # 2. Declare the output file
    out_file = ctx.actions.declare_file(ctx.attr.out_name)
    
    # 3. Register the bash action
    ctx.actions.run_shell(
        # CRITICAL: If you omit `inputs`, Bazel will not mount the file 
        # into the execution sandbox, and the bash command will fail.
        inputs = [in_file],
        
        # Bazel needs to know what files to extract from the sandbox 
        # after the bash command finishes.
        outputs = [out_file],
        
        # Inject the sandbox-relative paths into the bash string
        command = "cat {src} | tr -d ' \n' > {dst}".format(
            src = in_file.path,
            dst = out_file.path,
        ),
    )
    
    # 4. Return the DefaultInfo provider so Bazel knows what this target yields
    return [DefaultInfo(files = depset([out_file]))]

minify_json = rule(
    implementation = _minify_json_impl,
    attrs = {
        "src": attr.label(allow_single_file = True, mandatory = True),
        "out_name": attr.string(mandatory = True),
    },
)
```

### `BUILD`

```python
load("//:my_rules.bzl", "minify_json")

# Note: data.json must exist in this directory!
minify_json(
    name = "my_minifier",
    src = "data.json", 
    out_name = "minified.json",
)
```

---

## 4. Debugging and Verification

### Inspecting the Action Graph
Before building, you can verify exactly what bash command Bazel plans to run:

```bash
bazel aquery //:my_minifier
```

**Key Takeaways from aquery:**
*   **`Mnemonic: Action`**: This indicates a generic shell action (unlike `FileWrite` from Level 2).
*   **`Inputs: [data.json]`**: Bazel explicitly tracks the input file. If `data.json` changes, Bazel knows it must re-run this action. If it hasn't changed, Bazel hits the cache and skips execution.
*   **`Command Line:`**: You will see the fully resolved `.format()` string with the exact sandbox paths.

### Building and Finding the Output

```bash
bazel build //:my_minifier
cat bazel-bin/minified.json
```
