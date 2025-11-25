# Parser Implementation Analysis (`parsrepl_rps.cc`)

## Overview

The `parsrepl_rps.cc` file implements the core parser for the Read-Eval-Print Loop (REPL) in the Reflective Persistent System (RefPerSys), providing a recursive descent parser that translates tokenized input into executable expressions. This file establishes the grammatical foundation for RefPerSys expressions, implementing operator precedence, associativity rules, and the transformation of lexical tokens into abstract syntax trees represented as RefPerSys values and instances.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2019-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// Git versioning information
extern "C" const char rps_parsrepl_gitid[];
extern "C" const char rps_parsrepl_date[];
extern "C" const char rps_parsrepl_shortgitid[];
extern "C" const char rps_parsrepl_timestamp[];
```

### Key Dependencies
- **refpersys.hh**: Core system definitions and Rps_TokenSource class
- **generated/rps-parser-impl.cc**: Auto-generated parser implementation
- **Token management**: Rps_LexTokenZone and token stream handling
- **Value system**: Rps_Value, Rps_InstanceValue for AST representation

## Rps_TokenSource: Parser Infrastructure

### Class Overview
`Rps_TokenSource` provides the core parsing infrastructure, managing token streams, position tracking, and recursive descent parsing. It transforms sequences of lexical tokens into executable RefPerSys expressions.

### Key Infrastructure Methods
```cpp
// Token management
Rps_Value lookahead_token(Rps_CallFrame* cf, int ahead = 0);  // Peek ahead in token stream
Rps_Value get_token(Rps_CallFrame* cf);                       // Consume and return next token
void consume_front_token(Rps_CallFrame* cf, bool* psuccess = nullptr); // Consume token with success flag

// Position tracking
std::string position_str() const;                             // Current parsing position
const char* curcptr() const;                                   // Current character pointer
void display_current_line_with_cursor(std::ostream& out);     // Debug display with cursor

// Garbage collection integration
void gc_mark(Rps_GarbageCollector& gc) const;                 // Mark token references
```

### Parsing State Management
```cpp
std::deque<Rps_Value> toksrc_token_deq;  // Token queue for lookahead
std::string toksrc_input;                // Original input string
const char* toksrc_curcptr;              // Current parsing position
```

**Purpose**: Provides thread-safe token stream management with lookahead capabilities and precise position tracking for error reporting.

## Expression Parsing Hierarchy

### Grammar Structure
RefPerSys expressions follow a hierarchical grammar with clear precedence levels:

```
expression ::= disjunction { "||" disjunction }*
disjunction ::= conjunction { "&&" conjunction }*
conjunction ::= comparison
comparison ::= sum
sum ::= term { ("+" | "-") term }*
term ::= factor { ("*" | "/" | "%") factor }*
factor ::= primary
primary ::= INTEGER | DOUBLE | STRING | OBJECT | "(" expression ")"
```

### Expression Parsing (Top Level)
```cpp
Rps_Value Rps_TokenSource::parse_expression(Rps_CallFrame* callframe, bool* pokparse)
{
  // Parse sequence of disjunctions separated by ||
  // Returns Rps_InstanceValue if multiple disjunctions, otherwise single disjunction
}
```

**Purpose**: Handles the top-level expression parsing with logical OR operations.

### Disjunction Parsing
```cpp
Rps_Value Rps_TokenSource::parse_disjunction(Rps_CallFrame* callframe, bool* pokparse)
{
  // Parse sequence of conjunctions separated by &&
  // Returns Rps_InstanceValue if multiple conjunctions, otherwise single conjunction
}
```

**Purpose**: Handles logical AND operations between conjunctions.

### Conjunction Parsing
```cpp
Rps_Value Rps_TokenSource::parse_conjunction(Rps_CallFrame* callframe, bool* pokparse)
{
  // Parse single comparison (placeholder for future conjunction logic)
  // Currently delegates to parse_comparison
}
```

**Purpose**: Handles conjunction operations (currently a placeholder for future expansion).

### Comparison Parsing
```cpp
Rps_Value Rps_TokenSource::parse_comparison(Rps_CallFrame* callframe, bool* pokparse)
{
  // Parse single sum (placeholder for comparison operators)
  // Currently delegates to parse_sum
}
```

**Purpose**: Handles comparison operators (< > <= >=) - currently unimplemented.

### Sum Parsing (Addition/Subtraction)
```cpp
Rps_Value Rps_TokenSource::parse_sum(Rps_CallFrame* callframe, bool* pokparse)
{
  // Parse left term, then handle + and - operators
  // Supports multiple terms with same operator type
  // Returns Rps_InstanceValue for binary operations
}
```

**Purpose**: Handles addition and subtraction with left-associative semantics.

### Term Parsing (Multiplication/Division)
```cpp
Rps_Value Rps_TokenSource::parse_term(Rps_CallFrame* callframe, bool* pokparse)
{
  // Parse sequence of factors with *, /, % operators
  // All operators in sequence must be the same type
  // Returns Rps_InstanceValue for binary operations
}
```

**Purpose**: Handles multiplication, division, and modulus operations.

### Factor Parsing
```cpp
Rps_Value Rps_TokenSource::parse_factor(Rps_CallFrame* callframe, bool* pokparse)
{
  // Parse primary with optional unary operators
  // Currently delegates to parse_primary
}
```

**Purpose**: Handles unary operators and primary expressions - currently unimplemented.

### Primary Parsing (Atomic Expressions)
```cpp
Rps_Value Rps_TokenSource::parse_primary(Rps_CallFrame* callframe, bool* pokparse)
{
  // Parse atomic expressions: literals, objects, parenthesized expressions
  // Handles: integers, doubles, strings, objects, parentheses
}
```

**Purpose**: Handles the most basic expression elements and parenthesized subexpressions.

## Operator Precedence and Associativity

### Precedence Hierarchy (Highest to Lowest)
1. **Primary**: Literals, objects, parentheses `(expr)`
2. **Unary**: Factor operations (unimplemented)
3. **Multiplicative**: `*`, `/`, `%` (left-associative)
4. **Additive**: `+`, `-` (left-associative)
5. **Comparison**: `<`, `>`, `<=`, `>=` (unimplemented)
6. **Conjunctive**: `&&` (left-associative)
7. **Disjunctive**: `||` (left-associative)

### Associativity Rules
- **Left-associative**: Most binary operators associate left-to-right
- **Same operator requirement**: Within a term/sum, all operators must be identical
- **Instance creation**: Binary operations create `Rps_InstanceValue` objects

### Binary Operation Representation
```cpp
// For expression: a + b - c
// Creates: Rps_InstanceValue(minus_binop, [Rps_InstanceValue(plus_binop, [a, b]), c])

// For expression: x * y / z
// Creates: Rps_InstanceValue(div_binop, [Rps_InstanceValue(mult_binop, [x, y]), z])
```

## Token Consumption and Lookahead

### Lookahead Mechanism
```cpp
Rps_Value lookahead_token(Rps_CallFrame* cf, int ahead = 0)
{
  // Fill token queue as needed
  while (toksrc_token_deq.size() <= ahead) {
    // Get next token from lexer
    Rps_Value nexttok = get_next_lexical_token();
    toksrc_token_deq.push_back(nexttok);
  }
  return ahead < toksrc_token_deq.size() ? toksrc_token_deq[ahead] : nullptr;
}
```

**Purpose**: Provides arbitrary lookahead in the token stream without consumption.

### Token Consumption
```cpp
Rps_Value get_token(Rps_CallFrame* cf)
{
  if (toksrc_token_deq.empty()) {
    // Get next token from lexer
    return get_next_lexical_token();
  } else {
    // Return and remove from queue
    Rps_Value tok = toksrc_token_deq.front();
    toksrc_token_deq.pop_front();
    return tok;
  }
}

void consume_front_token(Rps_CallFrame* cf, bool* psuccess = nullptr)
{
  // Consume token and update success flag
  // Used when token value is not needed
}
```

**Purpose**: Manages token consumption with queue-based buffering for efficient parsing.

## Error Handling and Debugging

### Parsing Failure Mechanism
```cpp
#define RPS_PARSREPL_FAILURE(CallFrame, Message) \
  do { \
    std::ostringstream _failure_msg_; \
    _failure_msg_ << Message; \
    throw std::runtime_error(_failure_msg_.str()); \
  } while(0)
```

**Purpose**: Provides structured error reporting with position information and context.

### Debug Logging
```cpp
#define RPS_DEBUG_LOG(Category, Message) \
  // Extensive logging for parser debugging
  // Includes call numbers, positions, token states
```

**Purpose**: Comprehensive debug output for parser development and troubleshooting.

### Position Tracking
```cpp
std::string position_str() const
{
  // Returns "file:line:column" format
  // Used in error messages and debugging
}
```

**Purpose**: Provides human-readable position information for error reporting.

## Incomplete and Unimplemented Features

### Comparison Operators
```cpp
Rps_Value parse_comparison(Rps_CallFrame* callframe, bool* pokparse)
{
  // Currently just delegates to parse_sum
  // TODO: Implement < > <= >= operators
}
```

**Status**: Placeholder implementation, operators not yet supported.

### Factor Operations
```cpp
Rps_Value parse_factor(Rps_CallFrame* callframe, bool* pokparse)
{
  // Currently just delegates to parse_primary
  // TODO: Implement unary operators (+, -, !, etc.)
}
```

**Status**: Placeholder implementation, unary operators not yet supported.

### Primary Complements
```cpp
Rps_Value parse_primary_complement(Rps_CallFrame* callframe,
                                  Rps_Value primaryexparg, bool* pokparse)
{
  // TODO: Implement member access, function calls, indexing
}
```

**Status**: Completely unimplemented, needed for object.method() and array[index] syntax.

### Product Parsing
```cpp
Rps_Value parse_product(Rps_CallFrame* callframe, bool* pokparse)
{
  // Duplicate of parse_term
  // TODO: Remove duplication
}
```

**Status**: Redundant implementation that should be consolidated.

## Garbage Collection Integration

### Parser State GC Marking
```cpp
void Rps_TokenSource::gc_mark(Rps_GarbageCollector& gc) const
{
  // Mark all tokens in queue
  for (auto& tok : toksrc_token_deq) {
    if (tok.is_ptr()) gc.mark_value(tok);
  }

  // Mark additional parser state
  _.set_additional_gc_marker([&](Rps_GarbageCollector* gc) {
    // Mark local variables in parsing frames
    for (auto& disj : disjvect) gc.mark_value(disj);
    for (auto& conj : conjvect) gc.mark_value(conj);
    // ... etc for all parsing vectors
  });
}
```

**Purpose**: Ensures parser state doesn't cause memory leaks during garbage collection.

## Usage Patterns and Examples

### Simple Literal Parsing
```cpp
// Input: "42"
// parse_primary -> returns Rps_Value containing integer 42

// Input: "\"hello\""
// parse_primary -> returns Rps_Value containing string "hello"

// Input: "_1aGtWm38Vw701jDhZn" (object ID)
// parse_primary -> returns Rps_ObjectRef for the specified object
```

### Arithmetic Expression Parsing
```cpp
// Input: "2 + 3 * 4"
// Parsing sequence:
// parse_expression -> parse_disjunction -> parse_conjunction -> parse_comparison
//                 -> parse_sum -> parse_term("2") -> parse_factor -> parse_primary("2")
//                 -> consume "+" -> parse_term("3") -> parse_primary("3")
//                 -> consume "*" -> parse_factor("4") -> parse_primary("4")
// Result: Rps_InstanceValue(mult_binop, [3, 4]) then
//         Rps_InstanceValue(plus_binop, [2, mult_result])
```

### Logical Expression Parsing
```cpp
// Input: "a && b || c"
// Parsing sequence:
// parse_expression -> parse_disjunction("a && b || c")
//                  -> parse_conjunction("a && b") -> Rps_InstanceValue(and_binop, [a, b])
//                  -> consume "||" -> parse_conjunction("c") -> c
// Result: Rps_InstanceValue(or_binop, [and_result, c])
```

### Parenthesized Expressions
```cpp
// Input: "(2 + 3) * 4"
// Parsing sequence:
// parse_primary -> detect "(" -> parse_expression("2 + 3") -> consume ")"
//               -> returns result of "2 + 3" parsing
// Then continues with "* 4" in parse_term
```

## Design Rationale

### Recursive Descent Architecture
**Why recursive descent parsing?**
- **Clarity**: Each function handles one precedence level
- **Modularity**: Easy to add new operators and precedence levels
- **Debugging**: Clear call stack shows parsing progress
- **Extensibility**: New expression types can be added incrementally

### Strict Operator Associativity
**Why require same operators in sequences?**
- **Simplicity**: Avoids complex precedence resolution
- **Clarity**: Forces explicit parentheses for mixed operations
- **Safety**: Prevents ambiguous operator sequences
- **Implementation**: Simplifies AST construction

### Instance-Based AST Representation
**Why Rps_InstanceValue for operations?**
- **Uniformity**: All expressions are RefPerSys values
- **Reflection**: Operations can be inspected and manipulated
- **Evaluation**: Instances can be directly evaluated by the runtime
- **Persistence**: Expression trees can be stored and reloaded

### Extensive Debugging Support
**Why so much debug logging?**
- **Development**: Parser development requires detailed tracing
- **Maintenance**: Complex parsing logic needs observability
- **User Support**: Detailed error messages aid troubleshooting
- **Performance**: Debug logging can be conditionally compiled out

### Incremental Implementation
**Why many unimplemented functions?**
- **Bootstrapping**: Core functionality first, advanced features later
- **Testing**: Allows testing basic parsing before complex features
- **Evolution**: Parser can grow organically with language needs
- **Stability**: Incomplete features don't break existing functionality

This implementation establishes the grammatical foundation for RefPerSys expressions, providing a solid base for language evolution while maintaining the flexibility and reflection capabilities that characterize the system.