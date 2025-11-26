# Commutative Operator Creation Plugin (`plugins_dir/rpsplug_createcommutativeoperator.cc`)

**File Path:** `plugins_dir/rpsplug_createcommutativeoperator.cc`

## Overview

This RefPerSys plugin creates commutative binary operators for the REPL (Read-Eval-Print Loop) system. Commutative operators are mathematical or logical operations where the order of operands doesn't matter (e.g., addition, multiplication, logical AND). The plugin enables extension of the REPL's expression evaluation capabilities with custom operators that have precedence and are order-independent.

## Plugin Purpose

### Commutative Operation Extension
The plugin enables dynamic extension of REPL parsing with:

- **Binary Operations:** Creating two-operand operations
- **Commutativity:** Supporting order-independent operations (a + b = b + a)
- **Precedence Handling:** Managing operator precedence in expressions
- **Symbol Integration:** Associating operators with symbolic names
- **Expression Evaluation:** Enabling symmetric operations in complex expressions

## Implementation Status

### Current State
```cpp
#warning unimplemented rpsplug_createcommutativeoperator
```

**Status:** Framework created, operator registration incomplete

### Design Intent
- **Commutative:** Order doesn't matter in operations
- **Binary Operators:** Two-operand operations
- **Precedence System:** Operator precedence for parsing
- **REPL Integration:** Expression evaluation support

## Command-Line Interface

### Basic Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_createcommutativeoperator.so \
            --plugin-arg=rpsplug_createcommutativeoperator:++ \
            --extra=name=plusplus \
            --extra=precedence=8 \
            --batch --dump=.
```

### Required Arguments
- **`--plugin-arg`**: The operator symbol (e.g., "+", "*", "&")

### Optional Arguments
- **`--extra=name`**: Symbolic name for the operator
- **`--extra=precedence`**: Operator precedence level (numeric)
- **`--extra=comment`**: Documentation comment

## Plugin Architecture

### Entry Point
```cpp
void rps_do_plugin(const Rps_Plugin* plugin)
```

**Execution Flow:**
1. **Argument Validation:** Check operator symbol format
2. **Precedence Parsing:** Extract precedence value
3. **Operator Creation:** Instantiate operator object
4. **Metadata Setup:** Configure name and precedence attributes
5. **Registration:** (Currently incomplete - placeholder only)

## Operator Symbol Validation

### Format Requirements
```cpp
if (ispunct(plugarg[0])) {
  bool allpunct = true;
  for (const char* pc = plugarg; allpunct && *pc; pc++)
    allpunct = ispunct(*pc);
  argispunct = allpunct;
}

if (isalpha(plugarg[0])) {
  bool allident = true;
  for (const char* pc = plugarg; allident && *pc; pc++)
    allident = isalnum(*pc) || *pc == '_';
  argisident = allident;
}

if (!argispunct && !argisident)
  RPS_FATALOUT("bad argument not identifier or all-delim");
```

**Valid Formats:**
- **Punctuation Operators:** All characters must be punctuation (e.g., "+", "*", "&")
- **Identifier Operators:** All characters must form valid identifier (e.g., "plus", "and")

## Precedence Handling

### Precedence Parsing
```cpp
if (xtraprecedence && isdigit(xtraprecedence[0]))
  precedence = atoi(xtraprecedence);
```

**Precedence Requirements:**
- Must be numeric value
- Used for expression parsing order
- Conventionally small non-negative integers

## Core Operator Creation

### Operator Instantiation
```cpp
_f.obnewoper = Rps_ObjectRef::make_object(&_,
    _f.obclasscommut,  // repl_commutative_operator∈class
    Rps_ObjectRef::root_space());
```

**Operator Properties:**
- Instance of `repl_commutative_operator` class
- Placed in root object space
- Gets unique object ID

## Metadata Configuration

### Name Attribute
```cpp
if (xtraname) {
  _f.namestr = Rps_Value{std::string(xtraname)};
  _f.obnewoper->put_attr(RPS_ROOT_OB(_1EBVGSfW2m200z18rx), _f.namestr);
}
```

**Naming:** Optional symbolic name for the operator

### Precedence Attribute
```cpp
if (precedence >= 0) {
  _f.obnewoper->put_attr(RPS_ROOT_OB(_7iVRsTR8u3D00Cy0hp), // repl_precedence∈symbol
                         Rps_Value::make_tagged_int(precedence));
}
```

**Precedence Storage:** Tagged integer for parsing priority

## Missing Implementation

### Operator Registration
```cpp
/** TODO:
 * we need to fill obnewoper
 * we need to register obnewoper as a root or as a constant
 */
```

**Incomplete Features:**
- **Symbol Association:** No associated symbol created
- **Registration:** Not added to operator registries
- **REPL Integration:** Not connected to expression parser
- **Set Integration:** Not added to operator set

### Current Failure
```cpp
RPS_FATALOUT("not implemented for " << Rps_QuotedC_String(plugarg)
             << " but created " << RPS_OBJECT_DISPLAY(_f.obnewoper)
             << " see rpsplug_thesetreploper.cc plugin");
```

**Fatal Error:** Plugin currently fails due to incomplete implementation

## Integration with REPL System

### Expression Parsing
Once complete, operators would integrate with:
- **Precedence Parsing:** Expression evaluation order
- **Commutative Semantics:** Order-independent operations
- **Binary Operations:** Two-operand calculations
- **REPL Evaluation:** Interactive expression computation

## Use Cases

### Mathematical Operations
- **Addition:** a + b (order doesn't matter)
- **Multiplication:** a * b (order doesn't matter)
- **Logical AND:** a && b (order doesn't matter)
- **Set Union:** a ∪ b (order doesn't matter)

### Custom Operations
- **Bitwise Operations:** a & b, a | b
- **Set Operations:** a ∩ b, a ∪ b
- **Logical Operations:** a ∧ b, a ∨ b
- **Domain Operations:** Custom symmetric operations

## Build and Deployment

### Compilation
```bash
cd $REFPERSYS_TOPDIR
./do-build-refpersys-plugin \
  -i plugins_dir/rpsplug_createcommutativeoperator.cc \
  -o /tmp/rpsplug_createcommutativeoperator.so \
  -L /tmp/rpsplug_createcommutativeoperator.so
```

### Runtime Usage (When Complete)
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_createcommutativeoperator.so \
            --plugin-arg=rpsplug_createcommutativeoperator:+ \
            --extra=name=plus \
            --extra=precedence=6 \
            --batch --dump=.
```

## Example Usage (Planned)

### Creating Arithmetic Operators
```bash
# Addition operator
./refpersys --plugin-after-load=/tmp/rpsplug_createcommutativeoperator.so \
            --plugin-arg=rpsplug_createcommutativeoperator:+ \
            --extra=name=plus \
            --extra=precedence=6 \
            --batch --dump=.

# Multiplication operator
./refpersys --plugin-after-load=/tmp/rpsplug_createcommutativeoperator.so \
            --plugin-arg=rpsplug_createcommutativeoperator:* \
            --extra=name=multiply \
            --extra=precedence=7 \
            --batch --dump=.
```

### Creating Logical Operators
```bash
# Logical AND operator
./refpersys --plugin-after-load=/tmp/rpsplug_createcommutativeoperator.so \
            --plugin-arg=rpsplug_createcommutativeoperator:&& \
            --extra=name=logical_and \
            --extra=precedence=4 \
            --batch --dump=.
```

## Performance Characteristics

### Current Performance
- **Validation:** Fast string parsing and format checking
- **Object Creation:** Standard RefPerSys instantiation
- **Attribute Setup:** Minimal metadata configuration

### Future Performance Goals
- **Operator Lookup:** Fast operator resolution in parsing
- **Commutative Evaluation:** Efficient symmetric operations
- **Memory Usage:** Minimal per-operator overhead
- **Scalability:** Support for large operator sets

## Relationship to Other Plugins

### Complementary Plugins
- **`rpsplug_createnoncommutativeoperator.cc`**: Creates non-commutative operators
- **`rpsplug_createmonoper.cc`**: Creates monadic operators
- **`rpsplug_thesetreploper.cc`**: Manages operator registry

### Operator Ecosystem
```
Commutative ← Non-Commutative ← Monadic
     ↓              ↓              ↓
Symmetric     Ordered       Unary
Operations  Operations   Operations
```

## Thread Safety

### Synchronization
Plugin uses RefPerSys's existing object access patterns which include appropriate locking for thread-safe object creation and attribute setting.

## Error Handling

### Validation Failures
- **Missing Symbol:** Requires operator symbol
- **Invalid Format:** Must be valid operator format
- **Length Limits:** Symbol cannot exceed buffer size

### Fatal Errors
```cpp
RPS_FATALOUT("without argument; should be some non-empty string");
RPS_FATALOUT("not implemented for " << Rps_QuotedC_String(plugarg));
```

## File Status

**Status:** Framework/Incomplete
**Date:** 2023-2025
**Purpose:** Commutative operator creation infrastructure

## Summary

The `rpsplug_createcommutativeoperator.cc` plugin establishes the foundational infrastructure for creating commutative binary operators in RefPerSys's REPL system. While currently incomplete, it demonstrates the approach to operator creation with precedence handling and proper class relationships. The plugin creates operator objects with appropriate metadata but lacks the registration logic to integrate them into the REPL's expression evaluation system. Once completed, this plugin will enable sophisticated symmetric operations like addition, multiplication, and logical AND with proper operator precedence and commutative semantics, significantly extending the REPL's computational capabilities for order-independent operations. The plugin represents an important component in RefPerSys's extensible operator system, providing the foundation for mathematical and logical operations that don't depend on operand order.