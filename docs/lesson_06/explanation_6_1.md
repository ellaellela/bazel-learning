# Bazel Fundamentals - Level 6.1: The Schema & The Exporter (Explanations)

## 1. The Solution

Here is the completed code for `my_rules.bzl`:

```python
TeamMetadataInfo = provider(
    doc = "Holds ownership and versioning data for a target",
    fields = ["owner", "version"]
)

def _metadata_exporter_impl(ctx):
    # 1. Declare a dummy output file
    out_file = ctx.actions.declare_file(ctx.label.name + ".txt")
    ctx.actions.write(
        output = out_file,
        content = "This target is owned by: " + ctx.attr.owner,
    )
    
    # 2. Instantiate your custom provider using ctx.attr
    my_data = TeamMetadataInfo(
        owner = ctx.attr.owner,
        version = ctx.attr.version,
    )
    
    # 3. Return BOTH DefaultInfo and your custom provider in a list
    return [
        DefaultInfo(files = depset([out_file])),
        my_data
    ]

metadata_exporter = rule(
    implementation = _metadata_exporter_impl,
    attrs = {
        "owner": attr.string(mandatory = True),
        "version": attr.string(default = "1.0.0"),
    },
)
```

---

## 2. DevInfra Theory: What exactly is a Provider?

In Bazel, targets form a **Directed Acyclic Graph (DAG)**. 

When Target B depends on Target A, they need a way to communicate. A rule's implementation function is strictly isolated; it cannot reach into another rule and look at its internal variables. 

Instead, at the end of the Analysis Phase, Target A must pack everything it wants to share into structured boxes and hand them back to Bazel. These boxes are called **Providers**.

*   `DefaultInfo`: A built-in provider that holds actual files. 
*   `TeamMetadataInfo`: A custom provider you just created to hold strings.

When Target B asks Bazel for Target A, Bazel just hands Target B the list of Providers that Target A returned. 

---

## 3. Breaking Down the Code

### Strict Typing (`fields`)
By defining `fields = ["owner", "version"]` in the `provider()` declaration, you created a strict schema. If you accidentally typed `my_data = TeamMetadataInfo(owner_name = "...")` inside your rule, Bazel would immediately crash during the Analysis Phase. This protects massive codebases from silent typos.

### The Return List
A rule always returns a list of providers. Before today, you only returned `[DefaultInfo(...)]`. By returning `[DefaultInfo(...), my_data]`, you are telling Bazel: *"Anyone who depends on me gets these files, PLUS this custom metadata dictionary."*

---

## 4. Demystifying `cquery`

You used this command to verify your work:
```bash
bazel cquery //:core_network_lib --output=starlark --starlark:expr="providers(target)"
```

As a DevInfra engineer, mastering Bazel's query tools is critical. There are three different query commands, corresponding directly to Bazel's three phases:

1.  **`bazel query` (Loading Phase):** Reads the raw text of your `BUILD` files. It can tell you what targets exist, but it doesn't know what files they generate or what providers they return.
2.  **`bazel cquery` (Analysis Phase):** "Configured Query". This looks at the graph *after* Bazel has run your Python-like Starlark rule implementations. It knows exactly what Providers were returned and how the target was configured (e.g., if it was compiled for Mac vs Linux). 
3.  **`bazel aquery` (Execution Phase):** "Action Query". This looks at the sandboxes and bash commands Bazel is about to run. (You used this in Level 4 to debug the sandbox).

### The Flags
Because Providers are generated during the Analysis Phase, we must use `cquery` to see them.
*   `--output=starlark`: By default, `cquery` just spits out target names. This flag tells Bazel to format the output using the Starlark language.
*   `--starlark:expr="providers(target)"`: This is a mini-script executed against the node in the graph. `target` represents the Configured Target object. The `providers()` function reaches into that object and extracts the dictionary of all returned providers. 

When you ran it, Bazel dumped the internal memory state of that graph node directly to your terminal!