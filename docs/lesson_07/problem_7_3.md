# Bazel Fundamentals - Level 7.3: The Execution Transition (Problem Statement)

## The Scenario
Your team is building the firmware for a car's infotainment system. The car uses an ARM-based microchip. 
During the build, you need a custom code-generator tool (written in Python) to parse some JSON files and generate configuration headers. 

You run the build on your Intel/AMD laptop, passing the flag `--cpu=arm` to tell Bazel to compile the final firmware for the car. 

**The Trap:**
If you just depend on the Python tool normally, Bazel will try to build the Python runtime and the tool for the `--cpu=arm` architecture! But the tool isn't running on the car—it needs to run *on your laptop* during the build process to generate the files. If the tool is compiled for ARM, your Intel laptop cannot execute it, and the build crashes.

## DevInfra Concepts: Target vs. Execution Configurations

Bazel tracks "Configurations" (CPU architecture, OS, compiler flags). 

1. **The Target Configuration**
   This is what the user requests at the command line (e.g., `--cpu=arm`). It represents the environment where the *final output* will run. Everything in the graph uses this configuration by default.

2. **The Exec Configuration (`cfg = "exec"`)**
   This represents the environment where the *build actions* are executed (your laptop or CI runner).

3. **The Configuration Transition**
   By adding `cfg = "exec"` to a tool attribute, you tell Bazel to perform a "Transition." You are saying: *"I know the rest of this graph is being built for the car (ARM). But when you evaluate this specific tool dependency, shift it into the Execution Configuration (Intel/Mac) so I can run it right now."*

---

## The Task

You will write a `firmware_builder` rule that uses a custom code generator. You will then use Bazel's query tools to prove that the tool is built for a different CPU architecture than the firmware itself.

### 1. The Tool (`my_codegen.py`)
Create a simple Python script. (It just creates a dummy file so the action succeeds).
```python
import sys

def main():
    out_path = sys.argv[1]
    with open(out_path, "w") as f:
        f.write("Generated Firmware Config")

if __name__ == "__main__":
    main()
```

### 2. The Rule (`my_rules.bzl`)
Add this to your `.bzl` file and complete the `TODO`s.

```python
def _firmware_builder_impl(ctx):
    out_file = ctx.actions.declare_file(ctx.label.name + ".bin")
    
    ctx.actions.run(
        executable = ctx.executable.compiler,
        outputs = [out_file],
        arguments = [out_file.path]
    )
    
    return [DefaultInfo(files = depset([out_file]))]

firmware_builder = rule(
    implementation = _firmware_builder_impl,
    attrs = {
        # TODO: Add the "compiler" attribute. 
        # Make it point to "//:codegen_tool" as the default.
        # It MUST have executable = True.
        # It MUST have cfg = "exec".
    },
)
```

### 3. The Targets (`BUILD`)
Add the targets to your `BUILD` file:

```python
load("@rules_python//python:defs.bzl", "py_binary")
load("//:my_rules.bzl", "firmware_builder")

py_binary(
    name = "codegen_tool",
    srcs = ["my_codegen.py"],
    main = "my_codegen.py",
)

firmware_builder(
    name = "infotainment_firmware",
)
```

### 4. Verification (The DevInfra Magic)
We are going to simulate cross-compiling for an ARM processor using `--cpu=arm`. (Even if you are on an M1 Mac which is already ARM, Bazel will treat the generic `--cpu=arm` flag as a distinct target architecture).

Run this `cquery` command to look at the dependency graph and the transitions:

```bash
bazel cquery "deps(//:infotainment_firmware)" --cpu=arm --transitions=lite
```

**Success Criteria:** 
Look closely at the output in your terminal. You should see `//:infotainment_firmware` listed with one configuration hash (representing the ARM target). Beneath it, you should see `//:codegen_tool` listed with a *completely different configuration*, and an arrow `->` explicitly showing Bazel transitioning from the target configuration to the `exec` configuration!