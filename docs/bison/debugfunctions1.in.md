# BisonC++ Debug Functions Implementation 1 (`debugfunctions1.in`)

## Overview

The `debugfunctions1.in` file provides the core debug output implementation for BisonC++ parsers. It contains the `dflush_` function that manages debug output buffering and the `stype_` function that formats semantic values for display.

## File Structure

### Debug Output Stream Management
```cpp
#pragma message "RefPerSys bisonc++ debugfunctions1.in"

std::ostringstream @Base::s_out_;  // Static stream definition

std::ostream &@Base::dflush_(std::ostream &out) {
    std::ostringstream &s_out_ = dynamic_cast<std::ostringstream &>(out);
    
    std::cout << "    " << s_out_.str() << std::flush;
    s_out_.clear();
    s_out_.str("");
    return out;
}
```

### Semantic Value Formatting
```cpp
std::string @Base::stype_(char const *pre, 
                         STYPE_ const &semVal, char const *post) const {
@insert-stype
    using namespace std;
    ostringstream ostr;
    ostr << pre << semVal << post;
    return ostr.str();
@else
    return "";
@end
}
```

## Debug Output Management

### Buffered Output System
```cpp
std::ostream &dflush_(std::ostream &out) {
    // Cast to ostringstream reference (should be s_out_)
    std::ostringstream &s_out_ = dynamic_cast<std::ostringstream &>(out);
    
    // Output with indentation
    std::cout << "    " << s_out_.str() << std::flush;
    
    // Clear the stream for reuse
    s_out_.clear();
    s_out_.str("");
    
    return out;  // Return original stream reference
}
```

**Design Rationale**:
- **Buffering**: Accumulates debug messages to reduce I/O overhead
- **Indentation**: Adds 4-space indent for readability
- **Reuse**: Clears stream for next debug operation
- **Type Safety**: Uses dynamic_cast for safety (though s_out_ is always passed)

### Usage Pattern
```cpp
// Accumulate debug information
s_out_ << "State transition: " << oldState << " -> " << newState << std::endl;
s_out_ << "Token: " << symbol_(token_()) << std::endl;

// Flush to console
dflush_(s_out_);
```

## Semantic Value Formatting

### Template-Based Formatting
The `stype_` function uses template expansion to handle different semantic value types:

#### For Union-Based Semantic Values
```cpp
@insert-stype
    using namespace std;
    ostringstream ostr;
    ostr << pre << semVal << post;
    return ostr.str();
@else
    return "";
@end
```

**Direct Output**: Uses operator<< for STYPE_ directly.

#### For Polymorphic Semantic Values
```cpp
// When polymorphic values are enabled
@insert-stype
    // Type-aware formatting would be inserted here
    using namespace std;
    ostringstream ostr;
    ostr << pre << semVal << post;  // Type-aware output
    return ostr.str();
@else
    return "";
@end
```

**Type-Aware Output**: Polymorphic values have custom output operators.

### Formatting Examples
```cpp
// Simple semantic value
std::string debug = stype_("Value: ", vs_(0));
// Output: "Value: 42" (for integer value)

// With polymorphic values
SType polyVal;
polyVal.assign<Tag_::INT>(42);
std::string debug = stype_("Poly value: ", polyVal);
// Output: "Poly value: INT:42" (type-aware formatting)
```

## Integration with Parser Operations

### State Transition Debugging
```cpp
// In shift operations
if (d_debug_) {
    s_out_ << "SHIFT: state " << state_() << " <- "
           << stype_("token ", d_token, "") << " -> state " << action
           << std::endl;
    dflush_(s_out_);
}
```

### Reduction Debugging
```cpp
// In reduce operations
if (d_debug_) {
    s_out_ << "REDUCE: rule " << ruleNr << ", popping " << pi.d_size
           << " symbols" << stype_(", result: ", vs_(0)) << std::endl;
    dflush_(s_out_);
}
```

### Error Recovery Debugging
```cpp
// During error recovery
if (d_debug_) {
    s_out_ << "ERROR RECOVERY: popping to state " << state_()
           << ", inserting error token" << std::endl;
    dflush_(s_out_);
}
```

## Performance Considerations

### Output Buffering Strategy
- **Accumulate First**: Build complete messages in memory
- **Batch Output**: Single cout operation per debug point
- **Stream Reuse**: Clear and reuse s_out_ for next operation
- **Indentation**: Fixed 4-space indent for visual hierarchy

### Memory Management
- **Static Stream**: s_out_ persists for program lifetime
- **Automatic Growth**: ostringstream grows as needed
- **Clear on Flush**: Prevents memory accumulation
- **No Memory Leaks**: Stream cleared before reuse

### CPU Overhead
- **Conditional Execution**: Debug code only runs when d_debug_ is true
- **String Operations**: Relatively fast in modern C++
- **I/O Bottleneck**: cout flushing is the main cost
- **Minimal Formatting**: Simple string concatenation

## Customization Options

### Output Redirection
Users can modify `dflush_` to redirect debug output:

```cpp
std::ostream &MyParserBase::dflush_(std::ostream &out) {
    // Redirect to file
    static std::ofstream debugLog("parser_debug.log", std::ios::app);
    std::ostringstream &s_out_ = dynamic_cast<std::ostringstream &>(out);
    
    debugLog << s_out_.str() << std::flush;
    s_out_.clear();
    s_out_.str("");
    
    return out;
}
```

### Custom Semantic Formatting
For complex semantic types, users can override `stype_`:

```cpp
std::string MyParserBase::stype_(char const *pre, STYPE_ const &semVal,
                                 char const *post) const {
    std::ostringstream ostr;
    ostr << pre;
    
    // Custom formatting based on semantic value type
    if (semVal.is_ast_node()) {
        ostr << "AST:" << semVal.as_ast_node()->type_name();
    } else if (semVal.is_symbol()) {
        ostr << "SYM:" << semVal.as_symbol()->name();
    } else {
        ostr << "UNKNOWN:" << semVal.raw_value();
    }
    
    ostr << post;
    return ostr.str();
}
```

## Error Handling

### Stream Operation Safety
```cpp
try {
    s_out_ << debugMessage;
    dflush_(s_out_);
} catch (std::exception const &e) {
    // Handle debug output failures gracefully
    std::cerr << "Debug output failed: " << e.what() << std::endl;
}
```

### Type Casting Safety
```cpp
std::ostream &dflush_(std::ostream &out) {
    try {
        std::ostringstream &s_out_ = dynamic_cast<std::ostringstream &>(out);
        // ... use s_out_
    } catch (std::bad_cast const &) {
        // Fallback if wrong stream type passed
        std::cout << "Debug: bad stream type in dflush_" << std::endl;
    }
    return out;
}
```

## Thread Safety Considerations

### Single Static Stream
- **Not Thread-Safe**: s_out_ is shared static resource
- **Single-Threaded Assumption**: BisonC++ parsers typically single-threaded
- **MT Parser Issues**: Would need mutex protection for multi-threaded parsers

### Debug Output Synchronization
```cpp
// For multi-threaded parsers, add synchronization
std::ostream &dflush_(std::ostream &out) {
    static std::mutex debugMutex;
    std::lock_guard<std::mutex> lock(debugMutex);
    
    std::ostringstream &s_out_ = dynamic_cast<std::ostringstream &>(out);
    std::cout << "    " << s_out_.str() << std::flush;
    s_out_.clear();
    s_out_.str("");
    return out;
}
```

## Compatibility with Different Semantic Types

### Union-Based Semantic Values
- **Direct Output**: Uses built-in operator<< for STYPE_
- **Simple Formatting**: Pre/post strings for context
- **Type-Unaware**: No runtime type information

### Polymorphic Semantic Values
- **Type-Aware Output**: Custom operator<< implementations
- **Rich Formatting**: Can show type names and values
- **Runtime Type Info**: Uses tag() method for type identification

### Custom Semantic Types
- **User-Defined Output**: Can implement custom operator<<
- **Complex Formatting**: Support for nested structures
- **Debug-Friendly**: Can include structural information

This debug functions implementation provides efficient, flexible debug output capabilities for BisonC++ parsers, supporting both simple and complex semantic value types with minimal performance overhead when debugging is disabled.