# ANTLR Grammar Experiment (`attic/gramrepl_antlr_rps.g4`)

**File Path:** `attic/gramrepl_antlr_rps.g4`

## Overview

This file contains an experimental ANTLR4 grammar for RefPerSys REPL (Read-Eval-Print Loop) commands. It represents an attempt to replace or supplement the Bison-based parser with ANTLR, a popular parser generator that supports multiple target languages and has different parsing strategies.

## Grammar Structure

### Grammar Declaration
```antlr
grammar gramrepl_antlr_rps;

// the start symbol is the REPL command
```

**Key Characteristics:**
- ANTLR4 grammar format (.g4 extension)
- Focus on REPL command parsing
- Alternative to existing Bison grammar

## REPL Command Syntax

### Command Types
```antlr
repl_command : 'show' val_expr
             | 'in' obj_expr 'put' obj_expr ':' val_expr
             | 'in' obj_expr 'rem' obj_expr
             | 'in' obj_expr 'append' val_expr
             ;
```

**Supported Commands:**
1. **Show Command:** `show <value_expression>` - Display values
2. **Put Command:** `in <object> put <attribute> : <value>` - Set object attributes
3. **Remove Command:** `in <object> rem <attribute>` - Remove object attributes
4. **Append Command:** `in <object> append <value>` - Append to collections

## Value Expressions

### Expression Types
```antlr
val_expr : INT
         | DOUBLE
         | obj_expr
         ;
```

**Value Types:**
- **INT:** Integer literals `[0-9]+`
- **DOUBLE:** Floating-point literals `[0-9]+ '.' [0-9]+`
- **obj_expr:** Object references (see below)

**Note:** STRING literals mentioned but not implemented

## Object Expressions

### Object References
```antlr
obj_expr: OBJID
        | NAME
        ;
```

**Object Identification:**
- **OBJID:** Internal object IDs starting with underscore `_[A-Za-z0-9]*`
- **NAME:** Named objects `[A-Z][a-z][A-Za-z0-9]*`

## Terminal Definitions

### Literals and Identifiers
```antlr
INT: [0-9]+ ;
DOUBLE: [0-9]+ '.' [0-9]+ ;
OBJID: '_' [A-Za-z0-9]* ;
NAME: [A-Z][a-z][A-Za-z0-9]* ;
```

**Token Patterns:**
- **INT:** One or more digits
- **DOUBLE:** Digits, decimal point, digits
- **OBJID:** Underscore followed by alphanumeric characters
- **NAME:** Capital letter, lowercase letter, then alphanumeric

## Embedded C++ Actions

### Action Structure
Each grammar rule contains embedded C++ code in braces:

```antlr
repl_command : 'show' val_expr
{
/* C++ code for show REPL */
#warning missing C++ code for show command (ANTLR)
}
```

**Action Characteristics:**
- C preprocessor directives (`#warning`)
- Comments indicating missing implementation
- Placeholder for semantic actions

### Implementation Status
All semantic actions contain warnings:
```cpp
#warning missing C++ code for show command (ANTLR)
#warning missing C++ code for in ... put ... ':' ... command (ANTLR)
// ... etc for all rules
```

**Status:** Grammar structure complete, semantic actions unimplemented

## Comparison with Bison Grammar

### Similarities
- **Command Syntax:** Same REPL command structure
- **Value Types:** INT, DOUBLE, object references
- **Object Identification:** OBJID and NAME patterns

### Differences
- **Parser Generator:** ANTLR4 vs Bison
- **Parsing Strategy:** LL(*) vs LALR(1)
- **Target Languages:** Multi-language vs C/C++ focus
- **Integration:** Different runtime libraries

### ANTLR Advantages Explored
1. **Multi-Language Support:** Could generate parsers for different languages
2. **Better Error Messages:** ANTLR typically provides clearer error reporting
3. **IDE Integration:** Better tool support and IDE integration
4. **Grammar Debugging:** ANTLR provides grammar analysis tools

## Implementation Gaps

### Missing Features
1. **STRING Literals:** Mentioned in comments but not implemented
2. **Semantic Actions:** All C++ actions are placeholder warnings
3. **Error Handling:** No error recovery rules defined
4. **Integration Code:** No ANTLR runtime integration

### Token Limitations
- **OBJID Pattern:** `[A-Za-z0-9]*` allows digits first, unlike typical RefPerSys OIDs
- **NAME Pattern:** Requires specific capitalization pattern
- **No Whitespace Handling:** Implicit whitespace handling

## Historical Context

### Experiment Goals
This ANTLR experiment likely aimed to:
1. **Evaluate Alternatives:** Compare ANTLR with existing Bison implementation
2. **Multi-Language Support:** Enable parsing in different target languages
3. **Modern Tooling:** Leverage ANTLR's ecosystem and tools
4. **Grammar Analysis:** Use ANTLR's grammar validation capabilities

### Outcome
- **Grammar Structure:** Successfully defined command syntax
- **ANTLR Integration:** Basic grammar structure established
- **Implementation:** Stopped at grammar definition phase
- **Adoption:** Not integrated into main RefPerSys codebase

## File Status

**Status:** Experimental/Abandoned
**Date:** 2023
**Reason for attic:** Alternative parser approach not pursued

## Technical Analysis

### Grammar Completeness
- **Syntax Definition:** Complete command structure
- **Token Specification:** Well-defined terminal patterns
- **Rule Structure:** Proper ANTLR4 grammar format

### Integration Requirements
To complete this experiment, would need:
1. **ANTLR Runtime:** Link with ANTLR C++ runtime library
2. **Semantic Actions:** Implement C++ code for each grammar action
3. **Error Handling:** Add error recovery and reporting
4. **Build Integration:** Modify build system for ANTLR code generation

### Potential Benefits
1. **Better Errors:** ANTLR's error recovery and reporting
2. **Tool Support:** ANTLR's grammar development tools
3. **Multi-Target:** Generate parsers for other languages
4. **IDE Support:** Better development environment integration

This ANTLR grammar experiment demonstrates RefPerSys's exploration of alternative parsing technologies, providing a foundation for ANTLR-based parsing that could be completed if the project needed multi-language parser generation or enhanced error reporting capabilities.