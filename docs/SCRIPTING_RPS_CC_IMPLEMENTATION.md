# Scripting Implementation Analysis (`scripting_rps.cc`)

## Overview

The `scripting_rps.cc` file implements post-load scripting support for the Reflective Persistent System (RefPerSys), enabling execution of script files after the system heap has been loaded. This module provides infrastructure for running initialization scripts, batch operations, and automated system configuration through textual script files with embedded RefPerSys commands.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch
- **Copyright**: © 2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// GMP dependencies for arbitrary precision arithmetic
//@@PKGCONFIG gmp
//@@PKGCONFIG gmpxx

// Git versioning information
extern "C" const char rps_scripting_gitid[];
extern "C" const char rps_scripting_date[];
extern "C" const char rps_scripting_shortgitid[];
extern "C" const char rps_scripting_timestamp[];

// External function declarations
extern "C" void rps_scripting_help(void);
extern "C" void rps_scripting_add_script(const char*);

// Magic string for script detection
extern "C" const char rps_scripting_magic_string[];
```

### Key Dependencies
- **refpersys.hh**: Core system definitions and Rps_CallFrame
- **GMP/GMPXX**: Arbitrary precision arithmetic libraries
- **POSIX functions**: `realpath`, `access`, `strerror` for file operations

## Script File Management

### Global State
```cpp
// Maximum number of script files
extern "C" const int rps_script_maxnum = 1024;

// Vector storing real paths to script files
static std::vector<const char*> rps_scripts_vector;
```

**Purpose**: Maintains a registry of script files to be executed after system loading.

### Script Addition Function
```cpp
void rps_scripting_add_script(const char* path)
{
  // Validate file accessibility
  if (access(path, R_OK))
    RPS_FATALOUT("script file not accessible");

  // Get canonical path
  char* rp = realpath(path, nullptr);
  if (!rp) RPS_FATALOUT("realpath failed");

  // Thread safety check
  if (!rps_is_main_thread())
    RPS_FATALOUT("adding script from non-main thread");

  // Size limit check
  if ((int)rps_scripts_vector.size() > rps_script_maxnum)
    RPS_FATALOUT("too many script files");

  // Add to registry
  rps_scripts_vector.push_back(rp);
}
```

**Validation Process**:
1. **Accessibility**: Check read permission with `access()`
2. **Canonicalization**: Convert to absolute path with `realpath()`
3. **Thread Safety**: Ensure main thread execution
4. **Size Limits**: Prevent excessive script loading

## Script Execution Infrastructure

### Post-Load Script Execution
```cpp
void rps_run_scripts_after_load(Rps_CallFrame* caller)
{
  if (rps_scripts_vector.empty()) return;

  // Execute each script in sequence
  for (int ix = 0; ix < (int)rps_scripts_vector.size(); ix++) {
    try {
      rps_run_one_script_file(&_, ix);
    } catch (std::exception& ex) {
      RPS_FATALOUT("failed to run script#" << ix << ": " << ex.what());
    }
  }
}
```

**Execution Model**:
- **Sequential Processing**: Scripts run in registration order
- **Error Handling**: Exceptions cause immediate termination
- **Call Frame Context**: Each script executes within its own call frame
- **Debug Logging**: Comprehensive tracing of script execution

### Individual Script Processing
```cpp
void rps_run_one_script_file(Rps_CallFrame* callframe, int ix)
{
  // Create token source from script file
  Rps_MemoryFileTokenSource tsrc(rps_scripts_vector[ix]);

  // Process script content
  // Implementation incomplete - marked with warnings
}
```

**Purpose**: Process individual script files with proper tokenization and parsing.

## Magic String Detection

### Script Format Specification
```cpp
const char rps_scripting_magic_string[] = "REFPERSYS_SCRIPT";
#define RPS_SCRIPT_MAGIC_STR "REFPERSYS_SCRIPT"
```

### Script File Format
```
# Script files can contain arbitrary content before the magic string
# Lines before REFPERSYS_SCRIPT are ignored (can be shell scripts, etc.)

REFPERSYS_SCRIPT [mode] [parameters]
# RefPerSys script commands follow...
```

### Magic String Detection Algorithm
```cpp
// Scan file line by line until magic string found
while (!gotmagic && !tsrc.reached_end()) {
  if (!tsrc.get_line()) continue;

  const char* clp = tsrc.curcptr();
  const char* magp = strstr(clp, rps_scripting_magic_string);

  if (magp) {
    // Parse mode and parameters
    int n = sscanf(magp, "REFPERSYS_SCRIPT %60[A-Za-z0-9_]%n", modline, &p);
    if (n > 0 && isascii(modline[0]) && p > 0) {
      gotmagic = true;
      // Process based on mode (carbon, echo, etc.)
    }
  }
}
```

**Detection Process**:
1. **Line-by-Line Scanning**: Read file until magic string found
2. **Pattern Matching**: Use `strstr()` to locate magic string
3. **Mode Parsing**: Extract mode identifier after magic string
4. **Validation**: Ensure valid mode name format

## Script Modes

### Carbon Mode
```cpp
if (!strcmp(modline, "carbon")) {
  // Use Carburetta-generated parser
  // rps_do_carburetta_command integration planned
  RPS_WARNOUT("unimplemented carbon mode");
}
```

**Purpose**: Execute scripts using the Carburetta parser generator output.

### Echo Mode
```cpp
if (!strcmp(modline, "echo")) {
  // Output script content to stdout
  while (tsrc.get_line() && !tsrc.reached_end()) {
    const char* clp = tsrc.curcptr();
    if (!clp) break;
    std::cout << clp << std::flush;
  }
  std::cout << std::endl;
}
```

**Purpose**: Simple script content echoing for debugging and logging.

### Future Modes
- **Command Mode**: Execute RefPerSys REPL commands
- **Configuration Mode**: System configuration and setup
- **Batch Mode**: Multiple command execution
- **Plugin Mode**: Load and initialize plugins

## Integration with REPL System

### Script Command Processing
```cpp
// Scripts can contain REPL-style commands
// Integration with rps_do_one_repl_command planned
// Support for both !builtin and regular commands
```

**Planned Integration**:
- **Command Dispatch**: Route script commands through REPL infrastructure
- **Environment Sharing**: Scripts can access REPL environment
- **Error Propagation**: Script errors handled like REPL errors
- **Debug Support**: Full REPL debugging available in scripts

### Token Source Usage
```cpp
// Use Rps_MemoryFileTokenSource for script parsing
Rps_MemoryFileTokenSource tsrc(curpstr);

// Provides same interface as other token sources
// Supports line-by-line processing and position tracking
```

**Benefits**:
- **Consistent Parsing**: Same tokenizer as REPL
- **Position Tracking**: Error reporting with file/line information
- **Memory Efficiency**: File content loaded once
- **Debug Support**: Full token stream inspection

## Help and Documentation

### Help System
```cpp
void rps_scripting_help(void)
{
  // Display scripting help information
  RPS_FATALOUT("unimplemented rps_scripting_help");
}
```

### Help Text
```cpp
const char rps_scripting_help_english_text[] = R"help(
A script file is a textual file.
All its initial lines before a line containing REFPERSYS_SCRIPT are ignored.
Hence these initial lines could contain some shell script, etc.
)help";
```

**Purpose**: Provide user guidance for script file format and usage.

## Usage Patterns and Examples

### Basic Script File
```bash
#!/bin/bash
# This is a shell script that also contains RefPerSys commands
echo "Setting up RefPerSys environment..."

# RefPerSys script commands start here
REFPERSYS_SCRIPT echo
Hello from RefPerSys script!
This text will be echoed to stdout.
```

### Carbon Mode Script
```bash
# Use Carburetta parser for complex commands
REFPERSYS_SCRIPT carbon
# Complex RefPerSys commands would go here
# (Currently unimplemented)
```

### Echo Mode Script
```bash
# Simple output script
REFPERSYS_SCRIPT echo
System initialization complete.
Configuration loaded successfully.
Ready for interactive use.
```

### Script Registration
```cpp
// Add scripts during system initialization
rps_scripting_add_script("/etc/refpersys/init.rps");
rps_scripting_add_script("./user-config.rps");
rps_scripting_add_script("plugins/setup.rps");

// Execute all registered scripts after loading
rps_run_scripts_after_load(caller_frame);
```

### Error Handling
```cpp
try {
  rps_scripting_add_script("nonexistent.rps");
} catch (std::runtime_error& e) {
  // Handle script loading errors
  std::cerr << "Script loading failed: " << e.what() << std::endl;
}
```

## Implementation Status and Limitations

### Incomplete Features
- **Script Execution**: `rps_run_one_script_file()` is largely unimplemented
- **Carbon Mode**: Integration with Carburetta parser not implemented
- **Command Mode**: REPL command execution in scripts not implemented
- **Help System**: `rps_scripting_help()` not implemented
- **Error Recovery**: Limited error handling and recovery

### Current Capabilities
- **Script Registration**: Files can be added to execution queue
- **File Validation**: Basic accessibility and path validation
- **Magic String Detection**: Can identify script start markers
- **Echo Mode**: Basic text output functionality
- **Sequential Execution**: Scripts run in registration order

### Thread Safety
- **Main Thread Only**: Script operations restricted to main thread
- **No Concurrency**: Sequential script execution prevents race conditions
- **Memory Safety**: Proper resource cleanup with `realpath()` allocation tracking

### Performance Considerations
- **File Loading**: Scripts loaded into memory token sources
- **Sequential Processing**: No parallel script execution
- **Memory Usage**: Script paths stored in vector
- **Validation Overhead**: File accessibility checks on registration

## Design Rationale

### Post-Load Execution Model
**Why execute scripts after loading?**
- **System Readiness**: Ensures all objects and classes are available
- **Configuration**: Allows system configuration based on loaded state
- **Initialization**: Supports complex initialization sequences
- **Safety**: Prevents scripts from interfering with core loading

### Magic String Approach
**Why use magic strings instead of file extensions?**
- **Flexibility**: Scripts can be embedded in other file types
- **Shell Compatibility**: Allows shell script headers
- **Language Agnostic**: Works with various scripting approaches
- **Clear Boundaries**: Explicit script start markers

### Mode-Based Processing
**Why different script modes?**
- **Extensibility**: Support multiple script languages/parsers
- **Specialization**: Different modes for different use cases
- **Evolution**: Allows gradual feature addition
- **Compatibility**: Supports existing and future script formats

### Sequential Execution
**Why run scripts sequentially?**
- **Predictability**: Deterministic execution order
- **Dependency Management**: Later scripts can depend on earlier ones
- **Debugging**: Easier to trace execution and errors
- **Resource Safety**: Prevents resource conflicts

This implementation establishes the foundation for post-load scripting in RefPerSys, providing a flexible system for system initialization, configuration, and automation while maintaining the reflective and extensible nature of the platform.