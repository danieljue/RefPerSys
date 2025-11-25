# BisonC++ Debug Includes Template (`debugincludes.in`)

## Overview

The `debugincludes.in` file provides the necessary header includes for BisonC++ parser debug functionality. It includes standard library headers required for debug output, string manipulation, and data structures used in debugging operations.

## File Structure

### Header Includes
```cpp
#pragma message "RefPerSys bisonc++ debugincludes.in"

#include <iostream>
#include <sstream>
#include <string>
#include <iomanip>
#include <unordered_map>
```

## Required Headers

### I/O and String Operations
```cpp
#include <iostream>      // std::cout, std::cerr for debug output
#include <sstream>       // std::ostringstream for debug buffering
#include <string>        // std::string for token names and formatting
```

### Formatting and Containers
```cpp
#include <iomanip>       // std::setw, std::setfill, std::hex for formatting
#include <unordered_map> // std::unordered_map for symbol table lookup
```

## Integration with Debug System

### Symbol Table Support
The `unordered_map` header enables the symbol table (`s_symbol`) used in `debugfunctions2.in` for converting token codes to names.

### Output Formatting
The `iomanip` header provides manipulators for formatted debug output, particularly in `debugfunctions2.in` for hexadecimal formatting of non-printable characters.

### Stream Operations
The `iostream` and `sstream` headers support the buffered debug output system implemented in `debugfunctions1.in`.

## Usage Context

### Template Integration
```cpp
// In bisonc++base.h
$insert debugincludes  // Expands to the header includes above

// Results in:
#include <iostream>
#include <sstream>
#include <string>
#include <iomanip>
#include <unordered_map>
```

### Compilation Requirements
These headers are required when debug functionality is enabled in generated parsers. They provide the standard library components needed for:

- Token name resolution
- Formatted debug output
- Buffered I/O operations
- Symbol table management

This minimal set of includes ensures debug functionality without pulling in unnecessary dependencies.