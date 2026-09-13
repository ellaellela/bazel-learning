# Bazel Fundamentals - Level 9.8: External Toolchain Injection (Problem Statement)

## The Scenario
Your company requires a mandatory vulnerability scan on every single software component in the monorepo before deployment. You have decided to use an external security binary (to simulate tools like Trivy or OSV-Scanner). 

You need to write an Aspect that executes this external scanner on every node in the graph during the build process, generating an isolated `.scan` report for each target.

## DevInfra Concepts: Executable Attributes & `cfg = "exec"`

To inject an external tool into an Aspect, we use a private attribute (as we learned in Level 9.6). But we must add two critical flags to tell Bazel that this isn't just a data file—it is a tool we intend to run.

```python
"_scanner": attr.label(
    default = "//tools:mock_trivy",
    executable = True, 
    cfg = "exec"
)
```

1. **`executable = True`**: Tells Bazel this label points to a runnable binary (like a `sh_binary`, `py_binary`, or `cc_binary`). This makes the binary available in Starlark via `ctx.executable._scanner`.
2. **`cfg = "exec"`**: This is the bedrock of DevInfra cross-compilation. If you are building a C++ app for an embedded ARM processor in a car, the graph's target architecture is ARM. But your security scanner needs to run *on your local laptop* (or CI runner) during the build. `cfg = "exec"` forces Bazel to build/resolve the scanner for the **Execution Platform** (your Mac/Linux machine), not the Target Platform.

Once injected, you use `ctx.actions.run()` (not `run_shell`) to execute the tool natively.

---

## The Task

We will create a mock security scanner and wire it into an Aspect.

### 1. The External Tool (`tools/mock_trivy.sh` & `tools/BUILD`)
First, let's build the tool. Create a directory named `tools`.
Inside `tools/`, create a script called `mock_trivy.sh` and make it executable (`chmod +x tools/mock_trivy.sh`):

```bash
#!/bin/bash
# A mock security scanner. 
# Usage: ./mock_trivy.sh <target_name> <output_file>

TARGET=$1
OUTPUT_FILE=$2

echo "======================================" > "$OUTPUT_FILE"
echo "TRIVY/OSV-SCANNER VULNERABILITY REPORT" >> "$OUTPUT_FILE"
echo "Target: $TARGET" >> "$OUTPUT_FILE"
echo "Status: NO CVEs DETECTED (CLEAN)" >> "$OUTPUT_FILE"
echo "======================================" >> "$OUTPUT_FILE"
```

Now, expose it to Bazel by creating `tools/BUILD`:
```python
sh_binary(
    name = "mock_trivy",
    srcs = ["mock_trivy.sh"],
    visibility = ["//visibility:public"],
)
```

### 2. The Aspect (`security_pipeline.bzl`)
At the root of your workspace, create `security_pipeline.bzl` and complete the `TODO`s.

```python
ScanInfo = provider(fields = ["reports"])

def _scanner_aspect_impl(target, ctx):
    report_file = ctx.actions.declare_file(target.label.name + ".scan")
    
    args = ctx.actions.args()
    args.add(target.label.name) # Arg 1: Target Name
    args.add(report_file.path)  # Arg 2: Output File Path
    
    # TODO: Execute the external tool!
    # Use ctx.actions.run() to execute the scanner.
    ctx.actions.run(
        outputs = [report_file],
        arguments = [args],
        executable = ctx.executable._scanner,
        progress_message = "Running security scan on {}...".format(target.label.name)
    )

    # Standard Aggregation
    transitive_depsets = []
    if hasattr(ctx.rule.attr, "deps"):
        transitive_depsets = [
            dep[ScanInfo].reports 
            for dep in ctx.rule.attr.deps 
            if ScanInfo in dep
        ]
        
    all_reports = depset(direct = [report_file], transitive = transitive_depsets)
    return [ScanInfo(reports = all_reports)]

security_aspect = aspect(
    implementation = _scanner_aspect_impl,
    attr_aspects = ["deps"],
    attrs = {
        # TODO: Define the private attribute for the external tool.
        # Name it "_scanner".
        # Point the default to "//tools:mock_trivy".
        # Make sure to set executable = True and cfg = "exec"!
    }
)

def _pipeline_rule_impl(ctx):
    out_file = ctx.actions.declare_file(ctx.label.name + "_master_scan.txt")
    reports_depset = ctx.attr.target[ScanInfo].reports
    
    args = ctx.actions.args()
    args.add_all(reports_depset)
    
    ctx.actions.run_shell(
        inputs = reports_depset,
        outputs = [out_file],
        command = "cat $@ > " + out_file.path,
        arguments = [args]
    )
    
    return [DefaultInfo(files = depset([out_file]))]

security_pipeline = rule(
    implementation = _pipeline_rule_impl,
    attrs = {
        "target": attr.label(aspects = [security_aspect]),
    }
)
```

### 3. Verification (`BUILD`)
Back in your root `BUILD` file, instantiate the pipeline on the `mixed_app` from Level 9.5.

```python
load("//:security_pipeline.bzl", "security_pipeline")

security_pipeline(
    name = "audit_mixed_app",
    target = ":mixed_app",
)
```

Run the pipeline:
```bash
bazel build //:audit_mixed_app
```

**Success Criteria:**
1. Watch the terminal output closely—you should see your custom `progress_message` ("Running security scan on...") print out as the Aspect dynamically fires the tool.
2. Run `cat bazel-bin/audit_mixed_app_master_scan.txt`. You should see a concatenated Trivy/OSV report for `mixed_app` and `core_logic`!