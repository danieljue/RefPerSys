# 001test.bash - Test Script for Creating Named Selectors

## Overview

This Bash script (`001test.bash`) is designed to test the creation of a named selector called `prepare_cplusplus_generation` in the RefPerSys persistent store. It automates the process of building the RefPerSys system, loading a specific plugin, and verifying that the selector has been successfully created.

The script serves as a test case to ensure that the plugin system for creating named selectors functions correctly within the RefPerSys framework.

## Prerequisites

Before running this script, ensure the following:

- The `REFPERSYS_TOPDIR` environment variable is set to the root directory of the RefPerSys project.
- The RefPerSys project is properly configured and all dependencies are installed.
- The `refpersys.hh` header file exists in the project root directory.
- GNU Make is available for building the project.
- The `realpath` command is available (typically found in `/usr/bin/realpath`).
- The `grep` command is available for searching files.

## Usage

To run the script:

```bash
export REFPERSYS_TOPDIR=/path/to/refpersys/root
./test_dir/001test.bash
```

The script will execute with verbose output due to the `-x` flag in the shebang.

## Script Behavior

The script performs the following steps:

1. **Environment Check**: Verifies that `REFPERSYS_TOPDIR` is set. If not, exits with error code 1.

2. **Directory Change**: Changes to the RefPerSys top directory.

3. **File Verification**: Checks for the existence of `refpersys.hh`. If not found, exits with error code 1.

4. **Build Process**: Runs `make -j3 refpersys` to build the RefPerSys executable. If the build fails, exits with error code 1.

5. **Persistent Store Path**: Resolves the absolute path of the `persistore` directory.

6. **Pre-existence Check**: Searches for `prepare_cplusplus_generation` in the persistent store. If found, prints a message and exits successfully (code 0).

7. **Plugin Build**: Builds the `rpsplug_createnamedselector.so` plugin.

8. **Plugin Execution**: Runs RefPerSys with the plugin loaded, passing arguments to create the `prepare_cplusplus_generation` selector with specific properties:
   - Comment: "prepare in a C++ generation the module"
   - Rooted: false (0)
   - Constant: true (1)

9. **Verification**: Checks again if `prepare_cplusplus_generation` exists in the persistent store. If not found, exits with error code 1.

10. **Success Confirmation**: Prints a success message and exits with code 0.

## Exit Codes

- `0`: Success - The selector was either already present or successfully created.
- `1`: Failure - Environment not set, required files missing, build failure, or selector creation failed.

## Error Messages

The script outputs error messages to stderr in the following cases:

- Missing `REFPERSYS_TOPDIR`: "without REFPERSYS_TOPDIR"
- Missing `refpersys.hh`: "in <directory> no refpersys.hh header file"
- Selector not found after creation: "no prepare_cplusplus_generation in <persistore_path>"

## Dependencies

This script depends on:

- RefPerSys build system (GNU Make)
- The `rpsplug_createnamedselector` plugin
- RefPerSys executable (`./refpersys`)
- Standard Unix utilities: `realpath`, `grep`, `printf`

## License

This script is licensed under the GNU General Public License version 3 or later (GPL-3.0-or-later).

© Copyright 2025 Basile STARYNKEVITCH
See team@refpersys.org & http://refpersys.org/