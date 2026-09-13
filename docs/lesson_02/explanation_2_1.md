# Bazel Fundamentals - Level 2: The File Generator

## 1. Macros vs. Rules: The Mental Model

The biggest hurdle in Bazel is understanding that **Macros** and **Rules** run in completely different phases and have entirely different powers.

| Feature | Macro | Custom Rule |
|---|---|---|
| **Phase** | Loading Phase | Analysis Phase |
| **Primary Job** | Automate typing `BUILD` file targets. | Tell Bazel exactly how to execute a build. |
| **Under the Hood** | Just a Starlark (Python) function. | A strict API with a `ctx` (context) object. |
| **Can it run bash?** | No. | Yes, by registering an action. |
| **Can it read files?** | No, files don't exist in the Loading Phase. | No, it just passes file *pointers* to actions. |
| **Output** | Instantiates existing rules (like `genrule`). | Returns **Providers** (like `DefaultInfo`). |

**Rule of Thumb:** If you are just linking existing rules together, use a Macro. If you need a brand-new way of compiling, transforming, or packaging a file, you must write a Rule.

---

## 2. The Anatomy of a Custom Rule

A rule is highly restricted. When your `_impl` function runs during the **Analysis Phase**, it operates in a vacuum. 

*   **No I/O:** You cannot open a file, read its contents, or ping the network during Analysis. 
*   **Action Registration, Not Execution:** You don't perform work; you schedule work. You use `ctx.actions.run_shell` or `ctx.actions.write` to add nodes to Bazel's internal Action Graph. 
*   **Providers are the API:** Rules do not pass files directly to each other. They pass **Providers**. A Provider is a strictly typed data structure (like `DefaultInfo` or `CcInfo`). If Rule A depends on Rule B, Rule A inspects Rule B's providers to figure out what files were generated.

---

## 3. What is a Depset (Dependency Set)?

In the Level 2 code, we wrote: `depset([my_file])`. Why not just use a standard Python list?

As a devinfra engineer, you know that C++ projects (especially in automotive or embedded) have massive dependency trees. Imagine a binary that depends on 100 libraries, which depend on 1,000 headers. 

If Bazel used standard arrays to pass file lists up the graph, it would have to copy and concatenate those arrays at every step. This creates an $O(N^2)$ memory explosion that would instantly crash the Java heap on large monorepos.

**A `depset` is a specialized Directed Acyclic Graph (DAG).** 
*   Instead of copying lists, a `depset` just points to other `depsets`.
*   It operates in $O(1)$ time and memory during the Analysis Phase.
*   Bazel only flattens the `depset` into a real list at the very last possible second (the Execution Phase) when it actually needs to mount the files into the sandbox.

Whenever you return files in a Provider, Bazel mandates that you wrap them in a `depset`.

---

## 4. The Implementation

### `my_rules.bzl`
A custom rule consists of the rule declaration (attributes) and the implementation function (the logic).

    def _write_message_impl(ctx):
        # 1. Declare: Tell Bazel this file will exist in the sandbox.
        my_file = ctx.actions.declare_file(ctx.attr.out_name)
        
        # 2. Register: Tell Bazel HOW to make the file. 
        # (This does not write the file yet, it just adds a node to the Action Graph).
        ctx.actions.write(
            output = my_file,
            content = ctx.attr.message,
        )
        
        # 3. Return: Tell Bazel what files this rule explicitly provides.
        # If you omit this, Bazel aggressively prunes the graph and deletes your action.
        return [DefaultInfo(files = depset([my_file]))]

    write_message = rule(
        implementation = _write_message_impl,
        attrs = {
            "message": attr.string(mandatory = True),
            "out_name": attr.string(mandatory = True),
        },
    )

### `BUILD`
    load("//:my_rules.bzl", "write_message")

    write_message(
        name = "my_custom_greeting",
        message = "Hello from the Analysis Phase!",
        out_name = "output.txt",
    )

---

## 5. Debugging the Action Graph

Before you execute a build, you can inspect the exact blueprint Bazel created using `aquery` (Action Query):

    bazel aquery //:my_custom_greeting

**Understanding the Output:**
*   **`Mnemonic: FileWrite`**: Bazel categorizes every action. This tells you Bazel didn't spin up a bash shell to `echo` the text; it used an internal Java routine to write the string to disk.
*   **`Outputs: [bazel-out/.../bin/output.txt]`**: Proof of sandboxing. Bazel isolated the output in the private cache tree, not your source code folder.

To actually execute the action and generate the file:

    bazel build //:my_custom_greeting
