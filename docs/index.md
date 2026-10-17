# Bazel Learning

My notes and exercises for learning Bazel.

Each lesson pairs a hands-on **Problem** with a detailed **Explanation**. Use the
sidebar to jump straight to any lesson, or browse the summaries below.

## Lessons

### 1. Wrapper Macros

Learn the fundamentals of Bazel's loading phase by creating `.bzl` files and writing
reusable Starlark macros to dynamically generate `genrule` targets.

[Problem](lesson_01/problem_1_1.md) · [Explanation](lesson_01/explanation_1_1.md)

### 2. First Custom Rule: File Generation

Move beyond macros to build your first custom Bazel rule, exploring the analysis phase,
rule context (`ctx`), output declarations, and generating files with `ctx.actions.write`.

[Problem](lesson_02/problem_2_1.md) · [Explanation](lesson_02/explanation_2_1.md)

### 3. The Shell Transformer

Explore the execution phase and Bazel's hermetic sandboxing by using
`ctx.actions.run_shell` to transform files with external shell utilities while explicitly
declaring inputs and outputs.

[Problem](lesson_03/problem_3_1.md) · [Explanation](lesson_03/explanation_3_1.md)

### 4. Target Chaining & Graph Building

Connect targets into multi-stage build pipelines using `label_list` and `DefaultInfo`,
solve Starlark input-typing traps, and automate rule creation across large file sets with
macros.

- **4.1 The Graph Builder** — Chain build targets together using `attr.label_list` and
  `ctx.files` to consume generated outputs from upstream rules and build multi-step
  dependency graphs.
  [Problem](lesson_04/problem_4_1.md) · [Explanation](lesson_04/explanation_4_1.md)
- **4.2 Bonus: The Header Injector** — Design a clean separation-of-concerns pipeline
  linking upstream rules, while avoiding Starlark type-strictness traps when merging
  single-file and list attributes.
  [Problem & Explanation](lesson_04/problem_4_2.md)
- **4.3 Bonus: The Macro Wrapper** — Bridge the loading and analysis phases by authoring a
  Starlark wrapper macro that loops over source files to stamp out dynamic targets behind a
  clean developer API.
  [Problem & Explanation](lesson_04/problem_4_3.md)

### 5. The Workspace Mutator

Learn how to escape the hermetic build sandbox using the `bazel run` workflow, accessing
`BUILD_WORKSPACE_DIRECTORY` via `py_binary` and runfiles (`data = [...]`) to safely modify
Git-tracked source files in place.

[Problem](lesson_05/problem_5_1.md) · [Explanation](lesson_05/explanation_5_1.md)

### 6. Custom Providers & Transitive Data

Attach typed metadata to the build graph by defining, exporting, and consuming custom
Starlark Providers, then scale across deep dependency trees using `depset`s to collect
transitive data without memory bottlenecks.

- **6.1 The Schema & The Exporter** — Define strongly-typed global `provider()` schemas to
  broadcast non-file metadata (such as ownership and versioning) up the build graph, and
  inspect analysis-phase providers using `bazel cquery`.
  [Problem](lesson_06/problem_6_1.md) · [Explanation](lesson_06/explanation_6_1.md)
- **6.2 The Consumer** — Accept dependency targets in custom rules, safely validate provider
  presence using the `in` operator, and extract upstream metadata to generate downstream
  audit reports.
  [Problem](lesson_06/problem_6_2.md) · [Explanation](lesson_06/explanation_6_2.md)
- **6.3 Transitive Aggregation** — Collect metadata across deep dependency chains using
  nested `depset` pointers to prevent $O(N^2)$ memory copying, flattening them with
  `.to_list()` to produce a full Software Bill of Materials (SBOM).
  [Problem](lesson_06/problem_6_3.md) · [Explanation](lesson_06/explanation_6_3.md)

### 7. Compilers, Args & Configurations

Execute custom compiler and tool binaries natively with `ctx.actions.run`, scale CLI
argument handling and bypass OS limits using `ctx.actions.args()`, and master
cross-compilation transitions using `cfg = "exec"`.

- **7.1 The Custom Compiler** — Replace brittle shell strings with structured,
  platform-independent binary execution using `ctx.actions.run`, binding executable targets
  via `executable = True` to wire implicit tool dependencies into the graph.
  [Problem](lesson_07/problem_7_1.md) · [Explanation](lesson_07/explanation_7_1.md)
- **7.2 The Args Object** — Construct high-performance CLI arguments using
  `ctx.actions.args()` to eliminate analysis-phase memory bloat, and bypass OS character
  limits (`ARG_MAX`) using parameter files (`use_param_file`).
  [Problem](lesson_07/problem_7_2.md) · [Explanation](lesson_07/explanation_7_2.md)
- **7.3 The Execution Transition** — Navigate Bazel's multi-configuration graph and resolve
  cross-compilation constraints by applying `cfg = "exec"` to ensure build-time host tools
  compile for the execution platform rather than the target platform.
  [Problem](lesson_07/problem_7_3.md) · [Explanation](lesson_07/explanation_7_3.md)

### 8. Native C++ Rules & Custom Toolchains

Deep dive into Bazel's native C++ subsystem by emitting `CcInfo` providers, invoking the
`cc_common` compilation and linking APIs, wrapping precompiled vendor libraries, and
authoring custom toolchains from scratch.

- **8.1 The CcInfo Provider** — Bridge custom rules with Bazel's native C++ ecosystem by
  generating header files and packaging them into `CcInfo` using `compilation_context`
  (headers, include directories, and depsets).
  [Problem](lesson_08/problem_8_1.md) · [Explanation](lesson_08/explanation_8_1.md)
- **8.2 Toolchain Resolution** — Declare toolchain dependencies via `toolchain_type` and
  resolve Bazel's platform-agnostic C++ toolchain dynamically in `ctx.toolchains` to
  retrieve compiler paths and build features.
  [Problem](lesson_08/problem_8_2.md) · [Explanation](lesson_08/explanation_8_2.md)
- **8.3 The cc_common API** — Replace manual shell commands by programmatically invoking
  `cc_common.compile()` and constructing linking contexts to compile C++ source files
  natively through Bazel's internal engine.
  [Problem](lesson_08/problem_8_3.md) · [Explanation](lesson_08/explanation_8_3.md)
- **8.4 Wrapping Precompiled Libraries** — Package prebuilt third-party static and shared
  libraries (`.a` / `.so`) into first-class Bazel dependencies using
  `cc_common.create_linking_context` and `create_library_to_link`.
  [Problem](lesson_08/problem_8_4.md) · [Explanation](lesson_08/explanation_8_4.md)
- **8.5 Custom Toolchain Configuration** — Build and register a custom C++ cross-compiler
  toolchain from the ground up using `cc_common.create_cc_toolchain_config_info` for
  specialized environments and embedded hardware targets.
  [Problem](lesson_08/problem_8_5.md) · [Explanation](lesson_08/explanation_8_5.md)

### 9. Aspects & Graph Traversal

Master Bazel Aspects to inspect, transform, and aggregate metadata across entire dependency
graphs without modifying existing rules or BUILD files—scaling from shadow file generation
and native graph pruning to polyglot traversal and aspect composition.

- **9.1 The Observer** — Introduce Bazel Aspects to traverse build graphs non-invasively,
  configuring `attr_aspects = ["deps"]` to inspect target labels across dependencies from
  the command line using `--aspects`.
  [Problem](lesson_09/problem_9_1.md) · [Explanation](lesson_09/explanation_9_1.md)
- **9.2 The Generator** — Register shadow build actions within an aspect using
  `ctx.actions` and expose generated artifacts to the CLI via `OutputGroupInfo` and the
  `--output_groups` flag.
  [Problem](lesson_09/problem_9_2.md) · [Explanation](lesson_09/explanation_9_2.md)
- **9.3 The Aggregator** — Bubble analysis metadata and shadow artifacts up the dependency
  tree by defining custom aspect providers, accessing child aspect data via
  `ctx.rule.attr.deps`, and merging transitive `depset`s.
  [Problem](lesson_09/problem_9_3.md) · [Explanation](lesson_09/explanation_9_3.md)
- **9.4 IDE Integration** — Attach aspects directly to rule attributes using
  `attr.label(aspects = [...])` to programmatically consume aspect outputs inside custom
  rules for IDE project generation and build orchestration.
  [Problem](lesson_09/problem_9_4.md) · [Explanation](lesson_09/explanation_9_4.md)
- **9.5 Native Graph Pruning** — Optimize analysis-phase performance on large graphs by
  applying `required_providers` to prune non-matching subgraphs in Bazel's native engine
  before Starlark code is ever invoked.
  [Problem](lesson_09/problem_9_5.md) · [Explanation](lesson_09/explanation_9_5.md)
- **9.6 Parameterized Aspects** — Configure and control aspect behavior dynamically across
  the build graph by declaring custom aspect attributes and command-line flags.
  [Problem](lesson_09/problem_9_6.md) · [Explanation](lesson_09/explanation_9_6.md)
- **9.7 The Polyglot Traverse** — Propagate aspects across heterogeneous multi-language
  boundaries by traversing multiple attribute types (`deps`, `data`, `runtime_deps`) and
  branching on distinct language providers.
  [Problem](lesson_09/problem_9_7.md) · [Explanation](lesson_09/explanation_9_7.md)
- **9.8 External Toolchain Injection** — Bind external linters, security scanners, and host
  binaries to aspects using toolchains and private attributes to run hermetic analysis
  actions across every visited target.
  [Problem](lesson_09/problem_9_8.md) · [Explanation](lesson_09/explanation_9_8.md)
- **9.9 Aspect Composition** — Compose multiple independent aspects into a coherent pipeline
  using the `requires` attribute, allowing downstream consumer aspects to safely depend on
  upstream parser providers.
  [Problem](lesson_09/problem_9_9.md) · [Explanation](lesson_09/explanation_9_9.md)

