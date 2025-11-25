# BisonC++ Token Printing Template (`print.in`)

## Overview

The `print.in` file provides the token printing functionality for BisonC++ parsers. It defines how tokens are displayed during parsing, supporting both debug output and user-defined token display.

## File Structure

### Token Printing Implementation
```cpp
#pragma message "RefPerSys bisonc++ print.in matchedtextfunction= @matchedtextfunction"

std::cout << "Token: " << symbol_(token_()) << ", text: `";
if (token_() == Reserved_::UNDETERMINED_)
    std::cout << "'\n";
else
    std::cout << @matchedtextfunction << "'\n";
```

## Token Display Components

### Token Information Output
```cpp
// Output format: "Token: TOKEN_NAME, text: `TOKEN_TEXT'"
Token: 'IDENTIFIER', text: `myVariable'
Token: '+', text: `+'
Token: 'NUMBER', text: `123'
```

### Components:
- **Token Name**: From `symbol_(token_())` - human-readable token identifier
- **Token Text**: Actual matched text from input
- **Formatting**: Single quotes for names, backticks for text

## Matched Text Function

### Template Parameter
```cpp
// @matchedtextfunction is replaced with user-specified function
// Default options:
// - matched()     - For scanners with matched() method
// - YYText        - For Flex-generated lexers
// - str()         - For custom string access
```

### Common Implementations
```cpp
// For BisonC++ scanner
std::cout << matched() << "'\n";

// For Flex lexer
std::cout << YYText << "'\n";

// For custom lexer
std::cout << currentTokenText() << "'\n";
```

## Integration with Parser

### Automatic Token Printing
```cpp
// In nextToken_() after calling lex()
void @::nextToken_() {
    // ... token acquisition ...
    lex_(lex());
    print_();  // Calls the print function
}
```

### Debug Output Control
```cpp
// Only prints when debugging is enabled
void print_() {
    $insert print  // Expands to token printing code
}
```

## Usage Examples

### Basic Token Display
```
Token: 'IDENTIFIER', text: `variableName'
Token: '(', text: `('
Token: 'NUMBER', text: `42'
Token: '+', text: `+'
Token: 'IDENTIFIER', text: `anotherVar'
Token: ')', text: `)'
```

### Special Cases
```cpp
// Undefined token
Token: 'UNDETERMINED_', text: `'

// End of input
Token: 'EOF_', text: `'
```

## Customization Options

### Enhanced Token Display
Users can override the `print()` function for custom output:

```cpp
void MyParser::print() {
    std::cout << "Line " << currentLine << ": ";
    std::cout << "Token: " << symbol_(token_()) << ", text: `";
    if (token_() == Reserved_::UNDETERMINED_)
        std::cout << "'\n";
    else
        std::cout << matched() << "'\n";
}
```

### Selective Printing
```cpp
void MyParser::print() {
    // Only print certain tokens
    if (token_() == IDENTIFIER || token_() == NUMBER) {
        std::cout << "Token: " << symbol_(token_()) 
                  << ", text: `" << matched() << "'\n";
    }
}
```

### Formatted Output
```cpp
void MyParser::print() {
    // Structured output
    std::cout << std::left << std::setw(15) << symbol_(token_())
              << " | " << matched() << std::endl;
}
```

## Performance Considerations

### Output Frequency
- **Per Token**: Called for every token consumed
- **I/O Overhead**: Direct console output
- **Debug Only**: Typically disabled in production

### Conditional Execution
```cpp
// Can be made conditional
void print_() {
    if (debugTokens) {
        $insert print
    }
}
```

## Error Handling

### Text Access Errors
```cpp
// Handle cases where matched text is unavailable
std::cout << "Token: " << symbol_(token_()) << ", text: `";
try {
    std::cout << @matchedtextfunction;
} catch (...) {
    std::cout << "<?>";
}
std::cout << "'\n";
```

### Encoding Issues
```cpp
// Handle non-printable characters
std::string safeText = escapeNonPrintable(matched());
std::cout << "Token: " << symbol_(token_()) 
          << ", text: `" << safeText << "'\n";
```

## Integration with Debug System

### Combined Debug Output
```cpp
// Token printing can be part of larger debug output
if (d_debug_) {
    s_out_ << "Consuming token: " << symbol_(token_())
           << " with text: `" << matched() << "'" << std::endl;
    dflush_(s_out_);
    
    // Also call regular print for additional output
    print_();
}
```

### Location-Aware Printing
```cpp
void MyParser::print() {
    LTYPE_ const &loc = lsp_(0);
    std::cout << "Line " << loc.first_line << ": Token: " 
              << symbol_(token_()) << ", text: `" << matched() << "'\n";
}
```

## Compatibility Notes

### Scanner Integration
- **BisonC++ Scanner**: Uses `matched()` method
- **Flex**: Uses `YYText` macro
- **Custom Lexers**: Requires appropriate text access function

### Optional Feature
- **Default**: Prints to stdout
- **Override**: Users can customize or disable
- **Performance**: Can be expensive for large inputs

This token printing template provides clear visibility into the token stream during parsing, supporting both automatic debug output and user-customized token display for development and troubleshooting purposes.