# BisonC++ Parser Class Header Template (`bisonc++.h`)

## Overview

The `bisonc++.h` file is the header template for BisonC++-generated parser classes. It defines the public interface of the parser, including the main parse function and declarations for user-overridable methods like lexical analysis and error handling.

## File Structure

### Include Guards and Headers
```cpp
#ifndef @$_h_included
#define @$_h_included

$insert baseclass    // Include base class header
$insert scanner.h     // Include scanner interface
```

### Namespace Management
```cpp
$insert namespace-open

$insert undefparser   // Clean up parser name macros

// Parser class definition here

$insert namespace-close
```

## Parser Class Definition

### Class Declaration
```cpp
class @ : public @Base
{
    $insert 4 scannerobject   // Scanner member variable

public:
    @() = default;           // Default constructor
    int parse();             // Main parsing function

private:
    // User-overridable methods
    void error();                   // Syntax error handler
    int lex();                     // Lexical analyzer
    void print();                  // Token printing
    void exceptionHandler(std::exception const &exc);  // Exception handler

    // Support functions for parse()
    void executeAction_(int ruleNr);    // Execute grammar actions
    void errorRecovery_();              // Error recovery
    void nextCycle_();                  // Main parsing cycle
    void nextToken_();                  // Token acquisition
    void print_();                     // Internal printing
};
```

## Key Interface Methods

### Public Interface

#### `int parse()`
**Purpose**: Main entry point for parsing
- **Returns**: 0 on successful parse, error count on failure
- **Exceptions**: May throw exceptions during parsing
- **Thread Safety**: Not thread-safe (uses internal state)

**Usage**:
```cpp
@ parser;
int result = parser.parse();
if (result == 0) {
    // Parsing successful
} else {
    // result indicates number of errors
}
```

### Private User-Overridable Methods

#### `void error()`
**Purpose**: Handle syntax errors
- **Default Implementation**: Prints "Syntax error" to stderr
- **Override**: Provide custom error reporting
- **Called**: When parser encounters syntax error

**Example Override**:
```cpp
void MyParser::error() {
    std::cerr << "Parse error at line " << lineNumber << std::endl;
}
```

#### `int lex()`
**Purpose**: Return next token from input
- **Returns**: Token code (positive) or 0 for EOF
- **Implementation**: Must be provided by user
- **Integration**: Called by `nextToken_()` when tokens needed

**Example Implementation**:
```cpp
int MyParser::lex() {
    // Get next token from input stream
    // Set semantic value if needed
    // Return token code
    return TOKEN_IDENTIFIER;
}
```

#### `void print()`
**Purpose**: Print current token information
- **Default Implementation**: Empty (no output)
- **Override**: Enable token debugging output
- **Called**: After each token is consumed

**Example Override**:
```cpp
void MyParser::print() {
    std::cout << "Token: " << symbol_(token_()) << std::endl;
}
```

#### `void exceptionHandler(std::exception const &exc)`
**Purpose**: Handle exceptions thrown during parsing
- **Default Implementation**: Re-throws the exception
- **Override**: Provide custom exception handling
- **Called**: When grammar actions throw exceptions

**Example Override**:
```cpp
void MyParser::exceptionHandler(std::exception const &exc) {
    std::cerr << "Parser exception: " << exc.what() << std::endl;
    throw;  // Re-throw or handle differently
}
```

## Internal Support Methods

### Parsing Engine Methods
These methods implement the core parsing algorithm and are called internally:

- **`executeAction_(int ruleNr)`**: Execute semantic actions for grammar rules
- **`errorRecovery_()`**: Perform error recovery when syntax errors occur
- **`nextCycle_()`**: Execute one step of the parsing algorithm
- **`nextToken_()`**: Acquire next token from lexical analyzer
- **`print_()`**: Internal token printing (debug support)

## Template Integration

### Macro Expansion Points
- **`@`**: Parser class name (e.g., `MyParser`)
- **`@Base`**: Base class name (e.g., `MyParserBase`)
- **`$insert scannerobject`**: Scanner member variable declaration
- **`$insert namespace-open/close`**: Namespace management

### Generated Code Structure
```
MyParser : public MyParserBase
├── Public interface
│   └── int parse()
├── User-overridable methods
│   ├── void error()
│   ├── int lex()
│   ├── void print()
│   └── void exceptionHandler(std::exception const&)
└── Internal parsing engine
    ├── executeAction_()
    ├── errorRecovery_()
    ├── nextCycle_()
    ├── nextToken_()
    └── print_()
```

## Usage Pattern

### Basic Parser Implementation
```cpp
// Generated header
#include "myparser.h"

// User implementation
class MyParser : public MyParserBase {
public:
    int lex() override {
        // Implement lexical analysis
        return getNextToken();
    }

    void error() override {
        // Custom error handling
        std::cerr << "Parse error!" << std::endl;
    }
};

// Usage
MyParser parser;
if (parser.parse() == 0) {
    std::cout << "Parsing successful!" << std::endl;
}
```

### Advanced Features
```cpp
class AdvancedParser : public AdvancedParserBase {
    int lineNumber = 1;

    int lex() override {
        // Track line numbers
        int token = getNextToken();
        if (token == NEWLINE) lineNumber++;
        return token;
    }

    void error() override {
        std::cerr << "Error at line " << lineNumber << std::endl;
    }

    void print() override {
        // Debug token output
        std::cout << "Line " << lineNumber << ": "
                  << symbol_(token_()) << std::endl;
    }

    void exceptionHandler(std::exception const &exc) override {
        std::cerr << "Parser exception at line " << lineNumber
                  << ": " << exc.what() << std::endl;
        // Custom recovery logic
    }
};
```

## Integration with Base Class

### Inherited Functionality
The parser class inherits extensive functionality from `@Base`:

- **State Management**: Parser state stack and current state
- **Token Handling**: Current token and lookahead management
- **Debug Support**: Debug output and tracing facilities
- **Error Recovery**: Automatic error recovery mechanisms
- **Stack Operations**: Push/pop operations for states and values

### Base Class Methods Available
- **`setDebug(bool)`**: Enable/disable debug output
- **`setDebug(DebugMode_)`**: Fine-grained debug control
- **State Access**: `state_()`, `top_()`, `stackSize_()`
- **Token Access**: `token_()`, `savedToken_()`

## Error Handling

### Error Recovery
- **Automatic**: Base class handles most error recovery
- **Customizable**: Override `error()` for custom messages
- **Exception Handling**: Override `exceptionHandler()` for action exceptions

### Error Return Codes
- **`0`**: Successful parse
- **`1`**: Parse aborted (PARSE_ABORT_)
- **`> 1`**: Number of syntax errors encountered

## Performance Considerations

### Memory Usage
- **Minimal Overhead**: Inherits efficient base class implementation
- **Stack Growth**: Automatic stack expansion as needed
- **Token Buffering**: Efficient token lookahead management

### Execution Speed
- **Fast Lookup**: Optimized state transition tables
- **Inline Methods**: Many accessors are inline for speed
- **Minimal Indirection**: Direct access to parser state

This header template provides a clean, extensible interface for BisonC++-generated parsers, allowing users to focus on lexical analysis and semantic actions while inheriting robust parsing infrastructure.