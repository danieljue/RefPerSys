# BisonC++ Macro Cleanup Template (`undef.in`)

## Overview

The `undef.in` file provides macro cleanup for BisonC++ parser templates. It ensures that parser-specific macros are properly undefined after template expansion to prevent naming conflicts and maintain clean compilation environments.

## File Structure

### Macro Cleanup
```cpp
#undef @

#pragma message "RefPerSys bisonc++ undef.in"

// CAVEAT: between the baseclass-include directive and the
// #undef directive in the previous line references to @
// are read as @Base.
// If you need to include additional headers in this file
// you should do so after these comment-lines.
```

## Macro Management

### Parser Name Macro
```cpp
#undef @  // Undefine the parser class name macro
```

**Purpose**: Clean up the `@` macro that represents the parser class name throughout the template system.

**Example Expansion**:
```cpp
// Before undef:
class @ { ... };        // @ becomes MyParser
@Base::lookup_()        // @ becomes MyParser

// After undef:
class MyParser { ... }; // Direct class name
MyParserBase::lookup_() // Direct base name
```

## Timing of Cleanup

### Strategic Placement
```cpp
// The undef occurs after baseclass-include but before user includes
$insert baseclass  // May still use @

// User includes and code can be added here
// @ is no longer available

$insert namespace-use  // User namespace directives
```

### CAVEAT Note
The comment explains the timing constraint:
- **Before undef**: `@` still expands to parser name
- **After undef**: `@` is undefined, direct names must be used
- **User code**: Must use actual class names, not macros

## Usage Context

### Template Expansion Order
```cpp
// In generated parser file
#include "parser.h"     // @ still defined here

// Base class includes
$insert baseclass      // Uses @

// Macro cleanup
$insert undef          // Undefines @

// User includes (after undef)
#include "myheaders.h"  // Must use real names
```

### Namespace Management
```cpp
$insert namespace-open   // Uses @
$insert polymorphicCode  // Uses @

// Cleanup
$insert undef           // Undefines @

// User namespace usage
$insert namespace-use   // Must use real names
```

## Error Prevention

### Naming Conflicts
```cpp
// Without undef, @ could conflict with:
// - User-defined macros
// - System headers
// - Other parser instances
// - Template metaprogramming

// With proper undef:
// - Clean namespace
// - No macro leakage
// - Predictable compilation
```

### Multiple Parser Support
```cpp
// Multiple parsers in same program
// Each gets its own macro during generation
// Proper undef prevents cross-contamination

// Parser1 generation: @ = Parser1
// Parser2 generation: @ = Parser2
// Both properly undefined after use
```

## Best Practices

### User Code Placement
```cpp
// Place user includes after undef
$insert undef

// Now safe to include user headers
#include "my_custom_parser_utils.h"

// User can define their own macros without conflict
#define MY_PARSER_DEBUG 1
```

### Macro Hygiene
```cpp
// Avoid redefining @ in user code
// Use the actual class names instead

class MyParser : public MyParserBase {
    // Correct: use actual names
    void myFunction(MyParserBase &base) { ... }
    
    // Incorrect: would not work after undef
    // void myFunction(@Base &base) { ... }
};
```

## Debugging Macro Issues

### Common Problems
```cpp
// Error: @ was not declared in this scope
// Cause: Using @ after undef
// Fix: Use actual class name

// Error: macro redefinition
// Cause: @ conflicts with existing macro
// Fix: Ensure proper undef timing
```

### Verification
```cpp
// Check macro state
#ifdef @
#pragma message "Macro @ is still defined"
#else
#pragma message "Macro @ is properly undefined"
#endif
```

## Integration with Build System

### Compilation Units
- **Template Expansion**: `@` defined during BisonC++ code generation
- **Header Generation**: `@` used in .h file creation
- **Implementation**: `@` used in .cc file creation
- **Cleanup**: `@` undefined at end of each generated file

### Multi-File Projects
```cpp
// Each generated file is independent
// parser.h: defines @, uses it, undefines it
// parser.cc: defines @, uses it, undefines it
// No macro leakage between files
```

## Performance Impact

### Compilation Speed
- **Zero Runtime Cost**: Macro expansion at compile time
- **Clean Preprocessing**: No macro side effects
- **Predictable Expansion**: Consistent behavior

### Code Size
- **No Object Code**: Purely compile-time feature
- **Minimal Source**: Single undef directive
- **Clean Output**: No macro artifacts in generated code

This macro cleanup template ensures proper namespace hygiene in BisonC++ generated parsers, preventing macro conflicts and maintaining clean compilation environments for complex parser projects.