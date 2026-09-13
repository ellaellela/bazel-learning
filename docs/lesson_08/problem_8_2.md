# Bazel Fundamentals - Level 8.2: Toolchain Resolution (Problem Statement)

## The Scenario
Your DevOps team is debugging a cross-compilation issue. They want to know exactly which C++ compiler binary Bazel is selecting behind the scenes when they pass different `--cpu` flags.

Instead of actually compiling C++ files manually (which is complex and better left to Level 8.3), you are going to write a `toolchain_inspector` rule. This rule will query the internal toolchain resolution engine, extract the dynamic path to the C++ compiler, and write it to a text file report.

## DevInfra Concepts: The Toolchain API

1. **Requesting the Toolchain**
   Rules don't get access to toolchains by default. You must explicitly declare that your rule requires the C++ toolchain by adding two parameters to your `rule()` definition:
   ```python
   toolchain_inspector = rule(
       ...
       fragments = ["cpp"],
       toolchains = ["@bazel_tools//tools/cpp:toolchain_type"],
   )
   ```

2. **`find_cpp_toolchain`**
   Inside your implementation function, you use a helper function to grab the toolchain object out of the current Bazel execution context.
   ```python
   cc_toolchain = find_cpp_toolchain(ctx)
   ```

3. **Feature Configuration**
   In Bazel, flags like `-std=c++17` or `-O3` are treated as "features" that can be turned on or off based on the target architecture. Before you can extract the compiler, you must create a `feature_configuration` object that evaluates the current state of the build.
   ```python
   feature_config = cc_common.configure_features(
       ctx = ctx,
       cc_toolchain = cc_toolchain,
       requested_features = ctx.features,
       unsupported_features = ctx.disabled_features,
   )
   ```

4. **Extracting the Tool Path**
   Finally, you ask the feature configuration for the exact path to the tool used for a specific action (like compiling C++ or linking).
   ```python
   compiler_path = cc_common.get_tool_for_action(
       feature_configuration = feature_config,
       action_name = ACTION_NAMES.cpp_compile,
   )
   ```

---

## The Task

### 1. The Rule (`my_rules.bzl`)
Add this to your `.bzl` file. Notice we have to load two new helper modules from Bazel's native C++ rules!

```python
load("@bazel_tools//tools/cpp:toolchain_utils.bzl", "find_cpp_toolchain")
load("@rules_cc//cc:action_names.bzl", "ACTION_NAMES")

def _toolchain_inspector_impl(ctx):
    # 1. Grab the C++ toolchain
    cc_toolchain = find_cpp_toolchain(ctx)
    
    # 2. Configure features based on the current build state
    feature_config = cc_common.configure_features(
        ctx = ctx,
        cc_toolchain = cc_toolchain,
        requested_features = ctx.features,
        unsupported_features = ctx.disabled_features,
    )
    
    # 3. Extract the exact tool path for compiling C++
    # TODO: Use cc_common.get_tool_for_action to get the compiler path.
    # Pass 'feature_configuration = feature_config' and 'action_name = ACTION_NAMES.cpp_compile'
    # Assign the result to a variable called 'compiler_path'
    
    # 4. Write it to a report file
    out_file = ctx.actions.declare_file(ctx.label.name + "_report.txt")
    
    # TODO: Write an action (ctx.actions.write) that saves the compiler_path string into out_file.
    
    return [DefaultInfo(files = depset([out_file]))]

toolchain_inspector = rule(
    implementation = _toolchain_inspector_impl,
    # TODO: Add the 'fragments' and 'toolchains' lists here as shown in the DevInfra Concepts!
)
```

### 2. The Target (`BUILD`)
Add the target to your `BUILD` file:

```python
load("//:my_rules.bzl", "toolchain_inspector")

toolchain_inspector(
    name = "my_compiler_check"
)
```

### 3. Verification (The Cross-Compilation Test)
We are going to build this target twice, pretending we are targeting two completely different hardware architectures. 

Run the first build for your host machine:
```bash
bazel build //:my_compiler_check
cat bazel-bin/my_compiler_check_report.txt
```

Now, run the exact same build, but tell Bazel you are cross-compiling for WebAssembly (or ARM, or a custom internal CPU). Notice how the report automatically changes!
```bash
bazel build //:my_compiler_check --cpu=wasm
cat bazel-bin/my_compiler_check_report.txt
```

**Success Criteria:** The text file should print a path pointing to a compiler (e.g., `/usr/bin/gcc` or an Xcode wrapper path). When you add `--cpu=wasm`, the path should dynamically change, proving that your rule dynamically adapts to cross-compilation without changing a single line of Python code!