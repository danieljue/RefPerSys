# BisonC++ Debug Functions Implementation 3 (`debugfunctions3.in`)

## Overview

The `debugfunctions3.in` file provides the `errorVerbose_` function implementation for BisonC++ parsers. This function generates detailed error information during parsing, showing the current state of the parser stack when errors occur.

## File Structure

### Error Verbose Implementation
```cpp
#pragma message "RefPerSys bisonc++ debugfunctions3.in"

void @Base::errorVerbose_() {
    std::cout << "Parser State stack containing " << (d_stackIdx_ + 1) <<
                                                                 "elements:\n"
              "Each line shows a stack index followed "
                                     "by the value of that stack element\n";
    for (size_t idx = d_stackIdx_ + 1; idx--; )
       std::cout << std::setw(2) << idx << ": " <<
                    std::setw(3) << d_stateStack_[idx] << '\n';
}
```

## Error State Inspection

### Stack Content Display
```cpp
void errorVerbose_() {
    // Header message
    std::cout << "Parser State stack containing " << (d_stackIdx_ + 1) 
              << " elements:\n"
              << "Each line shows a stack index followed "
              << "by the value of that stack element\n";
    
    // Stack dump from top to bottom
    for (size_t idx = d_stackIdx_ + 1; idx--; ) {
        std::cout << std::setw(2) << idx << ": " 
                  << std::setw(3) << d_stateStack_[idx] << '\n';
    }
}
```

**Output Format**:
```
Parser State stack containing 5 elements:
Each line shows a stack index followed by the value of that stack element
 4:  12
 3:   8
 2:  15
 1:   3
 0:   0
```

### Stack Indexing
- **Index 0**: Bottom of stack (initial state 0)
- **Highest Index**: Top of stack (current state)
- **State Values**: Parser state numbers from state machine

## Integration with Error Recovery

### Error Recovery Context
```cpp
// In error recovery (from bisonc++.cc)
if (d_acceptedTokens_ >= d_requiredTokens_) {
    ++d_nErrors_;
    error();  // User error handler
    errorVerbose_();  // Detailed stack dump
}
```

### Error Handler Integration
```cpp
// User can call errorVerbose_() in custom error handlers
void MyParser::error() {
    std::cerr << "Syntax error at line " << lineNumber << std::endl;
    
    // Show detailed parser state
    if (debugEnabled) {
        errorVerbose_();
    }
}
```

## Stack State Interpretation

### State Machine Context
The stack shows the sequence of states that led to the current parsing position:

```cpp
// Example stack interpretation
// Stack: [0, 3, 8, 12, 15]
// Meaning:
// - Started in state 0 (initial)
// - Shifted to state 3 after first token
// - Shifted to state 8 after second token  
// - Shifted to state 12 after third token
// - Currently in state 15, expecting next token
```

### Error Location Identification
```cpp
// The top state (highest index) indicates where error occurred
size_t errorState = d_stateStack_[d_stackIdx_];
std::cout << "Error occurred in state " << errorState << std::endl;

// Can map to grammar positions or expected tokens
```

## Performance Considerations

### Output Overhead
- **I/O Intensive**: Direct console output for each stack element
- **Conditional Use**: Only called during error recovery
- **Debug Only**: Typically disabled in production builds

### Memory Access
- **Stack Traversal**: Iterates through entire stack
- **Read-Only**: No modifications to parser state
- **Safe Access**: Uses validated stack indices

## Customization Options

### Enhanced Error Reporting
Users can extend `errorVerbose_()` for more detailed information:

```cpp
void MyParserBase::errorVerbose_() {
    // Call base implementation first
    Base::errorVerbose_();
    
    // Add custom information
    std::cout << "\nAdditional error context:\n";
    std::cout << "Current token: " << symbol_(token_()) << std::endl;
    std::cout << "Line number: " << currentLine << std::endl;
    std::cout << "Expected tokens: ";
    
    // Show expected tokens for current state
    auto expected = getExpectedTokens(state_());
    for (int token : expected) {
        std::cout << symbol_(token) << " ";
    }
    std::cout << std::endl;
}
```

### Alternative Output Formats
```cpp
void MyParserBase::errorVerbose_() {
    // JSON format for programmatic processing
    std::cout << "{\"stack\": [";
    for (size_t idx = 0; idx <= d_stackIdx_; ++idx) {
        if (idx > 0) std::cout << ",";
        std::cout << d_stateStack_[idx];
    }
    std::cout << "]}" << std::endl;
}
```

### File Output
```cpp
void MyParserBase::errorVerbose_() {
    // Redirect to error log file
    static std::ofstream errorLog("parser_errors.log", std::ios::app);
    
    errorLog << "Parser State stack containing " << (d_stackIdx_ + 1) 
             << " elements:" << std::endl;
    
    for (size_t idx = d_stackIdx_ + 1; idx--; ) {
        errorLog << std::setw(2) << idx << ": " 
                 << std::setw(3) << d_stateStack_[idx] << std::endl;
    }
    
    errorLog << std::endl;
}
```

## Debugging Complex Errors

### Stack Trace Analysis
```cpp
// Helper function to analyze stack patterns
void analyzeStack() {
    std::cout << "Stack analysis:\n";
    
    // Check for common error patterns
    if (d_stackIdx_ >= 2) {
        int top3[3] = {
            d_stateStack_[d_stackIdx_],
            d_stateStack_[d_stackIdx_ - 1], 
            d_stateStack_[d_stackIdx_ - 2]
        };
        
        // Pattern recognition
        if (top3[0] == 15 && top3[1] == 12 && top3[2] == 8) {
            std::cout << "Likely missing semicolon in statement\n";
        }
    }
}
```

### State Machine Integration
```cpp
// Map states to grammar rules for better error messages
std::string getStateDescription(int state) {
    switch (state) {
        case 0: return "initial state";
        case 3: return "after identifier";
        case 8: return "in expression";
        case 12: return "after operator";
        case 15: return "expecting operand";
        default: return "unknown state";
    }
}

void MyParserBase::errorVerbose_() {
    Base::errorVerbose_();
    
    std::cout << "\nState descriptions:\n";
    for (size_t idx = 0; idx <= d_stackIdx_; ++idx) {
        std::cout << std::setw(2) << idx << ": " 
                  << std::setw(3) << d_stateStack_[idx] << " - "
                  << getStateDescription(d_stateStack_[idx]) << std::endl;
    }
}
```

## Error Recovery Integration

### Recovery State Tracking
```cpp
// Show recovery progress
void MyParserBase::errorVerbose_() {
    Base::errorVerbose_();
    
    if (d_recovery) {
        std::cout << "Currently in error recovery mode\n";
        std::cout << "Accepted tokens: " << d_acceptedTokens_ << std::endl;
        std::cout << "Required tokens: " << d_requiredTokens_ << std::endl;
    }
}
```

## Thread Safety

### Output Synchronization
For multi-threaded parsers, add synchronization:

```cpp
void MyParserBase::errorVerbose_() {
    static std::mutex errorMutex;
    std::lock_guard<std::mutex> lock(errorMutex);
    
    // Original implementation
    std::cout << "Parser State stack containing " << (d_stackIdx_ + 1) 
              << " elements:\n"
              << "Each line shows a stack index followed "
              << "by the value of that stack element\n";
              
    for (size_t idx = d_stackIdx_ + 1; idx--; ) {
        std::cout << std::setw(2) << idx << ": " 
                  << std::setw(3) << d_stateStack_[idx] << '\n';
    }
}
```

## Compatibility Notes

### Stack Representation
- **d_stackIdx_**: Top of stack index (0-based)
- **d_stateStack_**: Array of state values
- **Stack Size**: `d_stackIdx_ + 1` elements

### Version Compatibility
- **BisonC++ 6.x**: Uses `d_stackIdx_` and `d_stateStack_`
- **Older Versions**: May use different member names
- **Future Versions**: Interface may change

This error verbose implementation provides crucial debugging information during parser errors, showing the complete state stack to help developers understand how the parser reached an error condition and what recovery actions may be appropriate.