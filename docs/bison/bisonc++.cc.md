# BisonC++ Parser Implementation Template (`bisonc++.cc`)

## Overview

The `bisonc++.cc` file is the main implementation template for BisonC++-generated parsers. It contains the core parsing algorithm, state management, error recovery, and action execution logic that gets instantiated for each generated parser class.

## File Structure and Template Directives

### Template Directives Used
- `$insert class.ih` - Includes the class implementation header
- `$insert debugincludes` - Debug header includes
- `$insert 4 staticdata` - Static parser data (state tables, production info)
- `$insert namespace-open` - Opens the parser namespace
- `$insert polymorphicCode` - Polymorphic semantic value support
- `$insert 4 baseclasscode` - Base class initialization code
- `$insert executeactioncases` - Grammar action implementations
- `$insert namespace-close` - Closes the parser namespace

### Key Components

## Parser State Management

### State Representation
```cpp
// SR_ (Shift-Reduce) array structure:
// First element: {state-type, last-element-index}
// Other elements: {required-token, action-to-perform}
// Action values: < 0 (reduce), 0 (accept), > 0 (shift)
```

### State Types
```cpp
enum StateType {
    NORMAL,        // Normal parsing state
    ERR_ITEM,      // Error recovery state
    REQ_TOKEN,     // State requires a token
    ERR_REQ,       // Error state requiring token
    DEF_RED,       // State with default reduction
    ERR_DEF,       // Error state with default reduction
    REQ_DEF,       // Token-required state with default reduction
    ERR_REQ_DEF    // Error state requiring token with default reduction
};
```

## Core Parsing Algorithm

### Main Parse Loop
```cpp
int @::parse() try {
    clearin_();                            // Initialize parser state
    while (true) {
        nextCycle_();                      // Execute one parsing cycle
    }
} catch (Return_ retValue) {
    return retValue or d_nErrors_;         // Return success/error count
}
```

### Parsing Cycle (`nextCycle_()`)
1. **Token Acquisition**: Get next token if state requires it
2. **Lookup**: Find action for current token in state table
3. **Action Execution**:
   - **Shift**: Push new state, consume token
   - **Reduce**: Execute grammar rule, pop states
   - **Accept**: Parsing successful
   - **Error**: Initiate error recovery

## Error Recovery System

### Error Recovery Process
```cpp
void @::errorRecovery_() {
    // Find error-handling state by popping stack
    while (not (s_state[top_()][0].d_type & ERR_ITEM)) {
        pop_();  // Pop until error state found
    }

    // Continue parsing from error state
    startRecovery_();
}
```

### Error State Transition
```cpp
void @::startRecovery_() {
    pushToken_(Reserved_::errTok_);      // Insert error token
    push_(lookup_());                     // Push error state
    d_recovery = true;                    // Enter recovery mode
}
```

## Action Execution

### Rule Execution Framework
```cpp
void @::executeAction_(int production) try {
    if (token_() != Reserved_::UNDETERMINED_)
        pushToken_(token_());     // Preserve current token

    switch (production) {
        $insert 8 actioncases    // Grammar-specific actions
    }
} catch (std::exception const &exc) {
    exceptionHandler(exc);       // Handle action exceptions
}
```

### Semantic Value Management
- **STYPE_**: Semantic value type (polymorphic or union-based)
- **vs_(idx)**: Access semantic values on parse stack
- **d_val_**: Current semantic value

## Stack Management

### Parser Stack Operations
```cpp
void @Base::push_(size_t state) {
    // Expand stack if necessary
    if (stackSize_() == currentSize) {
        newSize = currentSize + STACK_EXPANSION_;
        d_stateStack.resize(newSize);
    }

    ++d_stackIdx;
    d_stateStack[d_stackIdx] = StatePair{d_state = state, d_val_};
}

void @Base::pop_(size_t count) {
    d_stackIdx -= count;
    d_state = d_stateStack[d_stackIdx].first;
}
```

### Stack Elements
```cpp
using StatePair = std::pair<size_t, STYPE_>;  // {state, semantic_value}
std::vector<StatePair> d_stateStack;
StatePair *d_vsp;                              // Value stack pointer
```

## Token Management

### Token Processing
```cpp
void @::nextToken_() {
    if (token_() != Reserved_::UNDETERMINED_) {
        return;  // Token already available
    }

    if (savedToken_() != Reserved_::UNDETERMINED_) {
        popToken_();  // Use saved token
    } else {
        ++d_acceptedTokens_;
        lex_(lex());  // Get new token from lexer
        print_();     // Debug output
    }
}
```

### Token Constants
```cpp
enum Reserved_ {
    UNDETERMINED_ = -2,  // No token available
    EOF_          = -1,  // End of input
    errTok_       = 256  // Error token
};
```

## Debug Support

### Debug Output Control
```cpp
void @Base::setDebug(bool mode) {
    d_actionCases_ = false;
    d_debug_ = mode;
}

void @Base::setDebug(DebugMode_ mode) {
    d_actionCases_ = mode & ACTIONCASES;
    d_debug_ = mode & ON;
}
```

### Debug Output Points
- State transitions and token consumption
- Rule reductions and semantic actions
- Error recovery operations
- Stack operations and contents

## Integration with Generated Code

### Template Expansion Points
- **`$insert executeactioncases`**: Grammar-specific reduction actions
- **`$insert 4 staticdata`**: Parser tables and production information
- **`$insert polymorphicCode`**: Polymorphic semantic value support
- **`$insert debugincludes`**: Debug facility headers

### Generated Parser Structure
```
@Base (base class)
├── State management
├── Stack operations
├── Token handling
└── Debug facilities

@ (derived parser class)
├── Grammar-specific actions
├── Lexer interface
├── Error handling
└── Semantic value processing
```

## Performance Characteristics

### Time Complexity
- **Lookup**: O(1) average case (linear search in SR arrays)
- **Stack Operations**: O(1) amortized (occasional resizing)
- **Error Recovery**: O(depth) where depth is stack unwind distance

### Space Complexity
- **State Tables**: O(number of states × average transitions per state)
- **Parse Stack**: O(maximum parse tree depth)
- **Semantic Values**: O(stack depth × semantic value size)

## Error Handling

### Exception Safety
- **Action Exceptions**: Caught and handled by `exceptionHandler`
- **Parse Exceptions**: Converted to return codes
- **Stack Underflow**: Detected and aborted

### Error Recovery
- **Token Insertion**: Error tokens allow continuation
- **Stack Unwinding**: Pop to error-handling states
- **Continuation**: Resume parsing after error state

This template provides the complete parsing infrastructure that BisonC++ uses to generate efficient, maintainable C++ parsers with comprehensive error handling and debugging capabilities.