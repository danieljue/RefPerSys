# 004crscaltypes.bash - Script to Create Scalar Type Objects

## Overview

This Bash script (`004crscaltypes.bash`) provides a framework for creating multiple RefPerSys objects that reify various C++ scalar (primitive) types. It defines a reusable function `rps_add_scal_type` that can be used to create type objects in the persistent store.

The script is designed as a template for batch-creating scalar type representations within the RefPerSys type system, with examples provided for `char` and `long` types (currently commented out).

## Prerequisites

Before running this script, ensure the following:

- The `REFPERSYS_TOPDIR` environment variable is set to the root directory of the RefPerSys project.
- The RefPerSys project is properly configured and all dependencies are installed.
- The `refpersys.hh` header file exists in the project root directory.
- GNU Make is available for building the project.
- The `realpath` command is available (typically found in `/usr/bin/realpath`).
- The `fgrep` command is available for searching files.

## Usage

To run the script:

```bash
export REFPERSYS_TOPDIR=/path/to/refpersys/root
./test_dir/004crscaltypes.bash
```

The script will execute with verbose output due to the `-x` flag in the shebang.

## Script Behavior

The script performs the following steps:

1. **Environment Check**: Verifies that `REFPERSYS_TOPDIR` is set. If not, exits with error code 1.

2. **Directory Change**: Changes to the RefPerSys top directory.

3. **File Verification**: Checks for the existence of `refpersys.hh`. If not found, exits with error code 1.

4. **Build Process**: Runs `make -j3 refpersys` to build the RefPerSys executable. If the build fails, exits with error code 1.

5. **Plugin Build**: Builds the `rpsplug_create_cplusplus_primitive_type.so` plugin. If the build fails, exits with error code 1.

6. **Persistent Store Path**: Resolves the absolute path of the `persistore` directory.

7. **Function Definition**: Defines the `rps_add_scal_type` function for creating scalar type objects.

## The rps_add_scal_type Function

The script defines a Bash function `rps_add_scal_type` with the following signature:

```bash
rps_add_scal_type <type_name> <comment> [cplusplus_type]
```

**Parameters:**
- `type_name`: The name of the RefPerSys object to create (e.g., `code_char`, `code_long`)
- `comment`: A descriptive comment for the type object
- `cplusplus_type` (optional): The specific C++ type name if different from the object name

**Function Behavior:**
1. Checks if the type already exists in the persistent store JSON files.
2. If it exists, prints a message and skips creation.
3. If it doesn't exist, runs RefPerSys with the plugin to create the object with:
   - Superclass: `cplusplus_primitive_type`
   - Comment: As provided
   - C++ type: Either the provided `cplusplus_type` or defaults to the `type_name`

## Example Usage

The script includes commented-out examples:

```bash
#rps_add_scal_type code_char "reification of C++ type char"
#rps_add_scal_type code_long "reification of C++ type long"
```

To activate these, uncomment the lines. Additional scalar types can be added following the same pattern.

## Exit Codes

- `0`: Success - The script completed without errors.
- `1`: Failure - Environment not set, required files missing, or build failure.

## Error Messages

The script outputs error messages to stderr in the following cases:

- Missing `REFPERSYS_TOPDIR`: "without REFPERSYS_TOPDIR"
- Missing `refpersys.hh`: "in <directory> no refpersys.hh header file"

## Development Notes

- After successfully running lines to create types, the script comments suggest commenting them out after git push to avoid re-running.
- The TODO comment indicates that more similar lines should be added for additional scalar types.
- Common scalar types that might be added include: `code_short`, `code_unsigned_int`, `code_double`, `code_float`, `code_bool`, etc.

## Dependencies

This script depends on:

- RefPerSys build system (GNU Make)
- The `rpsplug_create_cplusplus_primitive_type` plugin
- RefPerSys executable (`./refpersys`)
- Standard Unix utilities: `realpath`, `fgrep`, `printf`

## License

This script is licensed under the GNU General Public License version 3 or later (GPL-3.0-or-later).

© Copyright 2025 Basile STARYNKEVITCH
See team@refpersys.org & http://refpersys.org/