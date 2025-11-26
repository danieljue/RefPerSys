# 002crintptr.bash - Script to Create intptr_t Type Object

## Overview

This Bash script (`002crintptr.bash`) creates a RefPerSys object that reifies the C++ `intptr_t` primitive type. It automates the process of building the necessary plugin and using it to create the `code_intptr_t` object in the persistent store.

The script is part of the RefPerSys type system initialization, ensuring that fundamental C++ types are represented as objects within the system.

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
./test_dir/002crintptr.bash
```

The script will execute with verbose output due to the `-x` flag in the shebang.

## Script Behavior

The script performs the following steps:

1. **Environment Check**: Verifies that `REFPERSYS_TOPDIR` is set. If not, exits with error code 1.

2. **Directory Change**: Changes to the RefPerSys top directory.

3. **File Verification**: Checks for the existence of `refpersys.hh`. If not found, exits with error code 1.

4. **Build Process**: Runs `make -j3 refpersys` to build the RefPerSys executable. If the build fails, exits with error code 1.

5. **Persistent Store Path**: Resolves the absolute path of the `persistore` directory.

6. **Pre-existence Check**: Searches for `code_intptr_t` in the persistent store. If found, prints a message and exits successfully (code 0).

7. **Plugin Build**: Builds the `rpsplug_create_cplusplus_primitive_type.so` plugin. If the build fails, exits with error code 1.

8. **Plugin Execution**: Runs RefPerSys with the plugin loaded, creating the `code_intptr_t` object with the following properties:
   - Superclass: `cplusplus_primitive_type`
   - Comment: "the native intptr_t type"

9. **Verification**: Checks if `code_intptr_t` exists in the persistent store. If not found, exits with error code 1.

10. **Success Confirmation**: Prints a success message and a reminder to edit `rps_set_native_data_in_loader` in `load_rps.cc`.

## Exit Codes

- `0`: Success - The `code_intptr_t` object was either already present or successfully created.
- `1`: Failure - Environment not set, required files missing, build failure, or object creation failed.

## Error Messages

The script outputs error messages to stderr in the following cases:

- Missing `REFPERSYS_TOPDIR`: "without REFPERSYS_TOPDIR"
- Missing `refpersys.hh`: "in <directory> no refpersys.hh header file"
- `code_intptr_t` not found after creation: "no code_intptr_t in <persistore_path>"

## Post-Execution Notes

After successful execution, the script reminds the user to manually edit the `rps_set_native_data_in_loader` function in `load_rps.cc` to properly integrate the newly created `code_intptr_t` object into the system's native data loading mechanism.

## Dependencies

This script depends on:

- RefPerSys build system (GNU Make)
- The `rpsplug_create_cplusplus_primitive_type` plugin
- RefPerSys executable (`./refpersys`)
- Standard Unix utilities: `realpath`, `grep`, `printf`

## License

This script is licensed under the GNU General Public License version 3 or later (GPL-3.0-or-later).

© Copyright 2025 Basile STARYNKEVITCH
See team@refpersys.org & http://refpersys.org/