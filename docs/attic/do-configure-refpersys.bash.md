# Interactive Build Configuration Script (`attic/do-configure-refpersys.bash`)

**File Path:** `attic/do-configure-refpersys.bash`

## Overview

This interactive Bash script provides an advanced build configuration system for RefPerSys, allowing users to specify and validate C and C++ compilers through hands-on testing. Unlike the simpler `configure` script, this script interactively prompts users for compiler paths and performs comprehensive testing with sample programs to ensure compiler compatibility and functionality.

## Script Purpose

### Interactive Compiler Configuration
The script enables precise build setup by:

- **Interactive Compiler Selection:** User-guided compiler path specification
- **Comprehensive Testing:** Multi-stage compiler validation with sample programs
- **Dependency Verification:** Library linkage and runtime testing
- **Build Environment Setup:** Creation of validated compiler configurations
- **Error Detection:** Early identification of compilation and linking issues

## Implementation Status

### Current State
```bash
### TODO: not working so well in commit 8f0f3c2715a8 (Jan 29, 2024)
```

**Status:** Partially implemented, C++ compiler configuration incomplete

### Design Intent
- **Interactive Configuration:** User-driven compiler selection and testing
- **Comprehensive Validation:** Multi-level compiler capability testing
- **Build Verification:** End-to-end compilation and execution testing
- **Dependency Checking:** Library availability and linkage verification

## Script Architecture

### Global Variables and Setup
```bash
script_name=$0
declare -a files_to_remove
shopt -s extdebug
PS4='dolzero=$0 lineno=$LINENO bash_source=$BASH_SOURCE bash_lineno=$BASH_LINENO '
```

**Debugging Support:**
- **Script Name Tracking:** Self-identification for error messages
- **File Cleanup:** Array for temporary file management
- **Extended Debugging:** Enhanced error reporting with line numbers
- **Execution Tracing:** Detailed execution path logging

## C Compiler Testing

### Single-File Compilation Test
```bash
function try_c_compiler() { # $1 is the C compiler to try
    # Create temporary C source file
    csrc=$(/usr/bin/mktemp tmp_helloworld.XXXXX.c)
    testexe=$(/usr/bin/mktemp --dry-run tmp_helloworld.XXXXX.bin)
    
    # Generate hello world program
    printf '/// sample helloworld C program %s from %s\n' $csrc $script_name >> $csrc
    printf '#include <stdio.h>\n' >> $csrc
    printf 'int main(int argc, char**argv) {\n' >> $csrc
    printf '  printf("hello from %%s HERE %%s in %s\\n", argv[0], HERE, __FILE__);\n' >> $csrc
    printf '  return 0;\n' >> $csrc
    printf '}\n /// eof %s\n' $csrc >> $csrc
```

**Test Program Features:**
- **Dynamic Content:** Includes script name and source file path
- **Printf Validation:** Tests formatted output functionality
- **File Reference:** Verifies `__FILE__` macro expansion
- **Argument Processing:** Validates command-line argument access

### Compilation and Execution
```bash
$1 -DHERE=\"$script_name\" -O -g -o $testexe $csrc
if [ $? -ne 0 ]; then
    printf '%s: failed to C compile %s into %s with %s\n' $script_name $csrc $testexe $1 > /dev/stderr
    exit 1
fi

# Library dependency analysis
$script_name running ldd ./$testexe
/usr/bin/ldd ./$testexe

# Output validation
if ! ./$testexe | /bin/grep 'hello from' ; then
    echo $script_name failed ./$testexe "(no hello output)" > /dev/stderr
    exit 1
fi
```

**Validation Steps:**
- **Compilation Success:** Verifies compiler can generate executable
- **Library Dependencies:** Analyzes linked libraries with `ldd`
- **Runtime Execution:** Tests program runs without errors
- **Output Verification:** Validates expected output content

### Multi-File Compilation Test
```bash
# Create separate source files
csrc=$(/usr/bin/mktemp tmp_otherfirsthelloworld.XXXXX.c)
othercsrc=$(/usr/bin/mktemp tmp_othersecondhelloworld.XXXXX.c)

# First file: function definition
printf 'void sayhello(const char*c) {\n' >> $csrc
printf '     printf("hello from %%s HERE %%s in %s\\n", c, HERE, __FILE__);\n' >> $csrc
printf '}\n' >> $csrc

# Second file: main function
printf 'extern void sayhello(const char*c);\n' >> $othercsrc
printf 'int main(int argc,char**argv) {\n' >> $othercsrc
printf '   if (argc>0) sayhello(argv[0]);\n' >> $othercsrc
printf '   return 0;\n' >> $othercsrc
printf '}\n' >> $othercsrc

# Separate compilation
$1 -DHERE=\"$script_name\" -O -g -Wall -c $csrc
$1 -DHERE=\"$script_name\" -O -g -Wall -c $othercsrc

# Linking
$1 -O -g $firstobj $otherobj -o $otherexe
```

**Multi-File Testing:**
- **Separate Compilation:** Tests object file generation
- **Function Prototypes:** Validates extern declarations
- **Linking Process:** Verifies multi-object linking
- **Cross-File References:** Tests inter-module function calls

## Interactive User Interface

### C Compiler Selection
```bash
function ask_c_compiler() {
    local c
    echo $script_name asking for a C compiler
    echo Enter path of C99 compiler preferably GCC from gcc.gnu.org
    read -e -i /usr/bin/gcc -p "C compiler: " c
    echo Using $c as the C99 compiler
    $c --version
    try_c_compiler $c
}
```

**User Interaction:**
- **Prompt Guidance:** Clear instructions for compiler selection
- **Default Suggestion:** Pre-fills with common GCC location
- **Version Display:** Shows selected compiler version
- **Comprehensive Testing:** Full validation of selected compiler

## C++ Compiler Framework

### Incomplete Implementation
```bash
function try_cplusplus_compiler { # $1 is the C++ compiler to try
    # Basic C++ hello world test (similar to C version)
    cpsrc=$(/usr/bin/mktemp tmp_helloworldcxx.XXXXX.cc)
    # ... C++ code generation ...
    $1 -DHERE=\"$script_name\" -O -g -o $testexe $cpsrc
    # ... validation ...
}

function ask_cpp_compiler() {
    local cpp
    # TODO: Implement interactive C++ compiler selection
}
```

**Planned Features:**
- **C++ Standard Library:** Testing STL functionality
- **Exception Handling:** C++ exception mechanism validation
- **Template Support:** Basic template instantiation testing
- **RAII Compliance:** Resource management verification

## Resource Management

### Temporary File Cleanup
```bash
files_to_remove+=($csrc $testexe)
# ... at script end ...
echo $script_name should remove ${#files_to_remove[@]} files from ${files_to_remove}
for f in ${files_to_remove} ; do
    /bin/rm -vf $f
done
```

**Cleanup Strategy:**
- **File Tracking:** Array accumulates temporary files
- **Batch Removal:** Single cleanup pass at script end
- **Verbose Deletion:** Shows files being removed
- **Error Resilience:** Continues despite individual file removal failures

## Error Handling

### Compilation Failures
```bash
if [ $? -ne 0 ]; then
    printf '%s: failed to C compile %s into %s with %s\n' $script_name $csrc $testexe $1 > /dev/stderr
    exit 1
fi
```

**Error Responses:**
- **Immediate Termination:** Stops on compilation failures
- **Detailed Diagnostics:** Shows which file, compiler, and operation failed
- **User Guidance:** Clear error messages for troubleshooting

### Runtime Validation
```bash
if ! ./$testexe | /bin/grep 'hello from' ; then
    echo $script_name failed ./$testexe "(no hello output)" > /dev/stderr
    exit 1
fi
```

**Execution Testing:**
- **Output Analysis:** Verifies program produces expected results
- **Pattern Matching:** Uses grep for flexible output validation
- **Failure Isolation:** Identifies specific test failures

## System Integration

### Compiler Discovery
- **Path Resolution:** Supports absolute and relative compiler paths
- **Standard Locations:** Defaults to common installation directories
- **Custom Installations:** Allows user-specified compiler locations
- **Cross-Compilation:** Framework for different target architectures

### Build System Integration
- **Makefile Variables:** Generates compiler settings for make
- **Environment Setup:** Creates build environment configuration
- **Dependency Tracking:** Records tested compiler capabilities
- **Reproducibility:** Ensures consistent build environments

## Performance Considerations

### Testing Overhead
- **File I/O:** Temporary file creation and cleanup
- **Compilation Time:** Multiple compilation cycles for testing
- **Execution Time:** Runtime testing of generated executables
- **Library Analysis:** `ldd` dependency scanning

### Resource Usage
- **Disk Space:** Temporary files for testing
- **Memory:** Compiler processes and test execution
- **Time:** Comprehensive validation takes time but prevents build failures

## Limitations and Issues

### Current Limitations
1. **Incomplete C++ Support:** C++ compiler testing not implemented
2. **Single Compiler Focus:** Tests one compiler at a time
3. **Limited Library Testing:** Basic functionality only
4. **No Optimization Testing:** Focuses on correctness, not performance

### Known Issues
1. **Script Instability:** Marked as "not working so well"
2. **Hardcoded Paths:** Assumes `/usr/bin` locations
3. **Limited Error Recovery:** Fails fast on any error
4. **No Configuration Output:** Doesn't generate build files

## Comparison with Other Scripts

### vs. `configure` Script
- **Interactivity:** Interactive vs. automated
- **Testing Depth:** Comprehensive validation vs. basic checks
- **User Control:** Manual selection vs. predefined choices
- **Error Handling:** Detailed feedback vs. simple failures

### vs. Modern Build Systems
- **Approach:** Traditional script vs. declarative configuration
- **Maintainability:** Shell script vs. specialized tools
- **Extensibility:** Custom logic vs. plugin ecosystems
- **Integration:** Direct testing vs. cached results

## File Status

**Status:** Partially Implemented Framework
**Date:** 2024
**Purpose:** Interactive compiler validation and build setup

## Summary

The `do-configure-refpersys.bash` script represents an ambitious attempt to create an interactive, thoroughly-tested build configuration system for RefPerSys. While incomplete (particularly the C++ compiler testing), it demonstrates sophisticated shell scripting techniques for compiler validation, including multi-stage testing, comprehensive error handling, and user interaction. The script's approach of generating and testing actual programs provides confidence in compiler capability, though its complexity and incomplete state suggest it was supplanted by simpler configuration approaches. The framework shows good design principles for build system validation, with clear separation of concerns, robust error handling, and comprehensive testing methodologies that could inform future build system development.