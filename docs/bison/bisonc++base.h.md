# BisonC++ Base Parser Class Template (`bisonc++base.h`)

## Overview

The `bisonc++base.h` file defines the base class template for BisonC++-generated parsers. It provides the core parsing infrastructure including state management, stack operations, token handling, error recovery, and debug support that all generated parsers inherit.

## File Structure and Includes

### Header Guards and Includes
```cpp
#ifndef @Base_h_included
#define @Base_h_included

#include <exception>
#include <vector>
#include <iostream>
$insert polyincludes    // Polymorphic semantic value includes
$insert preincludes     // User-specified includes
$insert debugincludes   // Debug facility includes
```

### Namespace and Class Structure
```cpp
$insert namespace-open

$insert polymorphic     // Polymorphic semantic value definitions

$insert parserbase      // Main parser base class definition

$insert namespace-close
```

## Core Data Structures

### State Representation
```cpp
// SR_ (Shift-Reduce) structure for state tables
struct SR_ {
    union {
        int _field_1_;      // For initialization
        StateType d_type;   // State type flags
        int d_token;        // Token for transitions
    };
    union {
        int _field_2_;
        int d_lastIdx;      // Last element index (negative = SHIFT)
        int d_action;       // Action: <0 reduce, 0 accept, >0 shift
    };
};
```

### State Types
```cpp
enum StateType {
    NORMAL,        // Normal parsing state
    ERR_ITEM,      // Error recovery state
    REQ_TOKEN,     // State requires token
    ERR_REQ,       // Error + token required
    DEF_RED,       // Has default reduction
    ERR_DEF,       // Error + default reduction
    REQ_DEF,       // Token required + default reduction
    ERR_REQ_DEF    // Error + token required + default reduction
};
```

### Stack Management
```cpp
using StatePair = std::pair<size_t, STYPE_>;    // {state, semantic_value}
using TokenPair = std::pair<int, STYPE_>;       // {token, semantic_value}

std::vector<StatePair> d_stateStack;            // Parser stack
StatePair *d_vsp = 0;                          // Value stack pointer
size_t d_state = 0;                            // Current state
int d_stackIdx = -1;                           // Stack top index
```

## Parser State Members

### Token and Input Management
```cpp
TokenPair d_next;           // Saved token for lookahead
int d_token;               // Current token
bool d_terminalToken;      // Whether current token is terminal
```

### Error and Recovery State
```cpp
bool d_recovery = false;           // In error recovery mode
size_t d_requiredTokens_;          // Tokens needed for recovery
size_t d_nErrors_;                 // Error count
size_t d_acceptedTokens_;          // Tokens accepted
```

### Debug and Control Flags
```cpp
bool d_actionCases_ = false;       // Debug action cases
bool d_debug_ = true;              // Debug enabled
STYPE_ d_val_;                     // Current semantic value
```

## Core Methods

### Initialization and Cleanup
```cpp
@Base();                           // Constructor
void clearin_();                   // Reset parser state
```

**Purpose**: Initialize/reset parser to clean state for new parse.

### Stack Operations
```cpp
void push_(size_t state);          // Push state onto stack
void pop_(size_t count = 1);       // Pop states from stack
inline size_t top_() const;        // Get top state
inline size_t state_() const;      // Get current state
inline size_t stackSize_() const;  // Get stack size
```

**Stack Growth**: Automatic expansion when capacity reached.

### Token Management
```cpp
void lex_(int token);              // Set current token
void popToken_();                  // Consume current token
void pushToken_(int token);        // Save token for later
void redoToken_();                 // Re-queue current token
inline int token_() const;         // Get current token
inline int savedToken_() const;    // Get saved token
```

### Semantic Value Access
```cpp
inline STYPE_ &vs_(int idx);       // Access stack value by index
```

**Indexing**: `vs_(0)` = top of stack, `vs_(-1)` = second from top, etc.

## Parsing Algorithm

### State Lookup
```cpp
int lookup_() const;
```

**Algorithm**:
1. Get current state's SR array
2. Search for current token in transitions
3. Return action: negative (reduce), zero (accept), positive (shift)
4. Use default reduction if no match found

### Action Execution
```cpp
void reduce_(int rule);            // Execute reduction
void shift_(int state);            // Execute shift
```

**Reduction Process**:
1. Pop rule-length elements from stack
2. Set token to non-terminal
3. Execute semantic action (if any)

## Error Recovery System

### Error Detection and Recovery
```cpp
void startRecovery_();             // Initiate error recovery
```

**Recovery Protocol**:
1. Push error token onto input stream
2. Push error state onto stack
3. Continue parsing from error state
4. Accept tokens until valid continuation found

### Error State Navigation
```cpp
// Find error-handling state by unwinding stack
while (not (s_state[top_()][0].d_type & ERR_ITEM)) {
    pop_();  // Pop until error state reached
}
```

## Debug Support

### Debug Control
```cpp
void setDebug(bool mode);                      // Enable/disable debug
void setDebug(DebugMode_ mode);                // Fine-grained control

enum DebugMode_ {
    OFF = 0,
    ON = 1 << 0,
    ACTIONCASES = 1 << 1
};
```

### Debug Output Methods
```cpp
$insert debugdecl        // Debug function declarations
$insert debugfunctions   // Debug function implementations
```

**Debug Information Available**:
- State transitions and token consumption
- Rule reductions and semantic actions
- Stack operations and contents
- Error recovery operations

## Polymorphic Semantic Values

### Type-Safe Semantic Values
```cpp
$insert polymorphic      // Polymorphic value definitions

// Base class for all semantic values
class Base {
    virtual ~Base();
    virtual Base *clone() const = 0;
    virtual void *data() const = 0;
    Tag_ tag() const;
};

// Template for specific types
template <Tag_ tg_>
class Semantic : public Base {
    typename TypeOf<tg_>::type d_data;
    // ...
};
```

### SType Wrapper
```cpp
class SType : private std::unique_ptr<Base> {
public:
    template <Tag_ tag>
    typename TypeOf<tag>::type &get();          // Type-safe access

    template <Tag_ tagParam, typename ...Args>
    void assign(Args &&...args);                // Type-safe assignment

    Tag_ tag() const;                           // Get value type
};
```

## Location Tracking (Optional)

### Location Type Definition
```cpp
$insert LTYPE              // Location type (if enabled)

struct LTYPE_ {
    int timestamp;
    int first_line, first_column;
    int last_line, last_column;
    char *text;
};
```

### Location Stack Management
```cpp
$insert LTYPEstack         // Location stack members
$insert LTYPEdata          // Location data accessors

LTYPE_ const &lsp_(int idx) const;  // Location stack access
```

## Template Integration Points

### Generated Code Insertion
- **`$insert tokens`**: Token definitions and constants
- **`$insert LTYPE/STYPE`**: Semantic and location value types
- **`$insert parserbase`**: Main base class definition
- **`$insert staticdata`**: Parser tables and state machines
- **`$insert polymorphic`**: Polymorphic value support
- **`$insert debug*`**: Debug facility implementations

### Customization Points
- **`STYPE_`**: Semantic value type (union or polymorphic)
- **`LTYPE_`**: Location information type
- **Token constants**: Generated from grammar
- **State tables**: Generated from grammar analysis

## Memory Management

### Stack Expansion
```cpp
// Automatic stack growth
size_t currentSize = d_stateStack.size();
if (stackSize_() == currentSize) {
    size_t newSize = currentSize + STACK_EXPANSION_;
    d_stateStack.resize(newSize);
    $insert LTYPEresize  // Resize location stack if present
}
```

### Semantic Value Handling
- **Copy semantics**: Values copied during stack operations
- **Move optimization**: `std::move` used where possible
- **Polymorphic**: Clone-on-copy for polymorphic values

## Exception Safety

### Exception Handling
```cpp
enum Return_ {
    PARSE_ACCEPT_ = 0,     // Successful parse
    PARSE_ABORT_ = 1       // Parse aborted
};

void ABORT() const;        // Throw abort exception
void ACCEPT() const;       // Throw accept exception
void ERROR() const;        // Throw error exception
```

### Exception Propagation
- **Parse results**: Returned as exception codes
- **Action exceptions**: Propagated through `exceptionHandler`
- **Stack consistency**: Maintained during exception unwinding

## Performance Characteristics

### Time Complexity
- **State lookup**: O(transition count) - linear search in SR arrays
- **Stack operations**: O(1) amortized - occasional resizing
- **Memory allocation**: Minimal - pre-allocated stacks with growth

### Space Complexity
- **State tables**: O(states × average transitions)
- **Parser stack**: O(maximum parse depth)
- **Semantic values**: O(stack depth × value size)

### Optimization Features
- **Inline methods**: Many accessors are inline
- **Direct access**: Minimal indirection to parser state
- **Efficient stacks**: Vector-based with move semantics

## Integration with Generated Parser

### Inheritance Hierarchy
```
@Base (this template)
├── State management and stack operations
├── Token handling and lookahead
├── Error recovery mechanisms
├── Debug output facilities
└── Polymorphic semantic value support

@ (generated parser class)
├── Grammar-specific state tables
├── Token definitions and constants
├── Semantic actions and reductions
└── User-overridable methods (lex, error, etc.)
```

### Generated Code Dependencies
- **State tables**: `s_state[]`, `s_productionInfo[]`
- **Token mappings**: `s_symbol` map for debug output
- **Semantic actions**: `executeAction_()` implementations
- **Polymorphic types**: `Tag_`, `TypeOf<>` specializations

This base class template provides the complete parsing infrastructure that enables BisonC++ to generate efficient, feature-rich parsers with comprehensive error handling, debugging, and semantic value management capabilities.