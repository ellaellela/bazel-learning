# Bazel Fundamentals - Level 1: Wrapper Macros

## 1. The Core Concept: The Loading Phase

In Bazel, a **macro** is not a true rule. It is simply a Starlark (Python-like) function that is evaluated during the **Loading Phase**. 

Bazel evaluates builds in three distinct phases:
1. **Loading Phase:** Bazel executes `.bzl` files and `BUILD` files to figure out what targets exist. Macros run here. They programmatically stamp out standard Bazel rules (like `genrule`, `cc_binary`, etc.).
2. **Analysis Phase:** Bazel builds the dependency graph and determines exactly what actions (compilation, copying, zipping) need to happen.
3. **Execution Phase:** Bazel actually runs the bash commands, compilers, or tools inside isolated sandboxes to produce the output files.

Because macros run in the Loading Phase, they cannot read the contents of files or execute bash commands themselves. They can only generate rules that will execute later.

---

## 2. The Implementation

### `MODULE.bazel`
An empty file at the root of the repository. In Bazel 7+, this strictly defines the directory as a Bazel workspace.

### `my_macros.bzl`
The macro definition. It iterates over a list and instantiates a native `genrule` for each item.

    def generate_multiple_texts(name_prefix, texts):
        """
        A macro that generates a text file for each string in a list.
        Evaluated during the Loading Phase.
        """
        for text_item in texts:
            native.genrule(
                # 1. Target Name: Must be unique in the BUILD file
                name = name_prefix + "_" + text_item,
                
                # 2. Outputs: The files this rule promises to create
                outs = [text_item + ".txt"],
                
                # 3. Command: The bash execution that creates the output
                cmd = "echo 'Hello from " + text_item + "' > $@",
            )

### `BUILD`
The file that loads and invokes the macro.

    load("//:my_macros.bzl", "generate_multiple_texts")

    generate_multiple_texts(
        name_prefix = "my_auto_gen",
        texts = ["alpha", "beta", "gamma"],
    )

---

## 3. Technical Deep Dive

### Make Variables and Sandboxing (`$@`)
In the `cmd` attribute: `"echo 'Hello from " + text_item + "' > $@"`
*   The first half is **Starlark string concatenation**. It evaluates immediately in the Loading Phase to a literal string (e.g., `"echo 'Hello from alpha' > $@"`).
*   The `$@` is a **Bazel Make Variable**. It represents the path to the declared output file (`outs`). 
*   **Why use `$@`?** Bazel executes actions in strict, temporary sandboxes. The output file is not generated in your source tree; it is generated deep in the `bazel-out/` cache directory. Using `$@` ensures the shell command writes to the exact sandbox path Bazel expects, regardless of the host OS architecture.

### Verifying the Graph
Because macros don't exist as targets, running `bazel query //...` will not show `generate_multiple_texts`. Instead, it will reveal the dynamically stamped targets:
*   `//:my_auto_gen_alpha`
*   `//:my_auto_gen_beta`
*   `//:my_auto_gen_gamma`

---

## 4. Troubleshooting: Corporate Proxies & JVM Certificates

If you are on a corporate machine with an SSL inspection proxy (Zscaler, Netskope, etc.), Bazel will fail to download external dependencies (like the `platforms` module) with a `SunCertPathBuilderException`.

**The Cause:** Bazel runs its background server on an embedded Java Virtual Machine (JVM). By default, the JVM uses its own isolated certificate trust store and ignores the custom Root CA your IT department installed in your OS Keychain.

**The Fix:** Create a `.bazelrc` file at the workspace root to force the JVM to use the macOS System Keychain:

    # .bazelrc
    startup --host_jvm_args=-Djavax.net.ssl.trustStoreType=KeychainStore
    startup --host_jvm_args=-Djavax.net.ssl.trustStore=/Library/Keychains/System.keychain

*Note: You must run `bazel shutdown` after adding this file to kill the existing daemon so it picks up the new startup flags on the next run.*

---

## 5. Building and Finding Outputs

### The Build Command
To execute one of your dynamically generated targets, run:

    bazel build //:my_auto_gen_alpha

*   `//` signifies the root of the workspace.
*   `:` separates the package path (the directory containing the `BUILD` file) from the specific target name.

### Locating the Generated Files
Bazel strictly avoids polluting your source code directory. It executes builds inside a sandbox and stores the artifacts deep inside a local cache (which is how remote caching with Buildbarn/BuildBuddy scales so well).

To make these files easily accessible, Bazel creates several convenience symlinks in your workspace root (like `bazel-bin`, `bazel-out`, and `bazel-testlogs`).

To view the output of your build, inspect the `bazel-bin` directory:

    cat bazel-bin/alpha.txt

*(Note: If you run `ls -la bazel-bin`, you will see it is actually a symlink pointing into the private cache tree, e.g., `/private/var/tmp/.../bazel-out/darwin-arm64-fastbuild/bin`.)*