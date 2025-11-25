# Lexical Analyzer Implementation Analysis (`lexer_rps.cc`)

## Overview

The `lexer_rps.cc` file implements the **lexical analyzer (lexer)** for the Read-Eval-Print Loop (REPL) system in the Reflective Persistent System (RefPerSys), providing tokenization of input text into lexical tokens that can be processed by the parser. This file serves as the foundation for RefPerSys's interactive programming interface, supporting multiple input sources and complex token types including objects, strings, numbers, and code chunks.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2019-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// Unicode support
#include "unictype.h"
#include "uniconv.h"

// System libraries
#include <termios.h>
#include <wordexp.h>

// Git versioning information
extern "C" const char rps_lexer_gitid[];
extern "C" const char rps_lexer_date[];
extern "C" const char rps_lexer_shortgitid[];
extern "C" const char rps_lexer_timestamp[];

// Global token name string value
extern "C" Rps_StringValue rps_lexer_token_name_str_val;
```

### Key Dependencies
- **libunistring**: Unicode character classification and conversion
- **RefPerSys core**: Object system, value types, and call frames
- **Standard I/O**: File streams and terminal I/O

## Rps_TokenSource Base Class

The `Rps_TokenSource` class is the foundation for all token sources, providing common functionality for tokenization and position tracking.

### Class Structure
```cpp
class Rps_TokenSource {
protected:
    std::string toksrc_name;           // Source identifier
    int toksrc_line;                   // Current line number
    int toksrc_col;                    // Current column number
    int toksrc_counter;                // Token counter
    unsigned toksrc_number;            // Unique source number
    std::string toksrc_linebuf;        // Current line buffer
    std::deque<Rps_Value> toksrc_token_deq; // Token queue
    Rps_Value toksrc_ptrnameval;       // Cached name value
    rps_keyword_lexing_sigt* toksrc_keywfun; // Keyword lexing function

public:
    // Core methods
    virtual bool get_line(void) = 0;   // Read next line
    virtual bool reached_end(void) const = 0; // Check for end of input
    virtual void display(std::ostream& out) const; // Display source state

    // Token operations
    Rps_LexTokenValue get_token(Rps_CallFrame* callframe);
    Rps_Value lookahead_token(Rps_CallFrame* callframe, unsigned rank);
    void consume_front_token(Rps_CallFrame* callframe, bool* psuccess = nullptr);
    void append_back_new_token(Rps_CallFrame* callframe, Rps_Value tokenv);

    // Position and state
    const std::string& name() const;
    int line() const;
    int col() const;
    int token_count() const;
    const char* curcptr() const;
    const std::string& current_line() const;
    std::string position_str(int col = -1) const;
    void display_current_line_with_cursor(std::ostream& out) const;

    // Lifecycle
    void restart_token_source(void);
    void clear_token_dequeue(void);
    Rps_Value source_name_val(Rps_CallFrame* callframe);

    // Garbage collection
    void gc_mark(Rps_GarbageCollector& gc, unsigned depth);
    void really_gc_mark(Rps_GarbageCollector& gc, unsigned depth);

    // Construction/Destruction
    Rps_TokenSource(std::string name);
    virtual ~Rps_TokenSource();

    // Keyword lexing
    void set_keyword_lexing_fun(rps_keyword_lexing_sigt* fun);
};
```

### Token Management
- **Token Queue**: `toksrc_token_deq` stores lexed tokens for lookahead
- **Position Tracking**: Line/column counters for error reporting
- **Source Naming**: Unique identification and memoized name values
- **Keyword Support**: Pluggable keyword lexing functions

## Token Source Implementations

### Rps_StreamTokenSource Class
Reads tokens from file streams with word expansion support.

```cpp
class Rps_StreamTokenSource : public Rps_TokenSource {
    std::ifstream toksrc_input_stream;

public:
    Rps_StreamTokenSource(std::string path);
    virtual bool get_line(void);
    virtual bool reached_end(void) const;
    virtual void display(std::ostream& out) const;
    virtual ~Rps_StreamTokenSource();
};
```

**Features**:
- **Word Expansion**: Uses `wordexp()` for shell-style path expansion
- **Error Handling**: Validates file existence and readability
- **Stream Management**: Proper file stream lifecycle

### Rps_CinTokenSource Class
Reads tokens from standard input (stdin).

```cpp
class Rps_CinTokenSource : public Rps_TokenSource {
public:
    Rps_CinTokenSource();
    virtual bool get_line(void);
    virtual bool reached_end(void) const;
    virtual void display(std::ostream& out) const;
    virtual ~Rps_CinTokenSource();
};
```

**Features**:
- **Interactive Input**: Reads from `std::cin`
- **Simple Name**: Uses "-" as source identifier
- **EOF Detection**: Checks `std::cin.eof()`

### Rps_StringTokenSource Class
Reads tokens from string input with restart capability.

```cpp
class Rps_StringTokenSource : public Rps_TokenSource {
    std::istringstream toksrcstr_inp;
    std::string toksrcstr_str;

public:
    Rps_StringTokenSource(std::string inptstr, std::string name);
    void restart_string_token_source(void);
    virtual bool get_line(void);
    virtual bool reached_end(void) const;
    virtual void output(std::ostream& out, unsigned depth, unsigned maxdepth) const;
    virtual void display(std::ostream& out) const;
    virtual ~Rps_StringTokenSource();
};
```

**Features**:
- **String Stream**: Uses `std::istringstream` for input
- **Restart Support**: Can reset to beginning of string
- **Content Display**: Shows abbreviated string content

### Rps_MemoryFileTokenSource Class
Reads tokens from memory-mapped files for efficient large file handling.

```cpp
class Rps_MemoryFileTokenSource : public Rps_TokenSource {
    std::string toksrcmfil_path;
    char* toksrcmfil_start;
    char* toksrcmfil_line;
    char* toksrcmfil_end;
    char* toksrcmfil_nextpage;
    int toksrcmfil_fd;

public:
    Rps_MemoryFileTokenSource(const std::string path);
    virtual bool get_line(void);
    virtual bool reached_end(void) const;
    virtual void output(std::ostream& out, unsigned depth, unsigned maxdepth) const;
    virtual void display(std::ostream& out) const;
    virtual ~Rps_MemoryFileTokenSource();
};
```

**Features**:
- **Memory Mapping**: Uses `mmap()` for efficient file access
- **Page Alignment**: Handles page boundaries correctly
- **Huge Page Support**: Uses `MAP_HUGETLB` for large files
- **Resource Cleanup**: Proper `munmap()` and file descriptor management

## Token Lexing Methods

### Number Token Lexing
```cpp
Rps_LexTokenValue get__number__token(Rps_CallFrame* callframe, const char* curp)
```
**Purpose**: Lexes numeric literals (integers and floats)
- **Integer Parsing**: Uses `strtoll()` for integer conversion
- **Float Parsing**: Uses `strtod()` for floating-point conversion
- **Infinity Support**: Recognizes `+INF` and `-INF` patterns
- **Tagged Integers**: Uses `Rps_Value::Rps_IntTag{}` for small integers

### String Token Lexing

#### Quoted String Literals
```cpp
std::string lex_quoted_literal_string(Rps_CallFrame* callframe)
```
**Purpose**: Lexes C++-style quoted string literals
- **Escape Sequences**: Supports `\n`, `\t`, `\r`, `\"`, `\\`, etc.
- **Hex Escapes**: `\xHH` for byte values
- **Unicode Escapes**: `\uHHHH` and `\UHHHHHHHH` for Unicode codepoints
- **UTF-8 Output**: Converts Unicode to UTF-8 encoding

#### Raw String Literals
```cpp
std::string lex_raw_literal_string(Rps_CallFrame* callframe)
```
**Purpose**: Lexes C++-style raw string literals
- **Delimiter Support**: `R"delim(raw content)delim"`
- **Multi-line**: Preserves line breaks and whitespace
- **Delimiter Length**: Up to 15 alphabetic characters
- **Status**: Implementation incomplete (marked with warning)

### Object and Symbol Token Lexing
```cpp
Rps_LexTokenValue get__namoid__token(Rps_CallFrame* callframe, const char* curp)
```
**Purpose**: Lexes identifiers that may refer to RefPerSys objects or symbols
- **Object Lookup**: Uses `Rps_ObjectRef::find_object_or_null_by_string()`
- **Symbol Creation**: Creates new symbols for unknown identifiers
- **Keyword Support**: Integrates with keyword lexing functions
- **Context Awareness**: Considers preceding `@` for keyword detection

### Delimiter Token Lexing
```cpp
Rps_Value get_delimiter(Rps_CallFrame* callframe)
```
**Purpose**: Lexes punctuation sequences as delimiters
- **Dictionary Lookup**: Uses `repl_delim` string dictionary
- **Longest Match**: Finds longest matching delimiter sequence
- **UTF-8 Support**: Handles Unicode punctuation characters
- **Fallback Truncation**: Reduces delimiter length if not found

### Code Chunk Token Lexing
```cpp
Rps_Value lex_code_chunk(Rps_CallFrame* callframe)
```
**Purpose**: Lexes code chunks with embedded objects and meta-variables
- **Syntax**: `#{content}#` or `#{letter}{content}{letter}#`
- **Meta-variables**: `$name` for object references
- **Embedded Objects**: Direct object references in code
- **Vector Storage**: Uses `Rps_PayloadVectVal` for chunk elements

#### Code Chunk Elements
```cpp
Rps_Value lex_chunk_element(Rps_CallFrame* callframe, Rps_ObjectRef obchk, Rps_ChunkData_st* chkdata)
```
**Purpose**: Lexes individual elements within code chunks
- **Object Names**: C-style identifiers resolving to objects
- **Numbers**: Integer literals
- **Spaces**: Sequences of whitespace characters
- **Meta-variables**: `$name` and `$$` (escaped dollar)
- **Plain Strings**: Sequences of non-special characters
- **Status**: Implementation incomplete (marked with warnings)

## UTF-8 and Unicode Support

### Character Classification
- **Unicode Categories**: Uses `uc_is_punct()` for punctuation detection
- **UTF-8 Length**: Uses `u8_mblen()` and `u8_strmbtouc()` for character processing
- **Safe Truncation**: Uses `u8_prev()` for proper UTF-8 boundary handling

### String Processing
- **UTF-8 Validation**: Checks for valid UTF-8 encoding
- **Unicode Conversion**: Uses `u8_uctomb()` for Unicode to UTF-8 conversion
- **Length Calculation**: Proper byte vs character counting

## Token Lookahead and Consumption

### Lookahead Functionality
```cpp
Rps_Value lookahead_token(Rps_CallFrame* callframe, unsigned rank)
```
**Purpose**: Examines upcoming tokens without consuming them
- **Queue Management**: Maintains token queue for lookahead
- **Dynamic Filling**: Calls `get_token()` to fill queue as needed
- **Rank-based Access**: Access tokens by position in lookahead buffer

### Token Consumption
```cpp
void consume_front_token(Rps_CallFrame* callframe, bool* psuccess = nullptr)
```
**Purpose**: Removes and discards the front token from the queue
- **Queue Management**: Pops front element from token deque
- **Success Reporting**: Optional success flag for error handling
- **Thread Safety**: Main thread only operation

### Token Insertion
```cpp
void append_back_new_token(Rps_CallFrame* callframe, Rps_Value tokenv)
```
**Purpose**: Adds tokens to the end of the token queue
- **Parser Integration**: Allows parser to inject synthetic tokens
- **Queue Extension**: Appends to existing token stream

## Integration with RefPerSys Object System

### Object Resolution
- **Name Lookup**: Converts identifier strings to object references
- **Symbol Creation**: Generates new symbols for unknown names
- **Type Classification**: Distinguishes objects, symbols, and keywords

### Token Creation
```cpp
const Rps_LexTokenZone* make_token(Rps_CallFrame* callframe,
                                   Rps_ObjectRef lexkindarg,
                                   Rps_Value lexvalarg,
                                   const Rps_String* sourcev)
```
**Purpose**: Creates lexical token objects with proper metadata
- **Zone Allocation**: Uses `rps_make_lex_token_zone()` for token creation
- **Source Attribution**: Links tokens to their source location
- **Type Information**: Associates tokens with their semantic types

### Garbage Collection Integration
```cpp
void gc_mark(Rps_GarbageCollector& gc, unsigned depth)
void really_gc_mark(Rps_GarbageCollector& gc, unsigned depth)
```
**Purpose**: Marks token-related objects during garbage collection
- **Token Queue**: Marks all queued token values
- **Name Values**: Marks cached source name strings
- **Depth Limiting**: Prevents infinite recursion in GC marking

## Testing and Debugging Support

### Test Function
```cpp
void rps_run_test_repl_lexer(const std::string& teststr)
```
**Purpose**: Tests lexer functionality with string input
- **String Source**: Creates `Rps_StringTokenSource` for testing
- **Token Enumeration**: Lexes all tokens from input string
- **Progress Reporting**: Shows tokenization progress and results
- **Performance Timing**: Uses `RPS_TIMER_START/STOP` for measurement

### Debug Output
- **Position Display**: Shows current line and column with cursor
- **Token Visualization**: Displays token values and types
- **Source State**: Shows token queue and current position
- **Backtrace Integration**: Includes call stack information

## Implementation Status and TODOs

### Completed Features
- ✅ **Token Source Hierarchy**: Complete class hierarchy for different input types
- ✅ **Basic Token Types**: Numbers, strings, objects, symbols, delimiters
- ✅ **UTF-8 Support**: Unicode-aware character processing
- ✅ **Memory Mapping**: Efficient file reading for large inputs
- ✅ **Token Queue Management**: Lookahead and consumption functionality
- ✅ **Object Integration**: RefPerSys object and symbol resolution
- ✅ **Garbage Collection**: Proper marking of token-related objects
- ✅ **Error Handling**: Comprehensive error reporting and recovery
- ✅ **Debug Support**: Extensive logging and visualization

### Incomplete Implementations
- ❌ **Raw String Literals**: `lex_raw_literal_string()` incomplete
- ❌ **Code Chunk Elements**: `lex_chunk_element()` incomplete
- ❌ **Delimiter Parsing**: Some delimiter types not handled
- ❌ **Keyword Integration**: Full keyword lexing not implemented
- ❌ **Memory File Output**: `output()` method incomplete
- ❌ **Code Chunk Meta-variables**: Full meta-variable support missing

### Known Warnings and TODOs
- **Line 155**: TODO about using source string in token creation
- **Line 807**: TODO about using `startswithalpha` and `afterat` in namoid lexing
- **Line 1276**: Warning about incomplete `get_token()` implementation
- **Line 1597**: Warning about moving code from `repl_rps.cc`
- **Line 1916**: Warning about parsing delimiters in code chunks
- **Line 1919**: Warning about unimplemented chunk element parsing
- **Line 1920**: TODO about moving code from `repl_rps.cc`

### Future Enhancements
- Complete raw string literal support
- Full code chunk meta-variable implementation
- Enhanced delimiter recognition
- Improved keyword lexing integration
- Better error recovery and diagnostics
- Performance optimizations for large inputs
- Support for additional token types
- Integration with external lexers

## Usage Patterns

### Basic Tokenization
```cpp
// Create a string token source
Rps_StringTokenSource source("hello world 123", "test");

// Set up keyword lexing (optional)
source.set_keyword_lexing_fun(my_keyword_lexer);

// Get tokens
Rps_LexTokenValue token;
while ((token = source.get_token(callframe))) {
    std::cout << "Token: " << token << std::endl;
}
```

### Lookahead Usage
```cpp
// Look ahead at upcoming tokens
Rps_Value next_token = source.lookahead_token(callframe, 0);  // Next token
Rps_Value third_token = source.lookahead_token(callframe, 2); // Two ahead

// Consume tokens
source.consume_front_token(callframe);
```

### Object Resolution
```cpp
// Lexing "my_object" will create a token with:
// - lexkindob: object∈class
// - lextokv: reference to the object if it exists, or null
// - source location information
```

### Code Chunk Example
```cpp
// Input: #{hello $name world}#
Rps_Value chunk = source.lex_code_chunk(callframe);
// Creates a code_chunk object containing:
// - String "hello "
// - Meta-variable reference to object "name"
// - String " world"
```

### Testing
```cpp
// Test lexer with string input
rps_run_test_repl_lexer("hello 123 \"string\" my_object");
// Outputs all lexed tokens with positions
```

## Design Rationale

### Token Source Hierarchy
- **Polymorphism**: Common interface for different input sources
- **Efficiency**: Memory mapping for large files, streaming for others
- **Extensibility**: Easy to add new input source types

### Token Queue Architecture
- **Lookahead Support**: Enables LL(k) parsing capabilities
- **Backtracking**: Allows parser to examine upcoming tokens
- **Memory Management**: Deque provides efficient front operations

### UTF-8 First Approach
- **Unicode Support**: Proper handling of international text
- **Compatibility**: Works with ASCII as UTF-8 subset
- **Safety**: Validates UTF-8 encoding to prevent corruption

### Object System Integration
- **Live Resolution**: Objects resolved at lex time, not parse time
- **Symbol Creation**: Unknown names become symbols automatically
- **Type Safety**: Strong typing through RefPerSys value system

### Error Handling Strategy
- **Graceful Degradation**: Continues processing when possible
- **Detailed Diagnostics**: Position information and context
- **Exception Safety**: Uses exceptions for unrecoverable errors

This implementation provides a robust foundation for RefPerSys's lexical analysis, with clear pathways for completing the remaining functionality and supporting the full range of RefPerSys's syntactic constructs.