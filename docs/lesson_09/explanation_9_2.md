# Bazel Fundamentals - Level 9.2: The Generator (Explanations)

## 1. The Solution

Here is the completed code for `my_aspects.bzl`:

```python
def _security_scanner_impl(target, ctx):
    # 1. Declare a unique output file to prevent collisions
    out_file = ctx.actions.declare_file(target.label.name + ".audit")
    
    # 2. Generate the artifact
    ctx.actions.write(
        output = out_file,
        content = str(target.label)
    )
    
    # 3. Bubble the file up to the command line interface
    return [
        OutputGroupInfo(audit_reports = depset([out_file]))
    ]

security_scanner = aspect(
    implementation = _security_scanner_impl,
    attr_aspects = ["deps"],
)
```

---

## 2. DevInfra Theory: Shadow Actions

By using `ctx.actions` inside an Aspect, you have effectively created a **Shadow Build**. 

When a developer runs `bazel build //:app_server`, Bazel creates the standard action graph (compile `.cpp` into `.o`, link `.o` into an executable). 

When you attach the `security_scanner` aspect, Bazel creates a parallel, independent action graph sitting right next to the C++ actions. Because these actions are decoupled, a failure in your security scanner logic does not stop the C++ compiler from doing its job, and vice-versa. 

### Defeating File Collisions
Notice why we *had* to name the file `target.label.name + ".audit"`. 
If you used a static name like `ctx.actions.declare_file("report.txt")`, the aspect attached to `math_lib` would try to create `bazel-bin/report.txt`, and the aspect attached to `network_lib` would also try to create `bazel-bin/report.txt`. Bazel strictly forbids this and will immediately crash the Analysis Phase with an "Action Conflict Error." Tying aspect outputs to `target.label.name` guarantees safety.

---

## 3. Deep Dive: Lazy Evaluation & The Missing Files

When you ran `ls bazel-bin/*.audit`, you only saw `app_server.audit`. The files for `math_lib` and `network_lib` were missing. 

Why did this happen?

Bazel is fundamentally lazy. It operates on a strict supply-and-demand architecture. During the Analysis Phase, your aspect successfully propagated down the graph (as proven in Level 9.1) and it called `ctx.actions.write` for all three targets. But Bazel doesn't actually write files yet; it merely records the *promise* of a file in the action graph.

### The `OutputGroupInfo` Trap
To force Bazel to evaluate an action, you must create demand from the command line. You did this using `--output_groups=audit_reports`. 

However, command-line flags only apply to the **Target** you specifically requested (in this case, `//:app_server`). 
Bazel looked at `app_server`'s `OutputGroupInfo`, saw `app_server.audit`, generated it, and stopped. 

Because the aspect on `app_server` did not extract the `.audit` files from its dependencies and merge them into its own `OutputGroupInfo`, Bazel considered the dependency actions "dead weight" and **completely skipped generating them to save time**.

To fix this, an Aspect must not only declare files, but actively gather and bubble up the files from its children.