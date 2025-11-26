# Simple Interpreter Plugin (`plugins_dir/rpsplug_simpinterp.cc`)

**File Path:** `plugins_dir/rpsplug_simpinterp.cc`

## Overview

This RefPerSys plugin implements a simple interpreter for executing scripts. The plugin is designed to provide a scripting interface for RefPerSys developers to modify the system heap, test modules, and interact with code generation components. While currently incomplete, it establishes the framework for a C-like scripting language approved for RefPerSys.

## Plugin Purpose

### Scripting and Testing Infrastructure
The plugin enables dynamic system interaction by:

- **Script Execution:** Running scripts to modify system state
- **Heap Manipulation:** Direct access to RefPerSys object heap
- **Module Testing:** Testing code generation and other modules
- **Developer Tools:** Providing scripting interface for development
- **System Modification:** Runtime system configuration and testing

## Implementation Status

### Current State
```cpp
#warning a lot of missing code in rpsplug_simpinterp.cc
```

**Status:** Framework created, interpreter implementation incomplete

### Design Intent
- **C-like Syntax:** Approved C-inspired scripting language
- **Heap Access:** Direct manipulation of RefPerSys objects
- **Module Testing:** Test harness for system components
- **Development Tool:** Aid in system development and debugging

## Command-Line Interface

### Basic Usage
```bash
./refpersys --plugin-after-load=plugins_dir/rpsplug_simpinterp.so \
            --plugin-arg=rpsplug_simpinterp:$SCRIPT_FILE
```

### Required Arguments
- **`--plugin-arg`**: Path to script file to execute

## Plugin Architecture

### Entry Point
```cpp
void rps_do_plugin(const Rps_Plugin* plugin)
```

**Execution Flow:**
1. **Argument Validation:** Check script file path
2. **File Validation:** Verify file exists and is readable
3. **Path Resolution:** Convert to absolute path
4. **File Type Check:** Ensure regular file
5. **Token Source Creation:** (Incomplete - framework only)

## Script File Validation

### Comprehensive File Checking
```cpp
if (access(plugarg, R_OK))
  RPS_FATALOUT("bad script argument - not readable");

char* rpath = realpath(plugarg, nullptr);
if (!rpath)
  RPS_FATALOUT("failed to realpath argument");

struct stat plugstat = {};
if (stat(rpath, &plugstat))
  RPS_FATALOUT("cannot stat script argument");

if ((plugstat.st_mode & S_IFMT) != S_IFREG) {
  RPS_WARNOUT("script argument not a regular file");
  return;
}
```

**Validation Requirements:**
- File must exist and be readable
- Path must be resolvable to absolute path
- Must be a regular file (not directory, device, etc.)
- Comprehensive error reporting

## Token Source Framework

### Memory File Token Source
```cpp
Rps_MemoryFileTokenSource toksrc(rpath);
RPS_WARNOUT("missing code: plugin " << plugin->plugin_name
            << " script " << rpath);
```

**Framework Components:**
- Token source for lexical analysis
- Memory-based file processing
- Foundation for parser integration

## Language Design

### C-like Syntax
```cpp
/// on whatsapp March 14 2024 Abhishek approved a C like syntax
// See https://github.com/RefPerSys/RefPerSys/issues/21
```

**Approved Syntax:**
- C-inspired syntax approved by core team
- Formal specification expected in Feb 2025
- GitHub issue for language design tracking

## Research References

### Interpreter Implementations
```cpp
#warning see https://piumarta.com/software/maru/original/boot-eval.c
#warning see https://github.com/GollyTicker/Mini-LISP
#warning see https://github.com/arichel/PicoScheme
#warning see https://github.com/mzazon/coreinterpreter/
```

**Reference Implementations:**
- **Maru:** Bootstrapping evaluator
- **Mini-LISP:** Small Lisp interpreter
- **PicoScheme:** Minimal Scheme implementation
- **CoreInterpreter:** Core language interpreter

## Use Cases

### Development and Testing
- **Heap Modification:** Direct object manipulation for testing
- **Module Testing:** Test code generation modules
- **System Configuration:** Runtime system setup and modification
- **Debugging Scripts:** Automated debugging procedures

### System Administration
- **Batch Operations:** Execute system maintenance scripts
- **Configuration Scripts:** Automated system configuration
- **Test Suites:** Run comprehensive system tests
- **Deployment Scripts:** System setup and initialization

## Build and Deployment

### Compilation
```bash
cd $REFPERSYS_TOPDIR
./do-build-refpersys-plugin -v \
  -i plugins_dir/rpsplug_simpinterp.cc \
  -o plugins_dir/rpsplug_simpinterp.so \
  -L /tmp/rpsplug_simpinterp.so
```

### Runtime Usage (When Complete)
```bash
# Execute a script
./refpersys --plugin-after-load=/tmp/rpsplug_simpinterp.so \
            --plugin-arg=rpsplug_simpinterp:my_script.rps \
            --repl

# Test with REPL
./refpersys --plugin-after-load=/tmp/rpsplug_simpinterp.so \
            --plugin-arg=rpsplug_simpinterp:test_script.rps \
            -AREPL
```

## Performance Characteristics

### Current Performance
- **File Validation:** Fast filesystem operations
- **Path Resolution:** Standard realpath operations
- **Framework Setup:** Minimal overhead

### Future Performance Goals
- **Script Parsing:** Efficient lexical analysis
- **Execution Speed:** Fast script interpretation
- **Memory Usage:** Minimal interpreter overhead
- **Scalability:** Handle large scripts

## Relationship to Other Plugins

### Complementary Plugins
- **Code Generation Plugins:** Test generated code
- **Object Creation Plugins:** Create objects via scripts
- **REPL Plugins:** Interactive alternatives

### Development Workflow
```
Write Scripts → Execute with Interpreter → Test System → Debug Issues
     ↓              ↓                      ↓            ↓
Script Files   Interpreter Execution    Validation   Fixes
```

## Thread Safety

### Synchronization
Plugin uses RefPerSys's existing file access patterns and should integrate with system's threading model when interpreter is implemented.

## Error Handling

### Validation Failures
- **Missing Script:** Requires script file path
- **File Access:** Must be readable regular file
- **Path Resolution:** Must resolve to valid absolute path

### Fatal Errors
```cpp
RPS_FATALOUT("without argument (script file expected)");
RPS_FATALOUT("bad script argument - not readable");
RPS_FATALOUT("failed to realpath argument");
```

## Integration with System

### Token Source Integration
```cpp
Rps_MemoryFileTokenSource toksrc(rpath);
```

**System Components:**
- Integrates with RefPerSys tokenization system
- Uses memory-based file processing
- Foundation for full parser integration

## File Status

**Status:** Framework/Incomplete
**Date:** 2024-2025
**Purpose:** Scripting infrastructure for RefPerSys development

## Summary

The `rpsplug_simpinterp.cc` plugin establishes the foundational infrastructure for a simple interpreter in RefPerSys. While currently incomplete, it demonstrates the approach to script execution with comprehensive file validation and integration points with the system's tokenization framework. The plugin is designed to provide a C-like scripting interface for developers to interact with the RefPerSys heap, test modules, and perform system modifications. Its references to other interpreter implementations and approval of C-like syntax indicate a well-thought-out design approach. Once completed, this plugin will be a crucial tool for RefPerSys development, enabling scripted testing, system configuration, and interactive development workflows. The plugin represents an important step toward making RefPerSys more accessible to developers through a familiar scripting interface.