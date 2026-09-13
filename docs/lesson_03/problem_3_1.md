# The Shell Transformer

## Overview

In this lesson, you'll be introduced to the **Execution Phase** and learn how Bazel actions declare their inputs and outputs.

## Task

Write a rule called `minify_json` that:

* Takes an input file, for example:

  * `src = "data.json"`
* Produces an output file, for example:

  * `out = "minified.json"`
* Uses `ctx.actions.run_shell` to execute a simple shell command that transforms the input into the output, such as:

```bash
jq -c . < input > output
```

## Learning Objectives

By completing this exercise, you'll learn:

* How `ctx.actions.run_shell` executes commands during Bazel's execution phase.
* How Bazel creates isolated execution sandboxes for actions.
* Why every file an action reads must be explicitly declared as an input.
* Why every file an action produces must be explicitly declared as an output.

If you forget to pass the input file to the action, the shell command will fail with a **"file not found"** error because Bazel will not mount the file into the execution sandbox.
