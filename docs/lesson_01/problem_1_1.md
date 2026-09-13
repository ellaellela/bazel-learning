# The Wrapper Macro

A macro is not a true rule—it is just a Python-like function that wraps existing rules. This is your introduction to the **Loading Phase**.

## The Task

Write a Starlark macro called `generate_multiple_texts` that takes a list of strings and dynamically generates a standard `genrule` for each one.

## What You Learn

- How to create a `.bzl` file.
- How to load a `.bzl` file into a `BUILD` file.
- How Bazel executes Starlark code to evaluate the build graph before any actual compilation happens.
