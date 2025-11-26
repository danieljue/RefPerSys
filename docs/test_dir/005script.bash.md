# 005script.bash - RefPerSys Scripting Test

## Overview

This Bash script (`005script.bash`) serves as a test case for RefPerSys's scripting functionality. It demonstrates how to execute RefPerSys scripts and provides a template for creating scripted interactions with the RefPerSys system.

The script is designed to be executed by the RefPerSys interpreter itself, containing both shell setup code and RefPerSys script content.

## Prerequisites

Before running this script, ensure the following:

- The RefPerSys project is properly configured and dependencies are installed.
- GNU Make (`gmake`) is available for building the project (if needed).
- The `REFPERSYS_TOPDIR` environment variable may optionally be set to the project root directory.

## Usage

To run the script:

```bash
export REFPERSYS_TOPDIR=/path/to/refpersys/root  # Optional
./test_dir/005script.bash
```

For debugging with GDB:

```bash
gdb --args ./refpersys -AREPL --script=test_dir/005script.bash --batch --run-name 005script
```

## Script Structure

The script has a dual nature:

1. **Shell Setup Section** (lines 1-33): Bash code that prepares the environment
2. **RefPerSys Script Section** (lines 39+): Content executed by RefPerSys

## Shell Setup Behavior

The script performs the following setup steps:

1. **Directory Change**: If `REFPERSYS_TOPDIR` is set, changes to that directory and prints the current working directory.

2. **Build Check**: Checks if the `refpersys` executable exists and is executable.

3. **Build Process**: If `refpersys` is not executable, runs `gmake -j3 refpersys` to build it.

4. **Executable Verification**: Re-checks if `refpersys` is executable. If not, prints an error and exits with code 1.

5. **RefPerSys Execution**: Runs RefPerSys with the following arguments:
   - `-AREPL`: Enables REPL mode
   - `--script=$0`: Uses the script itself as input
   - `--batch`: Runs in batch mode (non-interactive)
   - `--run-name=005script`: Sets the run name for identification

6. **Exit**: Exits with the same code returned by RefPerSys.

## RefPerSys Script Content

The script contains a magic string that RefPerSys recognizes:

```bash
REFPERSYS_SCRIPT carbon
```

This marker tells RefPerSys that the following content is script code to be interpreted. The "carbon" likely refers to a scripting language or mode within RefPerSys.

## Exit Codes

- `0`: Success - RefPerSys executed the script successfully.
- `1`: Failure - The `refpersys` executable could not be found or built.
- Other codes: As returned by the RefPerSys execution.

## Error Messages

The script outputs error messages to stderr in the following cases:

- Missing executable: "no refpersys executable in <current_directory>"

## Development Notes

- The script serves as a template for creating RefPerSys scripts.
- The magic string `REFPERSYS_SCRIPT carbon` is essential and should not be commented out.
- Additional script content can be added after the magic string.
- The GDB command provided allows for debugging the script execution within RefPerSys.

## Dependencies

This script depends on:

- RefPerSys build system (GNU Make/gmake)
- RefPerSys executable (`./refpersys`)
- Standard Unix utilities: `pwd`, `echo`

## License

This script is licensed under the GNU General Public License version 3 or later (GPL-3.0-or-later).

© Copyright (C) 2025 - 2025 The Reflective Persistent System Team
See team@refpersys.org & http://refpersys.org/