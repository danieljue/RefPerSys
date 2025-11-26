# Monadic Operator Creation Plugin (`plugins_dir/rpsplug_createmonoper.cc`)

**File Path:** `plugins_dir/rpsplug_createmonoper.cc`

## Overview

This RefPerSys plugin creates monadic (unary) operators for the REPL (Read-Eval-Print Loop) system. Monadic operators are single-operand operations that transform a single value (e.g., negation, logical NOT, absolute value). The plugin enables extension of the REPL's expression evaluation capabilities with custom unary operators that have precedence and operate on individual values.

## Plugin Purpose

### Unary Operation Extension
The plugin enables dynamic extension of REPL parsing with:

- **Unary Operations:** Creating single-operand operations
- **Monadic Semantics:** One-input, one-output transformations
- **Precedence Handling:** Managing operator precedence in expressions
- **Symbol Integration:** Associating operators with symbolic names
- **Expression Evaluation:** Enabling unary operations in complex expressions

## Implementation Status

### Current State
```cpp
#warning incomplete rpsplug_createmonoper
```

**Status:** Framework created, operator registration incomplete

### Design Intent
- **Unary Operators:** Single-operand operations
- **Monadic Operations:** Pure functions on single values
- **Precedence System:** Operator precedence for parsing
- **REPL Integration:** Expression evaluation support

## Command-Line Interface

### Basic Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_createmonoper.so \
            --plugin-arg=rpsplug_createmonoper:- \
            --extra=name=negate \
            --extra=precedence=8 \
            --extra=comment='unary negation operator' \
            --batch --dump=.
```

### Required Arguments
- **`--plugin-arg`**: The operator symbol (e.g., "-", "!", "~")

### Optional Arguments
- **`--extra=name`**: Symbolic name for the operator
- **`--extra=precedence`**: Operator precedence level (numeric)
- **`--extra=comment`**: Documentation comment
- **`--extra=rooted`**: Make operator a root object (true/1)
- **`--extra=constant`**: Make operator a constant object (true/1)

## Plugin Architecture

### Entry Point
```cpp
void rps_do_plugin(const Rps_Plugin* plugin)
```

**Execution Flow:**
1. **Argument Validation:** Check operator symbol format
2. **Lifecycle Parsing:** Interpret rooted/constant flags
3. **Precedence Parsing:** Extract precedence value
4. **Operator Creation:** Instantiate operator object
5. **Metadata Setup:** Configure name, precedence, and comment attributes
6. **Registration:** Add to appropriate system collections

## Operator Symbol Validation

### Format Requirements
```cpp
if (ispunct(plugarg[0])) {
  bool allpunct = true;
  for (const char* pc = plugarg; allpunct && *pc; pc++)
    allpunct = ispunct(*pc);
  argispunct = allpunct;
} else if (isalpha(plugarg[0])) {
  bool allident = true;
  for (const char* pc = plugarg; allident && *pc; pc++)
    allident = isalnum(*pc) || *pc == '_';
  argisident = allident;
}

if (!argispunct && !argisident)
  RPS_FATALOUT("bad argument not identifier or all-delim");
```

**Valid Formats:**
- **Punctuation Operators:** All characters must be punctuation (e.g., "-", "!", "~")
- **Identifier Operators:** All characters must form valid identifier (e.g., "not", "abs")

## Lifecycle Configuration

### Root vs Constant Determination
```cpp
if (rooted) {
  if (!strcmp(rooted, "true")) isrooted = true;
  if (atoi(rooted) > 0) isrooted = true;
} else if (constant) {
  if (!strcmp(constant, "true")) isconstant = true;
  if (atoi(constant) > 0) isconstant = true;
}
```

**Lifecycle Options:**
- **Root:** Protected from GC, mutable
- **Constant:** Protected from GC, immutable
- **Plain:** Subject to GC (currently fails)

## Precedence Handling

### Precedence Parsing
```cpp
if (xtraprecedence && isdigit(xtraprecedence[0]))
  precedence = atoi(xtraprecedence);
else
  RPS_FATALOUT("with bad precedence " << precedence);
```

**Precedence Requirements:**
- Must be numeric value
- Used for expression parsing order
- Conventionally small non-negative integers

## Core Operator Creation

### Operator Instantiation
```cpp
_f.obnewoper = Rps_ObjectRef::make_object(&_,
    _f.obclassoper,  // repl_unary_operator∈class
    Rps_ObjectRef::root_space());
```

**Operator Properties:**
- Instance of `repl_unary_operator` class
- Placed in root object space
- Gets unique object ID

## Metadata Configuration

### Name Attribute
```cpp
if (xtraname) {
  _f.strname = Rps_Value{std::string(xtraname)};
  _f.obnewoper->put_attr(RPS_ROOT_OB(_1EBVGSfW2m200z18rx), _f.strname);
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

### Comment Attribute
```cpp
if (comment != nullptr) {
  _f.strcomment = Rps_StringValue(comment);
  _f.obnewoper->put_attr(RPS_ROOT_OB(_0jdbikGJFq100dgX1n), _f.strcomment);
}
```

**Documentation:** Operators can be documented for future reference

## Registration System

### Root Registration
```cpp
if (isrooted) {
  rps_add_root_object(_f.obnewoper);
  RPS_INFORMOUT("created root monadic REPL operator" << std::endl
                << RPS_OBJECT_DISPLAY(_f.obnewoper));
  return;
}
```

**Root Benefits:** Survives garbage collection, always accessible

### Constant Registration
```cpp
else if (isconstant) {
  rps_add_constant_object(&_, _f.obnewoper);
  RPS_INFORMOUT("created constant monadic REPL operator" << std::endl
                << RPS_OBJECT_DISPLAY(_f.obnewoper));
  return;
}
```

**Constant Benefits:** Immutable, system-wide availability

### Plain Operator (Incomplete)
```cpp
else
  RPS_FATALOUT("not implemented for " << Rps_QuotedC_String(plugarg));
```

**Current Limitation:** Plain operators not supported

## Missing Implementation

### Operator Registration
```cpp
/** TODO:
 * we need to fill obnewoper and register it as a root or as a constant
 */
```

**Incomplete Features:**
- **Symbol Association:** No associated symbol created
- **REPL Integration:** Not connected to expression parser
- **Set Integration:** Not added to operator set
- **Plain Operators:** No support for non-root/non-constant operators

## Integration with REPL System

### Expression Parsing
Once complete, operators would integrate with:
- **Unary Operations:** Single-operand transformations
- **Precedence Parsing:** Expression evaluation order
- **Monadic Semantics:** Pure functional operations
- **REPL Evaluation:** Interactive expression computation

## Use Cases

### Mathematical Operations
- **Negation:** -x (arithmetic negation)
- **Absolute Value:** |x| (magnitude)
- **Square Root:** √x (radical)
- **Logarithm:** log x (logarithmic)

### Logical Operations
- **Logical NOT:** !x (boolean negation)
- **Bitwise NOT:** ~x (bit complement)
- **Type Check:** ?x (type query)

### Custom Operations
- **Domain Operations:** Custom unary transformations
- **Type Conversions:** Value type changes
- **Validation:** Input validation operations

## Build and Deployment

### Compilation
```bash
cd $REFPERSYS_TOPDIR
./do-build-refpersys-plugin -v \
  -i plugins_dir/rpsplug_createmonoper.cc \
  -o /tmp/rpsplug_createmonoper.so
```

### Alternative Build
```cpp
make one-plugin \
  REFPERSYS_PLUGIN_SOURCE=$REFPERSYS_TOPDIR/plugins_dir/rpsplug_createmonoper.cc \
  REFPERSYS_PLUGIN_SHARED_OBJECT=/tmp/rpsplug_createmonoper.so
```

### Runtime Usage (When Complete)
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_createmonoper.so \
            --plugin-arg=rpsplug_createmonoper:- \
            --extra=name=negate \
            --extra=precedence=8 \
            --extra=comment='unary negation operator' \
            --extra=constant=1 \
            --batch --dump=.
```

## Example Usage (Planned)

### Creating Mathematical Operators
```bash
# Negation operator
./refpersys --plugin-after-load=/tmp/rpsplug_createmonoper.so \
            --plugin-arg=rpsplug_createmonoper:- \
            --extra=name=negate \
            --extra=precedence=8 \
            --extra=comment='unary arithmetic negation' \
            --extra=constant=1 \
            --batch --dump=.

# Logical NOT operator
./refpersys --plugin-after-load=/tmp/rpsplug_createmonoper.so \
            --plugin-arg=rpsplug_createmonoper:! \
            --extra=name=logical_not \
            --extra=precedence=8 \
            --extra=comment='logical negation operator' \
            --extra=rooted=1 \
            --batch --dump=.
```

### Creating Bitwise Operators
```bash
# Bitwise NOT operator
./refpersys --plugin-after-load=/tmp/rpsplug_createmonoper.so \
            --plugin-arg=rpsplug_createmonoper:~ \
            --extra=name=bitwise_not \
            --extra=precedence=8 \
            --extra=comment='bitwise complement operator' \
            --extra=constant=1 \
            --batch --dump=.
```

## Performance Characteristics

### Current Performance
- **Validation:** Fast string parsing and format checking
- **Object Creation:** Standard RefPerSys instantiation
- **Attribute Setup:** Minimal metadata configuration

### Future Performance Goals
- **Operator Lookup:** Fast operator resolution in parsing
- **Unary Evaluation:** Efficient single-operand operations
- **Memory Usage:** Minimal per-operator overhead
- **Scalability:** Support for large operator sets

## Relationship to Other Plugins

### Complementary Plugins
- **`rpsplug_createcommutativeoperator.cc`**: Creates commutative binary operators
- **`rpsplug_createnoncommutativeoperator.cc`**: Creates non-commutative binary operators
- **`rpsplug_thesetreploper.cc`**: Manages operator registry

### Operator Ecosystem
```
Monadic ← Binary Commutative ← Binary Non-Commutative
    ↓              ↓                      ↓
Unary       Symmetric            Ordered
Operations  Operations          Operations
```

## Thread Safety

### Synchronization
Plugin uses RefPerSys's existing object access patterns which include appropriate locking for thread-safe object creation and attribute setting.

## Error Handling

### Validation Failures
- **Missing Symbol:** Requires operator symbol
- **Invalid Format:** Must be valid operator format
- **Bad Precedence:** Must be valid numeric precedence
- **Length Limits:** Symbol cannot exceed buffer size

### Fatal Errors
```cpp
RPS_FATALOUT("without argument; should be some non-empty string");
RPS_FATALOUT("with bad precedence " << precedence);
RPS_FATALOUT("not implemented for " << Rps_QuotedC_String(plugarg));
```

## File Status

**Status:** Framework/Incomplete
**Date:** 2025
**Purpose:** Monadic operator creation infrastructure

## Summary

The `rpsplug_createmonoper.cc` plugin establishes the foundational infrastructure for creating monadic (unary) operators in RefPerSys's REPL system. While currently incomplete, it demonstrates the approach to operator creation with precedence handling, lifecycle management, and proper class relationships. The plugin creates operator objects with appropriate metadata but lacks the registration logic to integrate them into the REPL's expression evaluation system. Once completed, this plugin will enable sophisticated unary operations like negation, logical NOT, and absolute value with proper operator precedence and monadic semantics, significantly extending the REPL's computational capabilities for single-operand transformations. The plugin represents an important component in RefPerSys's extensible operator system, complementing binary operators with unary operations.