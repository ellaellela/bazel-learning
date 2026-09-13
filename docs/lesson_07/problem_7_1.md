# Bazel Fundamentals - Level 7.1: The Custom Compiler (Problem Statement)

## The Scenario
Your build system relies on host bash utilities (`cat`, `tr`) to minify JSON files. The legal department just hired developers who use Windows, and your `run_shell` commands immediately failed because Windows doesn't have `tr`.

To fix this, you will write a custom cross-platform minifier tool in Python, build it as a `py_binary`, and then write a new Bazel rule that executes *that Python tool* instead of bash.

## DevInfra Concepts: `ctx.actions.run`

1. **The `executable` Attribute**
   To pass a tool into a rule, you define a label attribute and flag it as an executable. 
   *(Note: Bazel requires you to add `cfg = "exec"`. Treat this as magic boilerplate for now—we will fight the `cfg` Boss in Level 7.3).*
   ```python
   "compiler": attr.label(
       default = "//:my_python_tool", # Points to a py_binary target
       executable = True,
       cfg = "exec",
   )
   ```

2. **Accessing the Executable**
   Just like `ctx.file` gets a single file and `ctx.files` gets a list, you use `ctx.executable` to grab the compiled binary file of the tool:
   ```python
   tool_binary = ctx.executable.compiler
   ```

3. **`ctx.actions.run`**
   Unlike `run_shell`, `run` does not take a bash string. It takes an executable file and a list of string arguments. Bazel handles executing it securely.
   ```python
   ctx.actions.run(
       executable = tool_binary,
       inputs = [input_file],
       outputs = [output_file],
       arguments = ["--src", input_file.path, "--out", output_file.path]
   )
   ```

---

## The Task

### 1. The Tool (`my_compiler.py`)
Create a new file called `my_compiler.py` in your workspace. This is the script we will use as our "compiler".
```python
import sys
import json

def main():
    # Basic argument parsing: python my_compiler.py <src> <out>
    src_path = sys.argv[1]
    out_path = sys.argv[2]
    
    with open(src_path, "r") as f:
        data = json.load(f)
        
    with open(out_path, "w") as f:
        # Minify by removing whitespace
        json.dump(data, f, separators=(',', ':'))
        
if __name__ == "__main__":
    main()
```

### 2. The Rule (`my_rules.bzl`)
Add this to your `.bzl` file and complete the `TODO`s.

```python
def _hermetic_minify_impl(ctx):
    out_file = ctx.actions.declare_file(ctx.attr.out_name)
    
    # TODO: Call ctx.actions.run()
    # 1. Use ctx.executable.compiler as the 'executable'
    # 2. Put ctx.file.src in the 'inputs' array
    # 3. Put out_file in the 'outputs' array
    # 4. For 'arguments', pass a list with two strings: the path of the src file, and the path of the out_file
    
    return [DefaultInfo(files = depset([out_file]))]

hermetic_minify = rule(
    implementation = _hermetic_minify_impl,
    attrs = {
        "src": attr.label(allow_single_file = True, mandatory = True),
        "out_name": attr.string(mandatory = True),
        # Here is the boilerplate linking the rule to the tool!
        "compiler": attr.label(
            default = "//:json_compiler",
            executable = True,
            cfg = "exec",
        ),
    },
)
```

### 3. The Targets (`BUILD`)
Add the `py_binary` tool and your new custom rule target to your `BUILD` file. 
*(If you used the bash wrapper workaround in Level 5, you can use `py_binary` from `rules_python` here if you fixed it, or adapt it to `sh_binary`—but standard `py_binary` is preferred).*

```python
load("@rules_python//python:defs.bzl", "py_binary")
load("//:my_rules.bzl", "hermetic_minify")

# 1. The Tool Target
py_binary(
    name = "json_compiler",
    srcs = ["my_compiler.py"],
    main = "my_compiler.py",
)

# 2. The Consumer Target
hermetic_minify(
    name = "safe_minify",
    src = "data.json",
    out_name = "safe_minified.json",
)
```

### 4. Verification
Run the build and check the output!

```bash
bazel build //:safe_minify
cat bazel-bin/safe_minified.json
```

**Success Criteria:** Bazel should first compile the Python tool, then spin up a sandbox, inject the tool into it, and successfully execute it to produce the minified JSON file.