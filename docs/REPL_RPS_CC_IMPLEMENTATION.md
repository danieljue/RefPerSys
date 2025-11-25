# REPL Implementation Analysis (`repl_rps.cc`)

## Overview

The `repl_rps.cc` file implements the Read-Eval-Print Loop (REPL) functionality for the Reflective Persistent System (RefPerSys), providing an interactive command interface for users to interact with the system. This file handles command parsing, execution, and provides debugging facilities for the parser and runtime system. The REPL supports both traditional command-line style interactions and advanced debugging commands for system introspection.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema, Niklas Rozencrantz
- **Copyright**: © 2019-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"
#include "wordexp.h"  // For shell word expansion

// Git versioning information
extern "C" const char rps_repl_gitid[];
extern "C" const char rps_repl_date[];
extern "C" const char rps_repl_shortgitid[];
extern "C" const char rps_repl_timestamp[];

// External function declarations
extern "C" void rps_do_builtin_repl_command(Rps_CallFrame* callframe, ...);
extern "C" rps_applyingfun_t rpsapply_repl_not_implemented;
```

### Key Dependencies
- **refpersys.hh**: Core system definitions and Rps_TokenSource class
- **wordexp.h**: POSIX shell word expansion for command parsing
- **Generated parser**: Auto-generated parser code from Carburetta
- **Token management**: Rps_LexTokenZone and token stream handling

## REPL Command Creation and Management

### Command Registration System
```cpp
void rps_repl_create_command(Rps_CallFrame* callframe, const char* commandname)
{
  // Creates a new REPL command object with associated function
  // Generates C++ skeleton code for command implementation
  // Registers command in the repl_command class
}
```

**Purpose**: Dynamically creates REPL commands at runtime, generating both the command object and skeleton implementation code.

### Command Structure
Each REPL command consists of:
- **Command Object**: Instance of `repl_command` class
- **Parser Function**: Closure that handles command parsing and execution
- **Symbol Association**: Symbol object for command name lookup
- **Function Object**: Core function with applying function for execution

### Command Registration Process
1. **Validate Command Name**: Check for alphanumeric characters and underscores
2. **Create Symbol**: Generate unique symbol for command name
3. **Create Function Object**: Instantiate core function with stub implementation
4. **Create Command Object**: Build repl_command instance
5. **Generate Code**: Output C++ skeleton for command implementation
6. **Register Command**: Add to repl_command class components

## Built-in REPL Commands

### System Information Commands
```cpp
// Process information
rps_repl_builtin_pid_command()      // Display process ID
rps_repl_builtin_getpid_command()   // Alias for pid

// Version information
rps_repl_builtin_git_command()      // Display git commit information
rps_repl_builtin_version_command()  // Display system version

// Timing information
rps_repl_builtin_time_command()     // Display elapsed and CPU time
```

### File System Commands
```cpp
// Directory operations
rps_repl_builtin_pwd_command()      // Print working directory
rps_repl_builtin_cd_command()       // Change directory with word expansion

// File descriptors
rps_repl_builtin_pfd_command()      // List open file descriptors
rps_repl_builtin_pmap_command()     // Display process memory map
```

### System Control Commands
```cpp
// Garbage collection
rps_repl_builtin_gc_command()       // Trigger garbage collection

// Type information
rps_repl_builtin_typeinfo_command() // Display type system information

// Environment inspection
rps_repl_builtin_env_command()      // Display current environment
```

### Shell Integration
```cpp
// Shell command execution
rps_repl_builtin_shell_command()    // Execute shell commands
rps_repl_builtin_sh_command()       // Alias for shell command
```

## Parser Testing and Debugging Commands

### Expression Parsing Tests
```cpp
rps_repl_builtin_parse_expression_command()    // Test expression parsing
rps_repl_builtin_parse_disjunction_command()   // Test disjunction parsing
rps_repl_builtin_parse_conjunction_command()   // Test conjunction parsing
rps_repl_builtin_parse_comparison_command()    // Test comparison parsing
rps_repl_builtin_parse_sum_command()           // Test sum parsing
rps_repl_builtin_parse_product_command()       // Test product parsing
rps_repl_builtin_parse_term_command()          // Test term parsing
rps_repl_builtin_parse_factor_command()        // Test factor parsing
rps_repl_builtin_parse_primary_command()       // Test primary parsing
```

**Purpose**: Debug and test individual levels of the expression parser hierarchy.

### Command Dispatch System
```cpp
void rps_do_builtin_repl_command(Rps_CallFrame* callframe, ...)
{
  // Dispatch to appropriate builtin command handler
  // Supports extensible command registration
}
```

**Command Recognition**: Commands prefixed with `!` are treated as builtin commands.

## REPL Command Execution Infrastructure

### Single Command Execution
```cpp
void rps_do_one_repl_command(Rps_CallFrame* callframe, Rps_ObjectRef obenvarg,
                            const std::string& cmd, const char* title)
{
  // Parse and execute a single REPL command
  // Handle builtin commands (!prefix) and regular commands
}
```

### Command Processing Flow
1. **Tokenize Input**: Create Rps_StringTokenSource from command string
2. **Check Builtin**: Commands starting with `!` are builtin
3. **Check Carburetta**: Commands starting with `@` use generated parser
4. **Parse Command**: Extract command object/symbol from first token
5. **Execute Parser**: Apply command's parser closure
6. **Handle Results**: Process parser output and update environment

### Environment Management
```cpp
// Environment object creation
_f.envob = Rps_ObjectRef::make_object(&_,
                                     RPS_ROOT_OB(_5LMLyzRp6kq04AMM8a)); // environment∈class
auto paylenv = _f.envob->put_new_plain_payload<Rps_PayloadEnvironment>();
```

**Purpose**: Each command execution maintains its own environment for variable bindings and state.

## Lexical Token Management

### REPL Token Source
```cpp
// Token source with keyword lexing support
Rps_StringTokenSource intoksrc(cmd, std::string(title) + "°repl");
intoksrc.set_keyword_lexing_fun(rps_carbrepl_keyword_lexer);
```

### Token Types in REPL Context
- **Object Tokens**: Command objects and symbols
- **Delimiter Tokens**: Special parsing delimiters
- **Literal Tokens**: Strings, numbers, etc.
- **Keyword Tokens**: Special REPL keywords

### Token Processing
```cpp
// Extract command from first token
_f.lextokv = intoksrc.get_token(&_);
const Rps_LexTokenZone* lextokz = _f.lextokv.as_lextoken();

// Handle object and symbol commands
if (lextokz->lxkind() == RPS_ROOT_OB(_5yhJGgxLwLp00X0xEQ)) { // object∈class
  _f.cmdob = lextokz->lxval().as_object();
}
```

## Code Chunk Element Parsing (Obsolete)

### Legacy Code Chunk Support
```cpp
Rps_Value rps_lex_chunk_element(Rps_CallFrame* callframe,
                               Rps_ObjectRef obchkarg, Rps_ChunkData_st* chkdata)
{
  // Parse elements within code chunks
  // Handles metavariables, spaces, delimiters
  // Marked as obsolete code
}
```

**Status**: Legacy implementation from older code chunk system, now disabled.

### Chunk Element Types
- **Names**: Convert to object references or strings
- **Spaces**: Represented as space class instances with count
- **Metavariables**: `$name` syntax for template variables
- **Delimiters**: Special markers for chunk boundaries

## REPL Command Vector Processing

### Batch Command Execution
```cpp
void rps_do_repl_commands_vec(const std::vector<std::string>& cmdvec)
{
  // Execute a vector of REPL commands
  // Create shared environment for command sequence
  // Handle exceptions and continue processing
}
```

### Batch Processing Features
- **Shared Environment**: Commands share environment state
- **Error Recovery**: Exceptions don't stop batch processing
- **Progress Indication**: Debug logging for command execution
- **Rate Limiting**: Small delays between commands to prevent overwhelming output

### Exception Handling
```cpp
try {
  rps_do_one_repl_command(&_, _f.envob, cmdvec[cix], bufpath);
} catch (std::exception& ex) {
  // Demangle exception type name
  char* extyprealname = abi::__cxa_demangle(extyprawname.c_str(),
                                           nullptr, nullptr, &status);
  RPS_WARNOUT("REPL command#" << cix << " failed with exception: " << ex.what());
}
```

## Command Syntax and Patterns

### Builtin Command Syntax
```bash
# Builtin commands start with !
!pwd                    # Print working directory
!cd /tmp                # Change directory
!pid                    # Show process ID
!git                    # Show git information
!time                   # Show timing information
!gc                     # Trigger garbage collection
!env                    # Show environment
!parse_expression 2+3   # Test expression parsing
```

### Regular Command Syntax
```bash
# Commands are object references or symbols
some_command arg1 arg2  # Execute command object
@special_command args   # Use Carburetta-generated parser
```

### Parser Testing Commands
```bash
# Test individual parser levels
!parse_primary 42
!parse_term 2*3+4
!parse_expression x && y || z
!parse_sum a+b-c
```

## Integration with Parser System

### Parser Testing Integration
```cpp
// Commands delegate to parser testing functions
_f.parvalv = intoksrc.parse_expression(&_, &ok);
if (ok) {
  RPS_INFORMOUT("parse_expression result: " << _f.parvalv);
}
```

**Purpose**: REPL provides direct access to parser functionality for debugging and testing.

### Environment Integration
```cpp
// Commands operate within environment context
_f.obenv = obenvarg;
auto paylenv = _f.obenv->get_dynamic_payload<Rps_PayloadEnvironment>();
```

**Purpose**: REPL commands can inspect and modify the current environment state.

## Error Handling and Debugging

### Command Failure Handling
```cpp
if (!_f.parsmainv && !_f.parsextrav) {
  RPS_WARNOUT("REPL command failed using " << _f.cmdparserv);
  return;
}
```

### Debug Logging
```cpp
RPS_DEBUG_LOG(REPL, "rps_do_one_repl_command " << title
              << " cmd=" << Rps_QuotedC_String(cmd));
```

**Purpose**: Extensive logging for REPL command processing and debugging.

### Position Tracking
```cpp
std::string commandpos = intoksrc.position_str();
// Track parsing position for error reporting
```

## Implementation Status and Limitations

### Incomplete Features
- **Carburetta Integration**: `@` commands use generated parser (marked TODO)
- **Symbol Command Handling**: Symbol-based commands are unimplemented
- **Code Chunk Parsing**: Legacy chunk parsing is disabled
- **Command Completion**: Tab completion infrastructure exists but unused

### Extensibility Mechanisms
- **Dynamic Command Creation**: `rps_repl_create_command()` generates new commands
- **Builtin Registration**: `rps_do_builtin_repl_command()` dispatches to handlers
- **Parser Integration**: Direct access to all parser levels for testing

### Performance Considerations
- **Token Caching**: Token sources cache parsed tokens
- **Environment Sharing**: Batch commands share environment objects
- **Lazy Evaluation**: Commands execute on demand
- **Exception Safety**: Robust error handling prevents crashes

## Usage Examples

### Basic REPL Interaction
```cpp
// Start REPL session
rps_repl_interpret(callframe, nullptr, "stdin", lineno);

// Execute builtin commands
!pwd
!cd /home/user
!pid
!time
!gc
```

### Parser Debugging
```cpp
// Test parser components
!parse_primary 42
!parse_term x*y+z
!parse_expression a && b || c
!parse_sum 1+2-3*4
```

### Command Creation
```cpp
// Create new REPL command (generates C++ skeleton)
rps_repl_create_command(callframe, "mycommand");
// Outputs C++ code for implementing mycommand
```

### Batch Command Processing
```cpp
std::vector<std::string> commands = {
  "!pwd",
  "!time",
  "some_command arg1 arg2"
};
rps_do_repl_commands_vec(commands);
```

This implementation provides a comprehensive REPL system that serves as both a user interface and a debugging tool for the RefPerSys parser and runtime, with extensive facilities for testing and introspection of the system's parsing and execution capabilities.