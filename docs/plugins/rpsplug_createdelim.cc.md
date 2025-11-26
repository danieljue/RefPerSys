# REPL Delimiter Creation Plugin (`plugins_dir/rpsplug_createdelim.cc`)

**File Path:** `plugins_dir/rpsplug_createdelim.cc`

## Overview

This RefPerSys plugin creates delimiter objects for the REPL (Read-Eval-Print Loop) system. Delimiters are special tokens used in expression parsing to separate elements, provide structure, and control evaluation order. The plugin enables extension of the REPL's lexical analysis capabilities with custom delimiters.

## Plugin Purpose

### REPL Lexical Extension
The plugin enables dynamic extension of REPL parsing by:

- **Delimiter Creation:** Defining new delimiter tokens
- **Dictionary Registration:** Adding delimiters to the system dictionary
- **Symbol Association:** Creating symbolic names for delimiters
- **Parser Integration:** Making delimiters available to expression parsing
- **Lexical Analysis:** Supporting structured expression evaluation

## Command-Line Interface

### Basic Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_createdelim.so \
            --plugin-arg=rpsplug_createdelim:++ \
            --extra=name=plusplus \
            --extra=comment='increment delimiter' \
            --batch --dump=.
```

### Required Arguments
- **`--plugin-arg`**: The delimiter symbol (e.g., "++", "--", "->")

### Optional Arguments
- **`--extra=name`**: Symbolic name for the delimiter
- **`--extra=comment`**: Documentation comment

## Plugin Architecture

### Entry Point
```cpp
void rps_do_plugin(const Rps_Plugin* plugin)
```

**Execution Flow:**
1. **Argument Validation:** Check delimiter symbol format
2. **Delimiter Creation:** Instantiate delimiter object
3. **Dictionary Registration:** Add to delimiter dictionary
4. **Symbol Setup:** Create associated symbol (if named)
5. **Constant Registration:** Make delimiter immutable

## Delimiter Symbol Validation

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

if (!plugarg[0] && !argispunct && !argisident)
  RPS_FATALOUT("bad argument not identifier or all-delim");
```

**Valid Formats:**
- **Punctuation Delimiters:** All characters must be punctuation (e.g., "++", "--", "->")
- **Identifier Delimiters:** All characters must form valid identifier (e.g., "begin", "end")

## Core Object Creation

### Delimiter Instantiation
```cpp
_f.obdelim = Rps_ObjectRef::make_object(&_,
    _f.obclassrepldelim,  // repl_delimiter∈class
    Rps_ObjectRef::root_space());
```

**Delimiter Properties:**
- Instance of `repl_delimiter` class
- Placed in root object space
- Gets unique object ID

## Dictionary Registration

### Delimiter Dictionary
```cpp
_f.obdictdelim = RPS_ROOT_OB(_627ngdqrVfF020ugC5); // "repl_delim"∈string_dictionary
auto paylstrdict = _f.obdictdelim->get_dynamic_payload<Rps_PayloadStringDict>();
paylstrdict->add(plugarg, _f.obdelim);
```

**Dictionary Integration:** Delimiters are stored in a string-keyed dictionary for fast lookup

## Symbol Association (Optional)

### Named Delimiters
```cpp
if (xtraname && isalpha(xtraname[0])) {
  if (!Rps_PayloadSymbol::valid_name(xtraname))
    RPS_FATALOUT("invalid name");

  _f.obold = Rps_PayloadSymbol::find_named_object(xtraname);
  if (_f.obold)
    RPS_FATALOUT("name already used");

  _f.obsymbol = Rps_ObjectRef::make_new_strong_symbol(&_, xtraname);
  // ... bidirectional linking setup
}
```

**Symbol Benefits:**
- Named access to delimiters
- Symbolic references in code
- Integration with symbol system

## Metadata Configuration

### Delimiter Attribute
```cpp
_f.obdelim->put_attr(RPS_ROOT_OB(_2wdmxJecnFZ02VGGFK), _f.strdelim);
```

**Self-Reference:** Delimiter object references its own string value

### Comment Addition
```cpp
if (comment && comment[0]) {
  _f.strcomment = Rps_StringValue(comment);
  _f.obsymbol->put_attr(RPS_ROOT_OB(_0jdbikGJFq100dgX1n), _f.strcomment);
}
```

**Documentation:** Symbols can be documented for future reference

## Constant Registration

### Immutability
```cpp
rps_add_constant_object(&_, _f.obdelim);
```

**Constant Benefits:** Delimiters become immutable system constants

## Integration with REPL Parser

### Lexical Analysis
The created delimiters are used by:
- **Tokenization:** Breaking input into tokens
- **Parsing:** Structuring expressions
- **Evaluation:** Controlling evaluation order
- **Syntax:** Providing grammatical structure

### Carburetta Integration
```cpp
// See also carbrepl_rps.cbrt file, its routine
// rps_initialize_carburetta_after_load
```

**Parser Connection:** Delimiters integrate with the Carburetta parser system

## Use Cases

### Expression Structure
- **Grouping:** Parentheses, brackets, braces
- **Statements:** Semicolons, statement separators
- **Blocks:** Begin/end delimiters
- **Lists:** Comma separators, array delimiters

### Language Extensions
- **Custom Syntax:** Domain-specific delimiters
- **DSL Support:** Specialized notation delimiters
- **Macro Systems:** Template delimiters
- **Data Formats:** Structured data delimiters

## Build and Deployment

### Compilation
```bash
cd $REFPERSYS_TOPDIR
./do-build-refpersys-plugin -v \
  -i plugins_dir/rpsplug_createdelim.cc \
  -o plugins_dir/rpsplug_createdelim.so \
  -L /tmp/rpsplug_createdelim.so
```

### Runtime Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_createdelim.so \
            --plugin-arg=rpsplug_createdelim:-> \
            --extra=name=arrow \
            --extra=comment='arrow delimiter for function types' \
            --batch --dump=.
```

## Example Usage

### Creating Structural Delimiters
```bash
# Parentheses
./refpersys --plugin-after-load=/tmp/rpsplug_createdelim.so \
            --plugin-arg=rpsplug_createdelim:( \
            --extra=name=left_paren \
            --batch --dump=.

# Brackets
./refpersys --plugin-after-load=/tmp/rpsplug_createdelim.so \
            --plugin-arg=rpsplug_createdelim:[ \
            --extra=name=left_bracket \
            --batch --dump=.
```

### Creating Statement Delimiters
```bash
# Semicolon
./refpersys --plugin-after-load=/tmp/rpsplug_createdelim.so \
            --plugin-arg=rpsplug_createdelim:; \
            --extra=name=semicolon \
            --extra=comment='statement terminator' \
            --batch --dump=.
```

## Performance Characteristics

### Execution Time
- **Validation:** Fast string parsing and format checking
- **Object Creation:** Standard RefPerSys instantiation
- **Dictionary Operations:** Efficient string-keyed dictionary updates
- **Symbol Operations:** Hash table operations

### Memory Usage
- **Delimiter Object:** Standard object overhead
- **Dictionary Entry:** Minimal additional storage
- **Symbol Object:** Additional naming structure (optional)
- **Constant Registration:** Efficient system integration

## Relationship to Other Plugins

### Complementary Plugins
- **Operator Plugins:** Work together for complete expression syntax
- **Parser Integration:** Delimiters support operator parsing
- **Language Extensions:** Enable custom syntax definitions

### REPL Ecosystem
```
Delimiters ← Operators ← Parser ← Expressions
     ↓          ↓         ↓         ↓
Tokens      Evaluation  Syntax    Results
```

## Thread Safety

### Synchronization
Plugin uses appropriate locking for dictionary and symbol operations:
- Dictionary access for delimiter registration
- Symbol creation and linking
- Object attribute modifications

## Error Handling

### Validation Failures
- **Missing Argument:** Requires delimiter symbol
- **Invalid Format:** Must be valid delimiter format
- **Name Conflicts:** Cannot reuse existing symbol names
- **Invalid Names:** Symbol names must be valid identifiers

### Fatal Errors
```cpp
RPS_FATALOUT("without argument; should be some non-empty string");
RPS_FATALOUT("with bad argument not identifier or all-delim");
```

## File Status

**Status:** Active/Production
**Date:** 2025
**Purpose:** REPL delimiter creation and registration

## Summary

The `rpsplug_createdelim.cc` plugin provides essential functionality for extending RefPerSys's REPL lexical analysis capabilities. By creating delimiter objects and registering them in the system's delimiter dictionary, it enables structured expression parsing and evaluation. The plugin supports both anonymous delimiters and named delimiters with associated symbols, providing flexibility in delimiter management. This plugin is crucial for building sophisticated expression evaluation systems, supporting everything from basic grouping delimiters to complex domain-specific syntax. The integration with the Carburetta parser system ensures that delimiters work seamlessly with the broader REPL infrastructure, enabling rich and extensible expression languages within RefPerSys.