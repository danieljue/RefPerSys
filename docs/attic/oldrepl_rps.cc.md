# Old REPL Implementation (`attic/oldrepl_rps.cc`)

**File Path:** `attic/oldrepl_rps.cc`

## Overview

This file contains old, unused REPL (Read-Eval-Print Loop) implementation code that has been moved to the attic. It represents an earlier version of the RefPerSys interactive command interpreter, showcasing the evolution of the REPL system from basic tokenization to more sophisticated parsing.

## Historical Context

### Code Status
- **Status:** Obsolete/Archived
- **Purpose:** Historical reference for REPL development
- **Replacement:** Current REPL implementation in `repl_rps.cc`
- **Date Range:** 2019-2023

### Evolution Path
This file demonstrates the progression of RefPerSys's REPL from:
1. **Basic tokenization** (this file)
2. **Advanced parsing** (current `repl_rps.cc`)
3. **Integration with object system** (current implementation)

## Key Components

### Conditional Compilation
```cpp
#if 0 && oldcode
// All code is conditionally disabled
#endif /*0 && oldcode*/
```

**Purpose:** Code preservation without compilation

### Readline Integration
```cpp
#include "readline/readline.h"
#include "readline/history.h"
```

**Features:**
- Command-line editing
- History management
- Interactive input handling

## Core Functions

### Token Source Interpretation
```cpp
void rps_repl_interpret_token_source(Rps_CallFrame* callframe, Rps_TokenSource& toksource)
```

**Purpose:** Process tokens from a token source into REPL commands

**Algorithm:**
1. Create token deque for lexical tokens
2. Set up garbage collection marking
3. Loop through tokens until end
4. Parse command starting with object references
5. Execute commands (unimplemented in this version)

### Command Lexing
```cpp
Rps_TwoValues rps_repl_cmd_lexing(Rps_CallFrame* callframe, unsigned lookahead)
```

**Purpose:** Lexical analysis for REPL commands

**Features:**
- Lookahead token processing
- Integration with lexer functions
- Error handling and debugging

### Line Input Handling
```cpp
bool rps_repl_get_next_line(Rps_CallFrame* callframe, std::istream* inp,
                           const char* input_name, const char** plinebuf,
                           int* plineno, std::string prompt)
```

**Purpose:** Read next input line with readline support

**Capabilities:**
- File input processing
- Interactive readline input
- Terminal escape sequence handling
- History management

## Lexer Implementation

### Main Lexer Function
```cpp
Rps_TwoValues rps_repl_lexer(Rps_CallFrame* callframe, std::istream* inp,
                            const char* input_name, const char* linebuf,
                            int& lineno, int& colno)
```

**Purpose:** Tokenize input text into RefPerSys values

**Supported Token Types:**
1. **Numbers:** Integers and floating-point
2. **Infinity:** Special double values (±INF)
3. **Identifiers:** Object names and IDs
4. **Strings:** C++-style string literals
5. **Raw Strings:** Multi-line string literals
6. **Code Chunks:** Embedded code with objects
7. **Delimiters:** Punctuation-based tokens

### Number Parsing
```cpp
// Integer and float parsing with strtoll/strtod
long long l = strtoll(startnum, &endint, 0);
double d = strtod(startnum, &endfloat);
```

**Features:**
- Automatic type detection (int vs float)
- Error handling for malformed numbers

### Object Resolution
```cpp
_f.oblex = Rps_ObjectRef::find_object_or_fail_by_string(&_, namestr);
```

**Resolution Strategy:**
1. Look up existing objects by name/ID
2. Create new symbol objects for unknown names
3. Proper error handling for invalid identifiers

### String Literal Processing

#### Regular Strings
```cpp
std::string rps_lex_literal_string(const char* input_name, const char* linebuf,
                                  int lineno, int& colno)
```

**Escape Sequences Supported:**
- `\"` - Quote character
- `\\` - Backslash
- `\n`, `\r`, `\t` - Standard C escapes
- `\xHH` - Hexadecimal byte values
- `\uHHHH` - Unicode code points (4 hex digits)
- `\UHHHHHHHH` - Unicode code points (8 hex digits)

#### Raw Strings
```cpp
std::string rps_lex_raw_literal_string(Rps_CallFrame* callframe, std::istream* inp,
                                      const char* input_name, const char** plinebuf,
                                      int lineno, int& colno)
```

**Features:**
- Multi-line string literals
- Custom delimiters (up to 15 letters)
- C++11 raw string syntax: `R"delim(content)delim"`

### Code Chunk Processing
```cpp
// Macro strings mixing objects and text
// Syntax: #{object references and text}#
_f.chunkv = rps_lex_code_chunk(&_, inp, input_name, &linebuf, lineno, colno);
```

**GCC MELT Inspiration:**
- Embedded object references in text
- Complex macro processing
- Dynamic content generation

### Delimiter Recognition
```cpp
// Lookup in delimiter dictionary
_f.delimv = paylstrdict->find(delimstr);
```

**Features:**
- Configurable delimiter set
- Longest-match recognition
- Dictionary-based lookup

## Error Handling

### Lexical Errors
```cpp
RPS_WARNOUT("rps_repl_lexer " << input_name << " line " << lineno
            << ", column " << colno << " : bad name " << linebuf+colno);
```

**Error Types:**
- Invalid identifiers
- Malformed strings
- Unknown delimiters
- Unexpected characters

### Fatal Errors
```cpp
RPS_FATALOUT("unimplemented rps_repl_lexer inp@" << (void*)inp
             << " input_name=" << input_name << " ...");
```

**Unimplemented Features:**
- Many token types marked as unimplemented
- Placeholder for future extensions

## Integration Points

### Call Frame System
```cpp
RPS_LOCALFRAME(RPS_CALL_FRAME_UNDESCRIBED,
               /*callerframe:*/callframe,
               Rps_ObjectRef oblex; /* local variables */);
```

**Memory Management:**
- Proper call frame setup
- Garbage collection integration
- Local variable management

### Token Source Interface
```cpp
Rps_TokenSource& toksource
// Interface for token stream processing
```

**Abstraction:**
- Generic token source handling
- Position tracking
- Error reporting

### Readline Callbacks
```cpp
rps_readline_callframe = callframe;
// Global state for readline callbacks
```

**Threading Considerations:**
- Call frame preservation across readline calls
- Proper cleanup after input

## Limitations and Incomplete Features

### Marked Warnings
```cpp
#warning unimplemented rps_repl_interpret_token_source
#warning we need some condition on the lexing to stop it
#warning unimplemented rps_repl_lexer for delimiter
#warning incompletely unimplemented rps_repl_lexer
```

### Missing Functionality
1. **Command Interpretation:** Token processing incomplete
2. **Delimiter Handling:** Dictionary lookup not fully implemented
3. **Code Chunk Processing:** Complex macro expansion missing
4. **Error Recovery:** Limited error handling and recovery

### Commented-Out Code
Large sections of advanced tokenization code are commented out, indicating planned features that were never completed.

## Comparison with Current REPL

### Advancements Made
1. **Modular Design:** Separated lexer from interpreter
2. **Better Error Handling:** More robust error reporting
3. **Complete Implementation:** Working command processing
4. **Integration:** Better object system integration

### Preserved Concepts
1. **Token Types:** Same fundamental token categories
2. **String Processing:** Similar escape sequence handling
3. **Readline Integration:** Same interactive input approach

## File Status

**Status:** Historical Reference
**Significance:** Shows early REPL architecture and evolution
**Lessons Learned:** 
- Need for modular lexer design
- Importance of complete error handling
- Value of incremental development

## Future Relevance

This archived code serves as:
- **Historical Record:** Understanding system evolution
- **Design Reference:** Alternative implementation approaches
- **Code Reuse:** Potentially useful algorithms or patterns

The old REPL implementation demonstrates the iterative development process of RefPerSys, showing how complex interactive systems evolve from basic prototypes to production-ready implementations. While incomplete, it contains valuable insights into language design and interactive programming system development.