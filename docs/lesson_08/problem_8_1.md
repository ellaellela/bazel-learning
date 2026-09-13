# Bazel Fundamentals - Level 8.1: The CcInfo Provider (Problem Statement)

## The Scenario
Your automotive team has a custom tool that dynamically generates a C++ header file (`hw_version.h`) containing the firmware version for a specific ECU (Engine Control Unit). 

You need a standard Bazel `cc_binary` to depend on your custom rule, so the C++ developers can simply write `#include "hw_version.h"` in their `main.cpp` and compile successfully.

**The Trap:** 
If your custom rule just returns `DefaultInfo`, the `cc_binary` will fail to compile. It will complain that `"hw_version.h"` is not found. Why? Because the C++ compiler needs to know exactly which directories to pass to the `-I` (include search path) flag, and `DefaultInfo` doesn't provide that.

## DevInfra Concepts: `CcInfo`

To talk to Bazel's native C++ rules, your rule must speak their language. That language is a built-in provider called `CcInfo`.

1. **The `CcInfo` Provider**
   Just like you created `TeamMetadataInfo`, Bazel has a globally defined `CcInfo` provider. Any rule that returns `CcInfo` can be added to the `deps` of a `cc_library` or `cc_binary`.

2. **The `compilation_context`**
   C++ information is split into two halves: compile-time (headers, `-I` flags, macros) and link-time (object files, static/shared libraries). 
   To expose a header file, you only need to populate the compile-time half, called the `compilation_context`. 

3. **`cc_common.create_compilation_context`**
   You don't construct the context manually. You use Bazel's native API, passing it `depset`s of your headers and the directories those headers live in.
   ```python
   comp_ctx = cc_common.create_compilation_context(
       headers = depset([my_header_file]),
       includes = depset([my_header_file.dirname]), # The directory for the -I flag
   )
   ```

---

## The Task

You will write a custom rule `hardware_version_header` that generates a `.h` file and returns it inside a `CcInfo` provider. Then, you will depend on it from a standard `cc_binary`.

### 1. The C++ Source (`main.cpp`)
Create a simple `main.cpp` file in your workspace. Notice it includes our dynamically generated header!
```cpp
#include <iostream>
#include "hw_version.h"

int main() {
    std::cout << "Booting Engine Control Unit..." << std::endl;
    std::cout << "Hardware Version: " << HW_VERSION << std::endl;
    return 0;
}
```

### 2. The Rule (`my_rules.bzl`)
Add this to your `.bzl` file and complete the `TODO`s.

```python
def _hardware_version_header_impl(ctx):
    # 1. Generate the header file
    out_file = ctx.actions.declare_file("hw_version.h")
    ctx.actions.write(
        output = out_file,
        content = "#define HW_VERSION " + str(ctx.attr.version) + "\n",
    )
    
    # 2. Create the compilation context
    # TODO: Use cc_common.create_compilation_context
    # Pass out_file in a depset to 'headers'
    # Pass out_file.dirname in a depset to 'includes'
    
    # 3. Wrap it in a CcInfo provider
    # TODO: Create a CcInfo object passing your compilation context to 'compilation_context'
    
    # 4. Return the providers
    # TODO: Return a list containing BOTH DefaultInfo(files = depset([out_file])) AND your new CcInfo
    pass

hardware_version_header = rule(
    implementation = _hardware_version_header_impl,
    attrs = {
        "version": attr.int(mandatory = True),
    },
)
```

### 3. The Targets (`BUILD`)
Add the targets to your `BUILD` file. Watch how the native `cc_binary` seamlessly depends on your custom python-based rule!

```python
load("//:my_rules.bzl", "hardware_version_header")

# Your custom rule generating the header
hardware_version_header(
    name = "ecu_version",
    version = 42,
)

# Standard native C++ binary
cc_binary(
    name = "firmware_app",
    srcs = ["main.cpp"],
    deps = [":ecu_version"], # It accepts this because it returns CcInfo!
)
```

### 4. Verification
Run the C++ binary!

```bash
bazel run //:firmware_app
```

**Success Criteria:** Bazel will generate the header via your custom rule, then compile the `cc_binary`, link it, and execute it. Your terminal should print out `Hardware Version: 42`.