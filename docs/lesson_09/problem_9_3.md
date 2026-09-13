# Bazel Fundamentals - Level 9.3: The Aggregator (Problem Statement)

## The Scenario
Your security scanner is generating files, but it's dropping the transitive dependencies. To make this tool production-ready, it must collect every `.audit` file from the bottom of the graph and bubble them all the way up to the top-level binary. 

Furthermore, you are going to extract the actual compiler flags from the native C++ targets along the way using Bazel's built-in `CcInfo` provider.

## DevInfra Concepts: Aspect Aggregation

1. **Custom Providers in Aspects**
   Just like Rules, Aspects can return custom Providers. This is how an Aspect on a parent target reads the data collected by the Aspect on a child target.
   ```python
   AuditInfo = provider(fields = ["files"])
   ```

2. **Accessing Child Aspect Data (`ctx.rule.attr`)**
   When an Aspect evaluates a target, it can look at the target's dependencies. Because the Aspect already ran on those dependencies, it can extract the `AuditInfo` provider from them!
   ```python
   # ctx.rule.attr gives you access to the target's original attributes
   for dep in ctx.rule.attr.deps:
       # Extract the provider returned by the aspect on the child
       child_depset = dep[AuditInfo].files 
   ```

3. **Merging Depsets**
   You will combine your memory-efficient `depset` knowledge from Level 6 with your Aspect. The parent merges its own file with the children's files.
   ```python
   transitive_files = [dep[AuditInfo].files for dep in ctx.rule.attr.deps]
   combined = depset(direct = [my_file], transitive = transitive_files)
   ```

---

## The Task

### 1. The Aspect (`my_aspects.bzl`)
Open `my_aspects.bzl`. We are going to upgrade our scanner into an aggregator. Complete the `TODO`s.

```python
# 1. Define the custom provider to carry our data up the graph
AuditInfo = provider(fields = ["files"])

def _aggregator_impl(target, ctx):
    # Only process C++ targets (The professional filter!)
    if CcInfo not in target:
        return []

    # 1. Declare the output file for this specific target
    out_file = ctx.actions.declare_file(target.label.name + ".audit")
    
    # Let's extract real C++ data! We'll grab the compilation context.
    cc_ctx = target[CcInfo].compilation_context
    defines = cc_ctx.defines.to_list() # Get the preprocessor defines
    
    # 2. Write the file (including the C++ defines)
    content = "Target: {}\nDefines: {}\n".format(target.label, defines)
    ctx.actions.write(out_file, content)
    
    # 3. Collect the depsets from the children
    transitive_depsets = []
    
    # NOTE: Not all targets have 'deps' (e.g. leaf nodes). We must check first using hasattr!
    if hasattr(ctx.rule.attr, "deps"):
        for dep in ctx.rule.attr.deps:
            # TODO: Check if the child has our AuditInfo provider. 
            # If it does, append dep[AuditInfo].files to transitive_depsets
            pass
            
    # 4. Merge everything into one massive depset
    # TODO: Create a depset. Put `[out_file]` in the direct field. Put `transitive_depsets` in the transitive field.
    # Assign it to a variable named `all_audit_files`
    
    # 5. Return BOTH the custom provider (for the parents) AND OutputGroupInfo (for the CLI)
    return [
        AuditInfo(files = all_audit_files),
        OutputGroupInfo(audit_reports = all_audit_files)
    ]

aggregator = aspect(
    implementation = _aggregator_impl,
    attr_aspects = ["deps"],
)
```

### 2. Verification
Run the exact same command, but point it to your new `aggregator` aspect:

```bash
bazel build //:app_server \
  --aspects=//:my_aspects.bzl%aggregator \
  --output_groups=audit_reports
```

Then, list the `bazel-bin` directory:
```bash
ls bazel-bin/*.audit
```

**Success Criteria:** 
This time, because you correctly aggregated the files, Bazel's execution phase knows it needs *all of them*. You should see `app_server.audit`, `math_lib.audit`, and `network_lib.audit` successfully generated!