# The File Generator

This is your first actual rule. It introduces the **Analysis Phase** and the concept of `ctx` (the rule context).

## The Task

Write a custom rule called `write_message` that takes a string attribute (`message = "Hello Bazel"`) and an output filename (`out = "hello.txt"`). The rule should use `ctx.actions.write` to create that file.

## What You Learn

- The difference between when a rule is analyzed and when its actions are actually executed.
- How to declare outputs.
- How to use `ctx.actions.write` to generate files during the build.
