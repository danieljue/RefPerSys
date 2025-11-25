# BisonC++ Parser Implementation Header Template (`bisonc++.ih`)

## Overview

The `bisonc++.ih` file is the implementation header template that provides default implementations for the user-overridable methods declared in the parser class header. It serves as the bridge between the generated parser interface and user-provided implementations.

## File Structure

### Include and Setup
```cpp
// Include this file in the sources of the class @
$insert class.h    // Include the parser class header

$insert namespace-open  // Open parser namespace

// Default method implementations
inline void @::error() { ... }
$insert lex           // Lexical analyzer template
inline void @::print() { ... }
inline void @::exceptionHandler(std::exception const &exc) { ... }

$insert namespace-close  // Close parser namespace

// Additional includes can be added here
$insert namespace-use   // Namespace usage directives
```

## Default Method Implementations

### Error Handling
```cpp
inline void @::error()
{
    std::cerr << "Syntax error\n";
}
```

**Purpose**: Default syntax error handler
- **Output**: Prints "Syntax error" to standard error
- **Override**: Replace with custom error reporting
- **Called**: When parser encounters unrecoverable syntax error

### Lexical Analysis
```cpp
$insert lex  // Template for lexical analyzer implementation
```

**Purpose**: Template for token acquisition
- **Implementation**: Must be provided by user
- **Returns**: Token code from lexical analyzer
- **Integration**: Called by parser's `nextToken_()` method

### Token Printing
```cpp
inline void @::print()
{
    // Default: no output
}
```

**Purpose**: Token debugging output
- **Default**: Silent (no output)
- **Override**: Enable token tracing during parsing
- **Called**: After each token is processed

### Exception Handling
```cpp
inline void @::exceptionHandler(std::exception const &exc)
{
    throw;  // Re-throw by default
}
```

**Purpose**: Handle exceptions from semantic actions
- **Default**: Re-throws the exception
- **Override**: Provide custom exception handling
- **Called**: When grammar actions throw exceptions

## Template Integration

### Macro Expansion Points
- **`@`**: Parser class name (e.g., `MyParser`)
- **`$insert lex`**: Lexical analyzer implementation template
- **`$insert namespace-open/close`**: Namespace management
- **`$insert namespace-use`**: `using namespace` directives

### Generated Structure
```
#include "myparser.h"        // Parser class header

namespace ParserNamespace {

// Default implementations
inline void MyParser::error() {
    std::cerr << "Syntax error\n";
}

inline int MyParser::lex() {
    // User must implement this
    return 0;  // Placeholder
}

inline void MyParser::print() {
    // Default: no-op
}

inline void MyParser::exceptionHandler(std::exception const &exc) {
    throw;  // Re-throw
}

}  // namespace ParserNamespace

// Additional user includes can go here
// using namespace std;  // Optional
```

## Usage Patterns

### Basic Implementation
```cpp
// Include the implementation header
#include "myparser.ih"

// User provides minimal implementation
class MyParser : public MyParserBase {
    int lex() override {
        // Implement lexical analysis
        return getNextToken();
    }
};

// The .ih file provides defaults for:
// - error() -> prints "Syntax error"
// - print() -> does nothing
// - exceptionHandler() -> re-throws
```

### Advanced Implementation
```cpp
#include "myparser.ih"

class AdvancedParser : public AdvancedParserBase {
    int lineNumber = 1;
    std::string currentTokenText;

    int lex() override {
        // Implementation provided by user
        int token = scanner.getNextToken();
        currentTokenText = scanner.getTokenText();
        if (token == NEWLINE) lineNumber++;
        return token;
    }

    void error() override {
        // Custom error with location
        std::cerr << "Syntax error at line " << lineNumber
                  << " near '" << currentTokenText << "'" << std::endl;
    }

    void print() override {
        // Enable token debugging
        std::cout << "Line " << lineNumber << ": Token "
                  << symbol_(token_()) << " = '" << currentTokenText << "'"
                  << std::endl;
    }

    void exceptionHandler(std::exception const &exc) override {
        // Custom exception handling
        std::cerr << "Parser exception at line " << lineNumber
                  << ": " << exc.what() << std::endl;
        // Could implement recovery logic here
        throw;  // Or handle differently
    }
};
```

## Integration with Build System

### Include Order
```cpp
// In parser source file (.cc)
#include "myparser.h"     // Class declaration
#include "myparser.ih"    // Default implementations

// User implementations
int MyParser::lex() {
    // Custom lexer implementation
}
```

### Compilation Units
- **Header (.h)**: Class interface declaration
- **Implementation Header (.ih)**: Default method implementations
- **Source (.cc)**: User-provided implementations and main logic

## Method Override Requirements

### Required Overrides
- **`int lex()`**: Must be implemented by user
  - Provides tokens to the parser
  - Should return token codes defined in grammar
  - May set semantic values for tokens

### Optional Overrides
- **`void error()`**: Default prints "Syntax error"
  - Customize error messages and recovery
  - Access parser state for detailed error info

- **`void print()`**: Default does nothing
  - Enable debugging output
  - Show token consumption during parsing

- **`void exceptionHandler(std::exception const &exc)`**: Default re-throws
  - Handle exceptions from semantic actions
  - Implement custom error recovery

## Error Handling Strategy

### Default Behavior
```cpp
// Syntax errors
void @::error() {
    std::cerr << "Syntax error\n";  // Basic message
}

// Action exceptions
void @::exceptionHandler(std::exception const &exc) {
    throw;  // Re-throw - parser will handle
}
```

### Custom Error Handling
```cpp
class MyParser : public MyParserBase {
    void error() override {
        // Access parser state for detailed errors
        std::cerr << "Parse error at token " << symbol_(token_())
                  << " in state " << state_() << std::endl;
    }

    void exceptionHandler(std::exception const &exc) override {
        // Log and potentially recover
        logParserException(exc);
        if (canRecoverFrom(exc)) {
            performRecovery();
        } else {
            throw;
        }
    }
};
```

## Debug Support

### Token Tracing
```cpp
void @::print() {
    // Access current token information
    std::cout << "Processing token: " << symbol_(token_())
              << " (code: " << token_() << ")" << std::endl;
}
```

### Available Debug Information
- **`token_()`**: Current token code
- **`symbol_(token_())`**: Token name/symbol
- **`state_()`**: Current parser state
- **`vs_(idx)`**: Semantic values on stack
- **`d_val_`**: Current semantic value

## Performance Considerations

### Inline Methods
- **All methods are `inline`**: Minimal function call overhead
- **Default implementations**: Optimized for no-op cases
- **Override impact**: Only overridden methods incur call overhead

### Memory Usage
- **Zero overhead**: Default implementations generate no code if not used
- **Minimal footprint**: Only overridden methods contribute to binary size

### Compilation Speed
- **Template-based**: Fast compilation with minimal dependencies
- **Separate compilation**: Implementation header can be pre-compiled

This implementation header template provides a flexible foundation for BisonC++ parsers, offering sensible defaults while allowing complete customization of parser behavior through method overrides.