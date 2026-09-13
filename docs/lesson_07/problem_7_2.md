# Bazel Fundamentals - Level 7.2: The Args Object (Problem Statement)

## The Scenario
Your company has a C++ codebase where a single library might include 5,000 header files. You wrote a custom Python tool that needs to process all 5,000 of those file paths at once.

**The Trap:**
If you pass 5,000 paths directly into the `arguments = [...]` list of `ctx.actions.run`, you will hit two massive walls:
1. **The Memory Wall:** If you call `my_depset.to_list()` during the Analysis Phase to build that array of strings, you undo all the memory optimizations we learned in Level 6.3. The JVM will crash.
2. **The OS Wall:** Operating systems have a maximum character limit for terminal commands (e.g., `ARG_MAX`). Passing 5,000 paths will instantly exceed this limit, causing a "Command line too long" fatal error.

## DevInfra Concepts: The `Args` Object

Bazel solves both problems with a special object created by `ctx.actions.args()`. 

1. **Deferred Evaluation (The Memory Fix)**
   When you pass a `depset` into an `Args` object, Bazel *does not flatten it* during the Analysis Phase. It waits until the Execution Phase (when memory is no longer a bottleneck) to call `.to_list()` behind the scenes.
   ```python
   args = ctx.actions.args()
   args.add("--output", out_file)
   args.add_all(my_depset) # Pass the depset directly! No .to_list() needed!
   ```

2. **Parameter Files (The OS Fix)**
   If you tell the `Args` object that the command line might get too long, Bazel will magically write all 5,000 arguments into a temporary text file (a "params file"). It will then execute your tool like this: `python my_tool.py @params.txt`. Your Python script's `argparse` module will automatically read the text file as if the arguments were typed in the terminal!
   ```python
   # Forces Bazel to write arguments to a file instead of the terminal
   args.use_param_file("@%s", use_always = True) 
   ```

3. **Passing Args to Run**
   You just pass the `args` object in a list to the `arguments` parameter of `run`.
   ```python
   ctx.actions.run(
       ...
       arguments = [args]
   )
   ```

---

## The Task

You are going to write a `bulk_archiver` rule. It takes a massive list of files and passes them to a Python tool using the `Args` object and a params file.

### 1. The Tool (`archiver_tool.py`)
Create this Python script. Notice how standard Python `argparse` automatically supports reading from `@params.txt` files if we use the `fromfile_prefix_chars` parameter!
```python
import argparse
import sys

def main():
    parser = argparse.ArgumentParser(fromfile_prefix_chars='@')
    parser.add_argument("--out", required=True)
    parser.add_argument("--srcs", nargs='+', default=[])
    
    args = parser.parse_args()
    
    with open(args.out, "w") as f:
        f.write("Archived Files:\n")
        for src in args.srcs:
            f.write(f"- {src}\n")

if __name__ == "__main__":
    main()
```

### 2. The Rule (`my_rules.bzl`)
Add this to your `.bzl` file and complete the `TODO`s.

```python
def _bulk_archiver_impl(ctx):
    out_file = ctx.actions.declare_file(ctx.attr.out_name)
    
    # 1. Create the Args object
    args = ctx.actions.args()
    
    # 2. Add the output flag
    args.add("--out", out_file)
    
    # 3. Add the massive list of source files
    # TODO: Use args.add_all() to add the "--srcs" flag, followed by ctx.files.srcs
    
    # 4. Enable the Parameter File escape hatch
    # TODO: Tell the args object to ALWAYS use a param file using args.use_param_file("@%s", use_always = True)
    
    # 5. Execute the tool
    ctx.actions.run(
        executable = ctx.executable.compiler,
        inputs = ctx.files.srcs,
        outputs = [out_file],
        # TODO: Pass the args object in a list to the arguments parameter
    )
    
    return [DefaultInfo(files = depset([out_file]))]

bulk_archiver = rule(
    implementation = _bulk_archiver_impl,
    attrs = {
        "srcs": attr.label_list(allow_files = True, mandatory = True),
        "out_name": attr.string(mandatory = True),
        "compiler": attr.label(
            default = "//:archiver_tool",
            executable = True,
            cfg = "exec",
        ),
    },
)
```

### 3. The Targets (`BUILD`)
```python
load("@rules_python//python:defs.bzl", "py_binary")
load("//:my_rules.bzl", "bulk_archiver")

py_binary(
    name = "archiver_tool",
    srcs = ["archiver_tool.py"],
    main = "archiver_tool.py",
)

bulk_archiver(
    name = "massive_archive",
    # Imagine this is 5,000 files! We will just use two for testing.
    srcs = [
        "data.json",
        "legal.txt",
    ],
    out_name = "archive_manifest.txt",
)
```

### 4. Verification
Run the build!
```bash
bazel build //:massive_archive
cat bazel-bin/archive_manifest.txt
```

**Success Criteria:** The Python tool should successfully read the inputs and write the `archive_manifest.txt` file listing `data.json` and `legal.txt`. But secretly, Bazel used a param file to pass those arguments!