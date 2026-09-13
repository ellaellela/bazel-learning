# Bazel Fundamentals - Level 9.8: External Toolchain Injection (Explanations)

## 1. The Solution

Here is the completed Starlark code for injecting an external binary into an Aspect:

```python
ScanInfo = provider(fields = ["reports"])

def _scanner_aspect_impl(target, ctx):
    report_file = ctx.actions.declare_file(target.label.name + ".scan")
    
    args = ctx.actions.args()
    args.add(target.label.name)
    args.add(report_file.path) 
    
    # Executing the injected tool natively
    ctx.actions.run(
        outputs = [report_file],
        arguments = [args],
        executable = ctx.executable._scanner,
        progress_message = "Running security scan on {}...".format(target.label.name)
    )

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
        # The External Tool Injection
        "_scanner": attr.label(
            default = "//tools:mock_trivy",
            executable = True, 
            cfg = "exec"
        )
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

---

## 2. DevInfra Theory: Toolchain Injection

In enterprise DevInfra, Starlark is strictly used for graph orchestration, not heavy computation. When you need to parse ASTs, scan for CVEs, or generate complex documentation, you write a dedicated binary (in Go, C++, or Rust), and inject it into the Starlark Aspect.

This is done using a private attribute with two critical modifiers:
1.  **`executable = True`**: Informs Bazel that this label points to a runnable binary (like a `sh_binary` or `cc_binary`), exposing it to Starlark via `ctx.executable`.
2.  **`cfg = "exec"`**: The bedrock of DevInfra cross-compilation. It forces Bazel to compile the tool for the **Execution Platform** (e.g., your local x86/ARM Mac) rather than the Target Platform (e.g., an embedded vehicle processor), ensuring the tool can actually run locally during the build process.

---

## 3. Deep Dive: `ctx.actions.run` vs `ctx.actions.run_shell`

*   **`run_shell`**: Used for basic Unix commands (`cat`, `zip`, `echo`). It spins up a bash subshell, which is lightweight but limited, and requires you to write bash scripts directly inside Starlark strings (which is prone to escaping errors).
*   **`run`**: Used to execute natively compiled binaries or scripts injected via attributes. It does not spawn a bash subshell. It invokes the executable directly in a highly isolated sandbox, passing the `args` array securely. This is significantly safer, highly deterministic, and standard practice for custom DevInfra tooling.

---

## 4. Deep Dive: Externalizing Rules (`rules_shell`)
Just like `rules_python` and `rules_cc`, the core shell rules (`sh_binary`, `sh_test`) have been extracted from the monolithic Bazel binary in recent versions. To execute a bash script as a first-class Bazel target, you must load `rules_shell` in your `MODULE.bazel` file and explicitly load `sh_binary` into your `BUILD` file.

---

## 5. Debugging Fast Actions: The `-s` Flag

Bazel’s default terminal UI is optimized for speed and minimal visual noise. As a result, actions that take less than a millisecond will overwrite their `progress_message` before the terminal can even render it to the screen.

When you need to verify what Bazel is actually running under the hood, append the **Subcommands flag (`-s`)**:

```bash
bazel build //:audit_mixed_app -s
```

This flag disables the fleeting terminal animations and forces Bazel to print a permanent, verbose log of *every single action* it executes. You will see your exact `progress_message`, the path to the executable, and the fully resolved command-line arguments. It is an essential DevInfra debugging tool when an injected binary is crashing or receiving the wrong parameters.