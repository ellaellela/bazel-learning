# The Graph Builder

## Overview

This lesson introduces two of Bazel's most important concepts: **Providers** and **Depsets (Dependency Sets)**. These are the mechanisms that allow rules to communicate with one another and efficiently build large dependency graphs.

## Task

Write a rule called `zip_minified_jsons` that:

* Accepts a list of `minify_json` targets through a `deps` attribute.
* Extracts the generated output files from each dependency using `DefaultInfo`.
* Collects those files into a `depset`.
* Uses an action to package all of the minified JSON files into a single ZIP archive.

## Learning Objectives

By completing this exercise, you'll learn:

* How rules consume information from other rules using providers.
* How to access generated outputs through `DefaultInfo`.
* Why `depset`s are used to efficiently represent collections of files, including transitive dependencies.
* How Bazel traverses the build graph without repeatedly evaluating the same dependencies.
* How information flows between rules during the analysis phase.

These concepts are the foundation of Bazel's scalable build model and are used throughout the ecosystem.

## Why It Matters

Once you complete this lesson, you'll understand the fundamental mechanics behind how major Bazel rule sets—such as `rules_cc` and `rules_python`—are implemented under the hood.