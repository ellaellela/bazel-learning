# Bazel Fundamentals - Level 9.6: Parameterized Aspects (Explanations)

## 1. The Solution

Here is the completed Starlark code for `parameterized.bzl`, including the strict `values` constraint required by Bazel's Analysis Phase:

```python
AuditInfo = provider(fields = ["files"])

# ==========================================
# 1. THE ASPECT
# ==========================================
def _param_aspect_impl(target, ctx):
    out_file = ctx.actions.declare_file(target.label.name + "_param.audit")
    
    # Extract the parameter passed down from the Rule (from the Aspect's backpack)
    custom_header = ctx.attr.header_text
    
    content = "{}\nTarget: {}\n".format(custom_header, target.label)
    ctx.actions.write(out_file, content)

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
    attrs = {
        "header_text": attr.string(
            values = ["--- STRICT BACKEND SCAN ---", "DEFAULT SCAN"],
            default = "DEFAULT SCAN"
        )
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
        # The Handshake: Exact same attribute definition as the Aspect
        "header_text": attr.string(
            values = ["--- STRICT BACKEND SCAN ---", "DEFAULT SCAN"],
            default = "DEFAULT SCAN"
        )
    }
)
```

---

## 2. DevInfra Theory: The Attribute Handshake

To pass configuration from a `BUILD` file into an Aspect as it traverses the graph, the Rule and the Aspect must share the exact same attribute definition. 

When you instantiate `param_scanner` and set `header_text`, Bazel acts as a secure middleman. Before firing the Aspect down the graph, Bazel copies the value from the Rule's context and injects it into the Aspect's context. This allows you to create highly reusable DevInfra tools (like security scanners) that behave differently depending on how the developer configures the top-level Rule.

---

## 3. Deep Dive: `ctx.attr` vs `ctx.rule.attr`

When writing Aspects, understanding the difference between `ctx.attr` and `ctx.rule.attr` is the difference between a successful traversal and a fatal crash.

**`ctx.rule.attr` (The Target's State)**
This inspects the attributes of the underlying rule the Aspect is currently visiting. If the Aspect is visiting a `cc_library`, `ctx.rule.attr.srcs` returns the C++ source files. 
If you tried to read `ctx.rule.attr.header_text`, the build would crash. A `cc_library` does not have a `header_text` attribute natively, so Starlark throws an error.

**`ctx.attr` (The Aspect's State)**
This inspects the attributes of the Aspect itself. Because the Aspect is passed the `header_text` parameter at the very top of the graph (via the Rule-Aspect Handshake), it stores that value in `ctx.attr`. As the Aspect jumps from node to node down the dependency tree, it carries this parameter with it, allowing you to pass external configuration all the way to the bottom of the graph without ever modifying the native C++ targets.

**The Strict Handshake**
Bazel strictly enforces this parameter passing. If an Aspect defines a public attribute in its `attrs` dictionary, any custom Rule that applies that Aspect *must* define an attribute with the exact same name and type. Bazel acts as the secure middleman, copying the value from the Rule's instantiation in the `BUILD` file into the Aspect's `ctx.attr` before the traversal begins.

---

## 4. Architectural Diagram: The Attribute Handshake

Here is the visual flow of how Bazel extracts a parameter from the `BUILD` file, validates the handshake, and safely injects it into the Aspect's isolated context for the graph traversal.

```text
=========================================================================
                      THE ATTRIBUTE HANDSHAKE
=========================================================================

1. INVOCATION
   [ BUILD FILE ] 
     param_scanner(
         name = "backend_scan",
         target = ":mixed_app",
         header_text = "--- STRICT BACKEND SCAN ---"  <-- Developer configures
     )
            |
            v
2. VALIDATION (Bazel's Analysis Engine)
   Bazel intercepts the target request. 
   - Rule `param_scanner` requests `param_aspect`.
   - Rule provides `header_text`.
   - Does Aspect `param_aspect` define `header_text`? YES. (API Transparent)
            |
            v
3. THE COPY (Loading the Backpack)
   Bazel securely copies the Rule's string and locks it inside the 
   Aspect's internal state (`ctx.attr`).
   
   [ ASPECT BACKPACK: ctx.attr ]
     { "header_text": "--- STRICT BACKEND SCAN ---" }
            |
            |   (Aspect begins traversing the graph carrying the backpack)
            |
            v
4. THE TRAVERSAL (Independent Execution)
   The Aspect visits the nodes. It explicitly ignores the targets' native 
   attributes and pulls configuration directly from its own backpack.

   [ C++ TARGET: //:mixed_app ]               [ C++ TARGET: //:core_logic ]
   (ctx.rule.attr = srcs, deps, data)         (ctx.rule.attr = srcs, hdrs)

     Aspect executes.                           Aspect executes.
     Looks in ctx.rule.attr? NO. (Crash)        Looks in ctx.rule.attr? NO. (Crash)
     Pulls from ctx.attr? YES.                  Pulls from ctx.attr? YES.
            |                                          |
            v                                          v
     Writes "--- STRICT BACKEND SCAN ---"       Writes "--- STRICT BACKEND SCAN ---"
```

---

## 5. Deep Dive: The Analysis Cache and the `values` Restriction

When you declare a parameter on an Aspect, Bazel forces you to use the `values` list restriction (e.g., `values = ["A", "B"]`). 

Bazel heavily caches the Analysis Phase. The cache key for an Aspect includes the target label *and* the Aspect's parameters. If Bazel allowed `attr.string()` to accept any arbitrary string (like a timestamp or a user's name), it would have to generate and cache a brand-new execution graph for every unique string. This would cause an exponential explosion in memory usage, inevitably leading to an Out-Of-Memory (OOM) crash on large repositories. 

The `values` list mathematically bounds the maximum number of graph variations Bazel will ever have to cache. *(Note: In production DevInfra, we pass strict enums like `mode = "android"` and map them to strings inside the Starlark logic).*

---

## 6. Deep Dive: Default Shadowing

If the Rule and the Aspect declare the same attribute but define different `default` values, the build will still succeed, but it triggers a precedence hierarchy:

```python
# The Aspect
attrs = {"header_text": attr.string(default = "DEFAULT SCAN")}

# The Rule
attrs = {"header_text": attr.string(default = "DEFAULT")}
```

**Rule Invocations (Shadowing)**
When the Aspect is triggered by the Rule (the "Handshake"), the Rule's resolved value is passed down. If the developer omits the attribute in the `BUILD` file, the target receives the Rule's default (`"DEFAULT"`). Bazel passes `"DEFAULT"` to the Aspect, entirely shadowing `"DEFAULT SCAN"`.

**Command-Line Invocations (Fallback)**
Aspects can also be invoked directly from the CLI (`bazel build //:app --aspects=...`). In this scenario, there is no middleman Rule to supply a value. Bazel will fall back to the Aspect's natively defined `default = "DEFAULT SCAN"`. Defining defaults on the Aspect ensures it remains portable across both execution methods.

---

## 7. Deep Dive: API Transparency and Private Attributes

If an Aspect defines a public attribute, Bazel forces the applying Rule to define it as well. If the Rule misses it, Bazel does *not* fall back to the Aspect's default—it instantly crashes the build with a load-time error.

**Why the strictness? API Transparency.**
When a developer uses a Rule in a `BUILD` file, they are only looking at the Rule's public API. They shouldn't have to guess if an invisible Aspect is running underneath with hidden configuration options. By forcing the Rule to explicitly mirror the Aspect's attributes, Bazel guarantees the developer can see and configure every available parameter.

### The Escape Hatch: Private Attributes
What if your Aspect or Rule *needs* an attribute (like a helper script, a compiler tool, or an internal configuration file) but you do **not** want developers to override it in the `BUILD` file?

You use a **Private Attribute** by prefixing the name with an underscore (`_`). 
*Note: This mechanic applies identically to both `aspect()` and `rule()` definitions.*

```python
param_aspect = aspect(
    implementation = _param_aspect_impl,
    attrs = {
        # Public: The Rule MUST mirror this. Developers CAN set this in the BUILD file.
        "header_text": attr.string(default = "DEFAULT SCAN", values = ["..."]), 
        
        # Private: The Rule is FORBIDDEN from mirroring this. 
        # Developers CANNOT set this in the BUILD file. It must have a default.
        "_helper_tool": attr.label(default = "//tools:my_security_script") 
    }
)
```

By adding the underscore, you tell Bazel's Analysis engine: *"This is an internal implementation detail. Do not ask the Rule for this, and do not let the user touch it. Just quietly pass my default value directly into `ctx.attr`."*

### Private Attribute Isolation

What happens if both the Rule and the Aspect define a private attribute with the exact same name?

```python
# The Aspect defines a private tool
param_aspect = aspect(
    attrs = { "_helper_tool": attr.label(default = "//tools:aspect_script") }
)

# The Rule ALSO defines a private tool
param_scanner = rule(
    attrs = { 
        "target": attr.label(aspects = [param_aspect]),
        "_helper_tool": attr.label(default = "//tools:rule_script") 
    }
)
```

**Result: Total Isolation.**
This does not crash, and neither shadows the other. 

The Rule-Aspect Handshake (where Bazel enforces matching definitions and copies values) **only applies to public attributes**. Private attributes are completely excluded from the handshake. 

Bazel treats private attributes like private member variables in object-oriented programming. They are strictly localized to their own execution contexts. When the Aspect runs, `ctx.attr._helper_tool` points exclusively to `//tools:aspect_script`. When the Rule runs, its `ctx.attr._helper_tool` points exclusively to `//tools:rule_script`. They safely coexist without ever interfering with one another.