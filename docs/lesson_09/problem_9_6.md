# Bazel Fundamentals - Level 9.6: Parameterized Aspects (Problem Statement)

## The Scenario
Your security scanner is a massive success. However, different teams want to customize the output header of the audit files. The Android team wants their audit files to say `"--- ANDROID SCAN ---"`, and the Backend team wants `"--- BACKEND SCAN ---"`. 

You cannot hardcode this in Starlark anymore. You need to expose a parameter in the `BUILD` file so developers can configure the Aspect's behavior directly.

## DevInfra Concepts: The Attribute Handshake

Aspects can declare attributes (like `attr.string()`), just like Rules do. But how does the Aspect get the value? 

**The Rule and the Aspect must share the exact same attribute definition.** 

When Bazel intercepts the target and fires the Aspect, it copies the value of the attribute from the Rule's instantiation in the `BUILD` file and hands it to the Aspect.

```python
# 1. The Aspect defines it
my_aspect = aspect(
    implementation = _impl,
    attrs = {"mode": attr.string(values = ["fast", "strict"])}
)

# 2. The Rule defines the exact same attribute
my_rule = rule(
    implementation = _rule_impl,
    attrs = {
        "target": attr.label(aspects = [my_aspect]),
        "mode": attr.string(values = ["fast", "strict"]) # The Handshake!
    }
)
```
When the aspect runs, it can access the value using `ctx.attr.mode`!

---

## The Task

We are going to build a customizable scanner. Create a new file called `parameterized.bzl`.

### 1. The Aspect & Rule (`parameterized.bzl`)
Copy this code and complete the `TODO`s. We will use `ctx.attr.header_text` to customize the file contents.

```python
AuditInfo = provider(fields = ["files"])

# ==========================================
# 1. THE ASPECT
# ==========================================
def _param_aspect_impl(target, ctx):
    out_file = ctx.actions.declare_file(target.label.name + "_param.audit")
    
    # TODO: Extract the parameter passed down from the Rule!
    # Access the `header_text` attribute from `ctx.attr`.
    # Assign it to a variable named `custom_header`.
    
    # Write the file using the custom parameter
    content = "{}\nTarget: {}\n".format(custom_header, target.label)
    ctx.actions.write(out_file, content)

    # Standard Aggregation
    transitive_depsets = []
    if hasattr(ctx.rule.attr, "deps"):
        transitive_depsets = [
            dep[AuditInfo].files 
            for dep in ctx.rule.attr.deps 
            if AuditInfo in dep
        ]
        
    all_files = depset(direct = [out_file], transitive = transitive_depsets)
    return [AuditInfo(files = all_files)]

param_aspect = aspect(
    implementation = _param_aspect_impl,
    attr_aspects = ["deps"],
    # TODO: Define the attribute on the Aspect.
    # Name it "header_text". Make it an attr.string(default = "DEFAULT SCAN").
    # It goes right here inside the attrs dictionary:
    attrs = {
        
    }
)

# ==========================================
# 2. THE RULE
# ==========================================
def _param_rule_impl(ctx):
    out_file = ctx.actions.declare_file(ctx.label.name + "_master_report.txt")
    
    fragment_depset = ctx.attr.target[AuditInfo].files
    
    args = ctx.actions.args()
    args.add_all(fragment_depset)
    
    # Just concatenate all the small audit files into one master report
    ctx.actions.run_shell(
        inputs = fragment_depset,
        outputs = [out_file],
        command = "cat $@ > " + out_file.path,
        arguments = [args]
    )
    
    return [DefaultInfo(files = depset([out_file]))]

param_scanner = rule(
    implementation = _param_rule_impl,
    attrs = {
        "target": attr.label(aspects = [param_aspect]),
        # TODO: Define the EXACT SAME attribute on the Rule so the handshake works!
        # Name it "header_text", make it an attr.string(default = "DEFAULT SCAN").
    }
)
```

### 2. The Implementation (`BUILD`)
Let's instantiate your new customizable tool in the `BUILD` file. You can reuse the C++ targets we built in Level 9.5!

```python
load("//:parameterized.bzl", "param_scanner")

# We run the scanner on our mixed_app, and pass in our custom configuration!
param_scanner(
    name = "backend_scan",
    target = ":mixed_app",
    header_text = "--- STRICT BACKEND SCAN ---",
)
```

### 3. Verification
Run the build targeting your newly configured scanner:

```bash
bazel build //:backend_scan
```

Then, print out the final master report to see if the string made it all the way down the graph:
```bash
cat bazel-bin/backend_scan_master_report.txt
```

**Success Criteria:**
You should see your master report containing the file fragments. At the top of *each* target's fragment, you should see the exact string `"--- STRICT BACKEND SCAN ---"` that you injected from the `BUILD` file!