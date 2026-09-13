# Bazel Fundamentals - Level 5: The Workspace Mutator (Solution & Theory)

## 1. DevInfra Theory: `build` vs. `run`

To understand how devinfra teams write tools that format code or inject headers, you have to understand exactly what Bazel does differently between its two primary commands.

### `bazel build` (The Artifact Generator)
*   **Goal:** Produce a deterministic, cached output file.
*   **Execution Environment:** Bazel runs the action inside a temporary, isolated sandbox folder.
*   **Permissions:** The sandbox has absolutely no write access to your actual Git workspace. 
*   **Caching:** If the inputs haven't changed, `bazel build` skips execution entirely. It produces no side effects.

### `bazel run` (The Escape Hatch)
*   **Goal:** Execute a compiled artifact as a host process.
*   **Phase 1 (Build):** Bazel compiles the executable (inside the sandbox) and puts it in the `bazel-bin/` directory.
*   **Phase 2 (Execute):** Bazel attaches the executable directly to your terminal. 
*   **The Magic:** Because it is running as a host terminal process, it can reach anywhere on your hard drive. To help your script find your source code, Bazel injects special environment variables into the process:
    *   `BUILD_WORKSPACE_DIRECTORY`: The absolute path to the root of your Git repository.
    *   `BUILD_WORKING_DIRECTORY`: The absolute path to the directory where you typed the `bazel run` command.

---

## 2. The Implementation

### The Script: `format_sources.py`
```python
import os
import sys
from pathlib import Path

def main():
    # 1. Grab the escape hatch path
    workspace_root = os.environ.get("BUILD_WORKSPACE_DIRECTORY")
    
    if not workspace_root:
        print("FATAL ERROR: You must execute this tool via 'bazel run'.")
        sys.exit(1)
        
    root_dir = Path(workspace_root)
    legal_file = root_dir / "legal.txt"
    
    if not legal_file.exists():
        print(f"Error: Could not find {legal_file}")
        sys.exit(1)
        
    with open(legal_file, "r") as f:
        header_content = f.read().strip()
        
    # 2. Mutate the workspace files
    mutated_count = 0
    for json_file in root_dir.glob("*.json"):
        with open(json_file, "r") as f:
            original_content = f.read()
            
        # Idempotency check
        if original_content.startswith("// Copyright"):
            continue
            
        with open(json_file, "w") as f:
            f.write(f"{header_content}\n{original_content}")
            
        print(f"Updated: {json_file.name}")
        mutated_count += 1
        
    print(f"\nSuccess! Injected headers into {mutated_count} files.")

if __name__ == "__main__":
    main()
```

### The Target: `BUILD`
```python
# The native py_binary rule creates an executable Python zip/package
py_binary(
    name = "update_headers",
    srcs = ["format_sources.py"],
    # 'data' dependencies are bundled into the executable's "runfiles" tree
    # so the script can read them at runtime.
    data = ["legal.txt"],
)
```

---

## 3. The Execution

If you run:
```bash
bazel build //:update_headers
```
Bazel will simply package the Python script and `legal.txt` into the `bazel-bin/` directory and stop. The source code will remain untouched.

To actually mutate your workspace, execute it:
```bash
bazel run //:update_headers
```
Because of the `run` command, the script reads `BUILD_WORKSPACE_DIRECTORY`, reaches outside the execution sandbox, and modifies the raw source code on your machine.