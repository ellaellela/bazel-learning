# Bazel Fundamentals - Level 9.3: The Aggregator (Explanations)

## 1. The Solution

Here is the completed code for `my_aspects.bzl`:

```python
# Define the custom provider to carry our data up the graph
AuditInfo = provider(fields = ["files"])

def _aggregator_impl(target, ctx):
    if CcInfo not in target:
        return []

    out_file = ctx.actions.declare_file(target.label.name + ".audit")
    
    cc_ctx = target[CcInfo].compilation_context
    defines = cc_ctx.defines.to_list()
    
    content = "Target: {}\nDefines: {}\n".format(target.label, defines)
    ctx.actions.write(out_file, content)
    
    transitive_depsets = []
    if hasattr(ctx.rule.attr, "deps"):
        for dep in ctx.rule.attr.deps:
            # Check if the child aspect successfully returned our provider
            if AuditInfo in dep:
                transitive_depsets.append(dep[AuditInfo].files)
                
    # Merge the current file with all files from the children
    all_audit_files = depset(
        direct = [out_file],
        transitive = transitive_depsets
    )
    
    return [
        AuditInfo(files = all_audit_files),
        OutputGroupInfo(audit_reports = all_audit_files)
    ]

aggregator = aspect(
    implementation = _aggregator_impl,
    attr_aspects = ["deps"],
)
```

---

## 2. DevInfra Theory: The Data Pipeline

By returning `AuditInfo(files = all_audit_files)`, you built a reverse waterfall. 

While the aspect *propagates* from top to bottom (Analysis Phase), the data *bubbles up* from bottom to top. 

1. **The Leaf Nodes:** The aspect runs on `math_lib`. It has no dependencies. It creates a depset with just `math_lib.audit` and stores it inside `AuditInfo`.
2. **The Parent Node:** The aspect runs on `app_server`. It looks at `ctx.rule.attr.deps`. It sees `math_lib`. Because the aspect already evaluated `math_lib`, it can reach inside it, grab the `AuditInfo` provider, and extract the `math_lib.audit` file.
3. **The Command Line Hook:** Finally, the aspect on `app_server` packages its own file *plus* the children's files into one massive depset, and hands that single depset to `OutputGroupInfo`.

When Bazel evaluates `--output_groups=audit_reports`, it sees the massive depset containing all three files, generating the demand required to trigger every shadow action.

---

## 3. Deep Dive: Why `depset` is Mandatory in Aspects

You could technically write this logic using standard Python lists: `all_files = [my_file] + child_files`. Why does Bazel force you to use `depset`?

**Out of Memory (OOM) Crashes.**

Imagine your company's mono-repo has a core string-manipulation library that is depended on by 10,000 different binaries. 
If you used standard Python lists, the `string_lib.audit` file would be physically copied into 10,000 different lists in Bazel's RAM during the Analysis Phase. The memory overhead would grow exponentially, and the JVM running Bazel would crash.

A `depset` is a Directed Acyclic Graph (DAG) under the hood. When you put a child depset into the `transitive` field of a parent depset, Bazel does not copy the contents. It simply creates a pointer to the child. 
Even in a graph of 10,000 targets, `string_lib.audit` is only stored in memory *exactly once*, and 10,000 pointers point back to it. This memory architecture is the secret to Bazel's ability to analyze millions of targets in seconds.

---

## 4. Deep Dive: List Comprehensions vs. Provider Safety

If you are accustomed to Python, you might wonder why we used a traditional `for` loop instead of a cleaner list comprehension to gather the files:

```python
# Why not just do this?
transitive_files = [dep[AuditInfo].files for dep in ctx.rule.attr.deps]
```

If you use that exact line, **your build will crash** with a fatal Analysis Phase error the moment it hits a non-C++ dependency.

### The Missing Provider Trap
Remember the "Professional Filter" at the top of the Aspect?
```python
if CcInfo not in target:
    return []
```

If your `cc_binary` depends on a shell script, a code-generator tool, or a linker script, the aspect evaluates that target, sees it doesn't have `CcInfo`, and returns an empty list `[]`. That dependency **does not get the `AuditInfo` provider**.

If you blindly run `dep[AuditInfo]` in a list comprehension on a target that doesn't have it, Starlark instantly throws an exception:
`Error: target '//:some_script' does not have provider 'AuditInfo'`.

### The Safe List Comprehension
The `if AuditInfo in dep:` check acts as a safety net. It asks: *"Did the aspect successfully run and return data for this specific child?"*

If you want to use the elegant list comprehension style (which is highly encouraged in Starlark for readability), you simply move the safety check into the comprehension itself:

```python
transitive_depsets = [
    dep[AuditInfo].files 
    for dep in ctx.rule.attr.deps 
    if AuditInfo in dep  # <-- The DevInfra safety net!
]
```
This gives you the best of both worlds: compact Starlark syntax and total graph safety.

---

## 5. Deep Dive: External Workspaces and the `ls` Glob Trap

If you looked closely at Bazel's terminal output during the build, you saw this:
```text
Aspect //:my_aspects.bzl%aggregator of //:app_server up-to-date:
  bazel-bin/math_lib.audit
  bazel-bin/external/rules_cc+/empty_lib.audit
```
But when you ran `ls bazel-bin/*.audit`, the `empty_lib.audit` file was missing! Where did it go?

### The Shell Glob vs. The Sandbox
When you type `ls bazel-bin/*.audit` in your terminal, the bash `*` wildcard does not search recursively. It *only* looks at the very root of the `bazel-bin` folder. 

Because `app_server` and `math_lib` live in your main workspace, Bazel puts their `.audit` files right in the root of `bazel-bin`. 

However, `empty_lib` comes from an external toolchain repository (`@@rules_cc+`). To prevent file collisions between your company's code and third-party code, Bazel strictly sandboxes their outputs into a special `external/` sub-directory. 

If you want to see them with your own eyes, run a recursive search using `find`:
```bash
find bazel-bin -name "*.audit"
```
You will see the files safely tucked away at:
`bazel-bin/external/rules_cc+/empty_lib.audit`

This proves that your `CcInfo` provider filter is incredibly robust. It successfully identified that `empty_lib` was a C++ target (even though it is completely managed by an external toolchain), generated the shadow file, and safely isolated it in the external output directory.