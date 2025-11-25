# BisonC++ Location Type Template (`ltype.in`)

## Overview

The `ltype.in` file defines the location type (`LTYPE_`) used for tracking source code positions in BisonC++ parsers. Location tracking enables precise error reporting and debugging by maintaining line/column information for tokens and syntax elements.

## File Structure

### Location Type Definition
```cpp
#pragma message "RefPerSys bisonc++ ltype.in"

@ltype
@LTYPE
@else
struct LTYPE_
{
    int timestamp;
    int first_line;
    int first_column;
    int last_line;
    int last_column;
    char *text;
};
@end
```

## Location Information Fields

### Position Tracking
```cpp
struct LTYPE_ {
    int timestamp;     // File timestamp or version
    int first_line;    // Starting line number
    int first_column;  // Starting column number
    int last_line;     // Ending line number
    int last_column;   // Ending column number
    char *text;        // Optional token text
};
```

### Field Purposes
- **Line Numbers**: Track which lines tokens span
- **Column Numbers**: Track horizontal position within lines
- **Timestamp**: Enable cache invalidation for edited files
- **Text**: Store actual token text for detailed error messages

## Template Expansion

### Custom Location Types
```cpp
@ltype
// User-defined location type
struct MyLocation {
    std::string filename;
    int line;
    int column;
    int length;
};
@LTYPE
using LTYPE_ = MyLocation;
@else
// Default location type
struct LTYPE_ {
    int timestamp;
    int first_line;
    int first_column;
    int last_line;
    int last_column;
    char *text;
};
@end
```

## Integration with Parser

### Location Stack Management
```cpp
// In bisonc++base.h
$insert LTYPEstack   // Location stack declarations
$insert LTYPEdata    // Location data members
```

### Stack Operations
```cpp
// Location stack parallel to state stack
std::vector<LTYPE_> d_locationStack;
LTYPE_ *d_lsp;  // Location stack pointer

// Push/pop operations maintain location info
void push_(size_t state) {
    // ... state push ...
    $insert LTYPEpush  // Location push
}

void pop_(size_t count) {
    // ... state pop ...
    $insert LTYPEpop   // Location pop
}
```

## Usage in Error Reporting

### Enhanced Error Messages
```cpp
void MyParser::error() {
    LTYPE_ const &loc = d_lsp[0];  // Current location
    std::cerr << "Error at line " << loc.first_line 
              << ", column " << loc.first_column;
    if (loc.text) {
        std::cerr << ": near '" << loc.text << "'";
    }
    std::cerr << std::endl;
}
```

### Location Propagation
```cpp
// In grammar rules
expression:
    expr '+' expr {
        $$ = $1 + $3;
        // Location spans from $1 to $3
        $$.first_line = $1.first_line;
        $$.first_column = $1.first_column;
        $$.last_line = $3.last_line;
        $$.last_column = $3.last_column;
    }
```

## Memory Management

### Text Field Handling
```cpp
// Text field requires careful memory management
struct LTYPE_ {
    // ... other fields ...
    char *text;  // Must be managed properly
};

// Constructor
LTYPE_(const char *tokenText) {
    text = tokenText ? strdup(tokenText) : nullptr;
}

// Destructor
~LTYPE_() {
    free(text);
}
```

### Stack Expansion
```cpp
// When location stack grows
d_locationStack.resize(newSize);
$insert LTYPEresize  // Initialize new locations
```

## Performance Considerations

### Memory Overhead
- **Per Stack Element**: Size of LTYPE_ structure
- **Text Storage**: Additional memory for token strings
- **Stack Operations**: Location push/pop with each state change

### CPU Overhead
- **Location Updates**: Additional assignments in grammar actions
- **Memory Allocation**: `strdup()` calls for text storage
- **Stack Management**: Parallel location stack operations

## Customization Options

### Extended Location Information
```cpp
@ltype
struct DetailedLocation {
    std::string filename;
    int line_start, column_start;
    int line_end, column_end;
    std::string token_text;
    int file_id;  // For multi-file parsing
};
@LTYPE
using LTYPE_ = DetailedLocation;
@end
```

### Location Utilities
```cpp
// Helper functions for location handling
std::string formatLocation(LTYPE_ const &loc) {
    std::ostringstream oss;
    oss << "line " << loc.first_line;
    if (loc.first_column > 0) {
        oss << ", col " << loc.first_column;
    }
    return oss.str();
}
```

## Integration with Debug System

### Location-Aware Debugging
```cpp
// Enhanced debug output with location
if (d_debug_) {
    LTYPE_ const &loc = d_lsp[0];
    s_out_ << "Token at " << formatLocation(loc) << ": "
           << symbol_(token_()) << std::endl << dflush_;
}
```

### Error Recovery with Location
```cpp
void errorRecovery_() {
    // ... standard recovery ...
    
    LTYPE_ const &errorLoc = d_lsp[0];
    std::cout << "Error recovery at " << formatLocation(errorLoc) 
              << std::endl;
}
```

## Compatibility Notes

### Optional Feature
- **Default**: No location tracking if not specified
- **Activation**: Enabled by `%locations` directive in grammar
- **Backward Compatibility**: Works with existing grammars

### Tool Integration
- **BisonC++**: Automatic location propagation
- **Flex**: Location updates in lexer actions
- **Custom Lexers**: Manual location management

This location type template enables precise source code position tracking in BisonC++ parsers, supporting detailed error messages, debugging, and source code analysis tools.