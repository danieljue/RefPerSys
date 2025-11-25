# BisonC++ Location Type Data Template (`ltypedata.in`)

## Overview

The `ltypedata.in` file provides the data members and accessor functions for location tracking in BisonC++ parsers. It defines the location stack and provides functions to access location information during parsing.

## File Structure

### Location Data Members
```cpp
#pragma message "RefPerSys bisonc++ ltypedata.in"

LTYPE_   d_loc_;  // Current location (for compatibility)

// Location stack access function
LTYPE_ const &lsp_(int idx) const {
    return *(d_lsp + idx);
}
```

## Location Stack Architecture

### Stack Pointer Management
```cpp
// In bisonc++base.h, integrated with template
LTYPE_ *d_lsp = 0;  // Location stack pointer

// Parallel to state stack pointer d_vsp
// Points to current location on location stack
```

### Stack Access Pattern
```cpp
// Access locations relative to current position
LTYPE_ const &currentLoc = lsp_(0);   // Current location
LTYPE_ const &prevLoc = lsp_(-1);     // Previous location
LTYPE_ const &nextLoc = lsp_(1);      // Next location (if available)
```

## Integration with Parser Stack

### Parallel Stack Operations
```cpp
// State and location stacks maintained in parallel
std::vector<StatePair> d_stateStack;    // {state, semantic_value}
std::vector<LTYPE_> d_locationStack;    // Location information

// Stack pointers always synchronized
d_vsp = &d_stateStack[d_stackIdx];      // State pointer
d_lsp = &d_locationStack[d_stackIdx];   // Location pointer
```

### Stack Expansion
```cpp
// When stacks grow
size_t newSize = currentSize + STACK_EXPANSION_;
d_stateStack.resize(newSize);
d_locationStack.resize(newSize);  // Parallel expansion
$insert LTYPEresize  // Initialize new location entries
```

## Usage in Grammar Actions

### Location Access in Rules
```cpp
// In grammar rules
expression:
    expr '+' expr {
        $$ = $1 + $3;
        
        // Access locations of symbols
        $$.first_line = lsp_(-2).first_line;    // $1 location
        $$.first_column = lsp_(-2).first_column;
        $$.last_line = lsp_(0).last_line;       // $3 location
        $$.last_column = lsp_(0).last_column;
    }
```

### Location Propagation
```cpp
statement:
    expression ';' {
        // Statement spans entire rule
        $$.first_line = lsp_(-1).first_line;    // expression start
        $$.last_line = lsp_(0).last_line;       // semicolon end
    }
```

## Memory Management

### Location Object Lifetime
```cpp
// Locations live on parser stack
// Automatically managed with stack operations
// No manual memory management required
```

### Text Field Handling
```cpp
// If LTYPE_ contains text pointers
LTYPE_ loc = lsp_(0);
if (loc.text) {
    // Use token text
    processTokenText(loc.text);
    // Note: text may be freed when location is popped
}
```

## Error Reporting Integration

### Location-Aware Errors
```cpp
void MyParser::error() {
    LTYPE_ const &loc = lsp_(0);  // Location of current token
    
    std::cerr << "Parse error at line " << loc.first_line;
    if (loc.first_column > 0) {
        std::cerr << ", column " << loc.first_column;
    }
    if (loc.text) {
        std::cerr << " near '" << loc.text << "'";
    }
    std::cerr << std::endl;
}
```

### Stack Trace with Locations
```cpp
void errorVerbose_() {
    // Show state stack
    Base::errorVerbose_();
    
    // Show location stack
    std::cout << "\nLocation information:\n";
    for (size_t idx = 0; idx <= d_stackIdx_; ++idx) {
        LTYPE_ const &loc = d_locationStack[idx];
        std::cout << std::setw(2) << idx << ": line " 
                  << loc.first_line << std::endl;
    }
}
```

## Performance Characteristics

### Memory Usage
- **Stack Element**: Size of LTYPE_ per stack entry
- **Parallel Storage**: Location stack alongside state stack
- **No Extra Allocation**: Uses vector storage like state stack

### Access Speed
- **Direct Access**: Pointer arithmetic for location access
- **Cached Pointer**: `d_lsp` avoids repeated calculations
- **Inline Access**: `lsp_()` function is typically inlined

## Customization Options

### Extended Location Access
```cpp
// Additional location access functions
LTYPE_ const &currentLocation() const {
    return lsp_(0);
}

LTYPE_ const &locationAt(int depth) const {
    return lsp_(depth);
}

std::string formatLocation(LTYPE_ const &loc) const {
    std::ostringstream oss;
    oss << "line " << loc.first_line;
    if (loc.first_column > 0) {
        oss << ":" << loc.first_column;
    }
    return oss.str();
}
```

### Location Validation
```cpp
// Debug helper for location consistency
void validateLocations() const {
    for (size_t i = 0; i <= d_stackIdx_; ++i) {
        LTYPE_ const &loc = d_locationStack[i];
        if (loc.first_line < 0 || loc.last_line < loc.first_line) {
            std::cerr << "Invalid location at stack index " << i << std::endl;
        }
    }
}
```

## Compatibility Notes

### Optional Feature
- **Conditional Compilation**: Only included when locations are enabled
- **Zero Overhead**: When disabled, no location code is generated
- **Backward Compatibility**: Existing parsers work without modification

### Integration Requirements
- **Lexer Support**: Lexer must update location information
- **Grammar Actions**: May need location propagation in rules
- **Error Handlers**: Can access location information for reporting

This location data template provides efficient access to source code position information during parsing, enabling precise error reporting and debugging capabilities in BisonC++ parsers.