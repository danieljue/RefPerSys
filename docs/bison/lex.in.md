# BisonC++ Lexical Analyzer Template (`lex.in`)

## Overview

The `lex.in` file provides the template for the lexical analyzer interface in BisonC++ parsers. It defines how the parser obtains tokens from the lexical analyzer (lexer) and integrates with the parsing process.

## File Structure

### Lexical Analyzer Implementation
```cpp
#pragma message "RefPerSys bisonc++ lex.in"

inline int @::lex()
{
@printtokens
    print();
@end
    return @tokenfunction;
}
```

## Template Expansion

### Token Function Integration
```cpp
// @tokenfunction is replaced with user-specified token function name
// Default is typically "lex()" or "yylex()"

// Example expansions:
// return lex();      // Call member function
// return yylex();    // Call external function
// return getToken(); // Call custom function
```

### Debug Token Printing
```cpp
@printtokens
    print();  // Call debug print function
@end
```

**Purpose**: Enables token debugging when print() is implemented.

## Usage Patterns

### Member Function Lexer
```cpp
// In parser class
inline int MyParser::lex() {
    // User implements token acquisition
    return getNextToken();
}
```

### External Lexer Integration
```cpp
// Using external lexer (e.g., Flex)
extern int yylex();

inline int MyParser::lex() {
    print();  // Debug output
    return yylex();  // Call external lexer
}
```

### Custom Token Function
```cpp
// Specify custom function in grammar
%tokenfunction my_lexer

inline int MyParser::lex() {
    print();  // Debug output
    return my_lexer();  // Call custom lexer
}
```

## Integration with Parser

### Token Acquisition Flow
```cpp
// In parser's nextToken_()
void @::nextToken_() {
    if (token_() != Reserved_::UNDETERMINED_) {
        return;  // Token already available
    }
    
    if (savedToken_() != Reserved_::UNDETERMINED_) {
        popToken_();  // Use saved token
    } else {
        ++d_acceptedTokens_;
        lex_(lex());  // Call user lexer through template
        print_();     // Debug output
    }
}
```

### Token Processing
1. **Call `lex()`**: Get token code from user lexer
2. **Set Token**: Store in parser's token state
3. **Debug Output**: Print token if debugging enabled
4. **Continue Parsing**: Use token in state lookup

## Error Handling

### Invalid Tokens
```cpp
inline int @::lex() {
    int token = getToken();
    if (token < 0) {
        // Handle invalid tokens
        return Reserved_::EOF_;  // Or error token
    }
    print();
    return token;
}
```

### Lexer Exceptions
```cpp
inline int @::lex() {
    try {
        int token = lexer.getToken();
        print();
        return token;
    } catch (LexerError &e) {
        // Handle lexer exceptions
        error();  // Trigger parser error
        return Reserved_::errTok_;
    }
}
```

## Performance Considerations

### Inline Implementation
- **Zero Call Overhead**: `inline` eliminates function call
- **Direct Integration**: Lexer code integrated into parser
- **Optimization**: Compiler can optimize across boundaries

### Token Buffering
- **No Buffering**: Each call gets next token
- **Stateful Lexers**: Lexer maintains its own state
- **Reentrancy**: Depends on lexer implementation

## Customization Options

### Advanced Lexer Integration
```cpp
inline int MyParser::lex() {
    // Pre-token processing
    prepareForToken();
    
    // Get token with context
    int token = lexer.getToken(currentContext);
    
    // Post-token processing
    processToken(token);
    
    // Debug output
    print();
    
    return token;
}
```

### Multiple Lexer Support
```cpp
inline int MyParser::lex() {
    int token;
    
    // Switch lexers based on state
    if (inStringMode) {
        token = stringLexer.getToken();
    } else {
        token = mainLexer.getToken();
    }
    
    print();
    return token;
}
```

### Token Filtering
```cpp
inline int MyParser::lex() {
    int token = rawLexer.getToken();
    
    // Filter or transform tokens
    if (token == WHITESPACE) {
        return lex();  // Skip whitespace, get next token
    }
    
    print();
    return token;
}
```

This lexical analyzer template provides a flexible interface for integrating any token source with BisonC++ parsers, supporting both simple and complex lexer integrations with full debug support.