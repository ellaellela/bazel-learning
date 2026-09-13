# Bazel Fundamentals - Level 5: The Workspace Mutator (Problem Statement)

## The Scenario
The legal department is happy with your `inject_header` rule, but they changed their mind on the implementation. They do not want the copyright header injected *during* the build. They want the header physically written into the source `.json` files in the repository so the legal text is tracked in Git.

## The Trap: The Hermetic Seal
If you try to write a Custom Rule that modifies files in your source tree, Bazel will block it or it will cause an infinite build loop. `bazel build` is strictly sandboxed. 

To mutate the workspace, you must use a script executed via the `bazel run` escape hatch. 

## The Task
Write a standalone Python tool that injects `legal.txt` into every `.json` file in your source tree, and wire it up to Bazel.

1.  **The Script (`format_sources.py`):**
    *   Write a Python script that finds all `.json` files in your workspace root.
    *   Read the contents of `legal.txt`.
    *   Prepend the legal text to the top of the `.json` files and save them. 
    *   *Constraint:* The script must be idempotent (don't add the header if it already exists).

2.  **The BUILD Target:**
    *   You do not need `my_rules.bzl` for this. Use the native `py_binary` rule in your `BUILD` file.
    *   Name the target `update_headers`.
    *   Ensure the script has access to `legal.txt` at runtime.

3.  **The Execution:**
    *   Run the script using Bazel. Check your `git status` to prove it worked.

### 💡 DevInfra Clues
*   When executing via `bazel run`, Bazel sets a special environment variable called `BUILD_WORKSPACE_DIRECTORY`. Your Python script must read this to know where the Git repository lives on your hard drive (`os.environ.get(...)`).
*   To make non-source files (like `legal.txt`) available to a `py_binary` at runtime, you must pass them to the `data = [...]` attribute.