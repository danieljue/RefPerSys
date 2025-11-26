# 006echo.bash - RefPerSys Echo Scripting Test

## Overview

This Bash script (`006echo.bash`) serves as a test case for RefPerSys's echo scripting functionality. It demonstrates how RefPerSys can process and echo back script content, including formatted text and mathematical expressions.

The script contains sample content including a famous quote from Pierre Corneille's play "Le Cid" and a mathematical statement about integer arithmetic, serving as test data for the echo functionality.

## Prerequisites

Before running this script, ensure the following:

- The RefPerSys project is properly configured and dependencies are installed.
- GNU Make (`gmake`) is available for building the project (if needed).
- The `realpath` command is available (typically found in `/usr/bin/realpath`).
- The `REFPERSYS_TOPDIR` environment variable may optionally be set to the project root directory.

## Usage

To run the script:

```bash
export REFPERSYS_TOPDIR=/path/to/refpersys/root  # Optional
./test_dir/006echo.bash
```

For debugging with GDB:

```bash
gdb --args ./refpersys -AREPL --script=$(/usr/bin/realpath test_dir/006echo.bash) --batch --run-name 006echodbg
```

For debugging with CGDB:

```bash
cgdb --args ./refpersys -AREPL --script=$REFPERSYS_TOPDIR/test_dir/006echo.bash --batch --run-name 006echocgdb
```

## Script Structure

Similar to `005script.bash`, this script has a dual nature:

1. **Shell Setup Section** (lines 1-34): Bash code that prepares the environment
2. **RefPerSys Script Section** (lines 42+): Content to be echoed by RefPerSys

## Shell Setup Behavior

The script performs the following setup steps:

1. **Directory Change**: If `REFPERSYS_TOPDIR` is set, changes to that directory and prints the current working directory.

2. **Build Check**: Checks if the `refpersys` executable exists and is executable.

3. **Build Process**: If `refpersys` is not executable, runs `gmake -j3 refpersys` to build it.

4. **Executable Verification**: Re-checks if `refpersys` is executable. If not, prints an error and exits with code 1.

5. **RefPerSys Execution**: Runs RefPerSys with the following arguments:
   - `-AREPL`: Enables REPL mode
   - `--script=$(/usr/bin/realpath $0)`: Uses the absolute path of the script as input
   - `--batch`: Runs in batch mode (non-interactive)
   - `--run-name=006echo`: Sets the run name for identification

6. **Exit**: Exits with the same code returned by RefPerSys.

## RefPerSys Script Content

The script contains a magic string that RefPerSys recognizes:

```bash
REFPERSYS_SCRIPT echo
```

This marker tells RefPerSys to use echo mode for processing the following content. The script includes:

1. **Literary Quote**: An excerpt from Pierre Corneille's "Le Cid" (1637), which is in the public domain. This serves as test content with special characters and formatting.

2. **Mathematical Statement**: A formal mathematical expression about integer arithmetic:
   ```
   For every alpha in the set of relative integers alpha plus zero is alpha
   ∀ α ∈ ℤ  α + 0 = α
   ```

## Exit Codes

- `0`: Success - RefPerSys executed the echo script successfully.
- `1`: Failure - The `refpersys` executable could not be found or built.
- Other codes: As returned by the RefPerSys execution.

## Error Messages

The script outputs error messages to stderr in the following cases:

- Missing executable: "no refpersys executable in <current_directory>"

## Development Notes

- The script tests RefPerSys's ability to handle formatted text, special characters (accents), and mathematical notation.
- The magic string `REFPERSYS_SCRIPT echo` is essential and should not be commented out.
- The literary content is included as public domain material for testing purposes.
- The mathematical statement tests Unicode symbol rendering and formal notation processing.
- The use of `realpath` ensures the script path is absolute, avoiding issues with relative paths.

## Dependencies

This script depends on:

- RefPerSys build system (GNU Make/gmake)
- RefPerSys executable (`./refpersys`)
- Standard Unix utilities: `pwd`, `echo`, `realpath`

## License

This script is licensed under the GNU General Public License version 3 or later (GPL-3.0-or-later).

© Copyright (C) 2025 - 2025 The Reflective Persistent System Team
See team@refpersys.org & http://refpersys.org/