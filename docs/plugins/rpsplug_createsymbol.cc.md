# Symbol Creation Plugin (`plugins_dir/rpsplug_createsymbol.cc`)

**File Path:** `plugins_dir/rpsplug_createsymbol.cc`

## Overview

This RefPerSys plugin creates new symbol objects in the system. Symbols are fundamental to RefPerSys's object system, providing named references to objects and serving as the basis for attributes, classes, and other named constructs. The plugin enables dynamic extension of the system's symbol namespace.

## Plugin Purpose

### Symbol System Extension
The plugin enables dynamic extension of RefPerSys's naming system by:

- **Symbol Creation:** Instantiating new symbol objects
- **Name Registration:** Adding symbols to the global namespace
- **Attribute Support:** Creating symbols for use as attributes
- **Lifecycle Management:** Supporting root, constant, or plain symbols
- **System Integration:** Making symbols available throughout the system

## Implementation Status

### Current State
```cpp
#warning "is buggy at commit 80608c7722cf"
```

**Status:** Functional but with known bugs at specific commit

### Design Intent
- **Symbol Instantiation:** Create named symbol objects
- **Namespace Extension:** Add to global symbol table
- **Lifecycle Control:** Support different symbol persistence levels
- **System Integration:** Make symbols available for object referencing

## Command-Line Interface

### Basic Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_createsymbol.so \
            --plugin-arg=rpsplug_createsymbol:new_symbol_name \
            --extra=comment='some comment' \
            --extra=rooted=0 \
            --extra=constant=1 \
            --batch --dump=.
```

### Required Arguments
- **`--plugin-arg`**: The symbol name to create

### Optional Arguments
- **`--extra=comment`**: Documentation comment for the symbol
- **`--extra=rooted`**: Make symbol a root object (true/1)
- **`--extra=constant`**: Make symbol a constant object (true/1)

## Plugin Architecture

### Entry Point
```cpp
void rps_do_plugin(const Rps_Plugin* plugin)
```

**Execution Flow:**
1. **Name Validation:** Check symbol name format and uniqueness
2. **Lifecycle Determination:** Parse rooted/constant flags
3. **Symbol Creation:** Instantiate symbol object
4. **Metadata Setup:** Configure name and comment attributes
5. **Registration:** Add to appropriate system collections

## Symbol Name Validation

### Naming Rules
```cpp
bool goodplugarg = isalpha(plugarg[0]);
for (const char* pa = &plugarg[0]; goodplugarg && *pa; pa++)
  goodplugarg = isalnum(*pa) || *pa == '_';

if (!goodplugarg)
  RPS_FATALOUT("with bad name " << Rps_QuotedC_String(plugarg));
```

**Requirements:**
- Must start with alphabetic character
- Can contain alphanumeric characters and underscores
- Must not conflict with existing object names

## Lifecycle Configuration

### Root vs Constant Determination
```cpp
if (rooted && rooted[0]) {
  if (!strcmp(rooted, "true")) isrooted = true;
  else if (atoi(rooted) > 0) isrooted = true;
}

if (constant && constant[0]) {
  if (!strcmp(constant, "true")) isconstant = true;
  else if (atoi(constant) > 0) isconstant = true;
}
```

**Lifecycle Options:**
- **Root:** Protected from GC, mutable
- **Constant:** Protected from GC, immutable
- **Plain:** Not protected from GC, mutable

## Core Symbol Creation

### Symbol Instantiation
```cpp
_f.obsymbol = Rps_ObjectRef::make_new_strong_symbol(&_, std::string{plugarg});
```

**Symbol Properties:**
- Strong symbol (persistent reference)
- Unique in symbol namespace
- Gets unique object ID

## Metadata Configuration

### Name Attribute
```cpp
_f.namestr = Rps_Value{std::string(plugarg)};
_f.obsymbol->put_attr(RPS_ROOT_OB(_1EBVGSfW2m200z18rx), _f.namestr);
```

**Self-Naming:** Symbol references its own name

### Comment Attribute
```cpp
if (comment && comment[0]) {
  _f.commentstr = Rps_Value{std::string(comment)};
  _f.obsymbol->put_attr(RPS_ROOT_OB(_0jdbikGJFq100dgX1n), _f.commentstr);
}
```

**Documentation:** Symbols can be documented for future reference

## Registration System

### Root Registration
```cpp
if (isrooted) {
  rps_add_root_object(_f.obsymbol);
  RPS_INFORMOUT("added new root symbol " << _f.obsymbol);
}
```

**Root Benefits:** Survives garbage collection, always accessible

### Constant Registration
```cpp
else if (isconstant) {
  rps_add_constant_object(&_, _f.obsymbol);
  RPS_INFORMOUT("added new constant symbol " << _f.obsymbol);
}
```

**Constant Benefits:** Immutable, system-wide availability

### Plain Symbol
```cpp
else {
  RPS_INFORMOUT("added new plain symbol " << _f.obsymbol);
}
```

**Plain Symbols:** Subject to garbage collection

## Integration with Symbol System

### Symbol Resolution
Created symbols integrate with:
- **Attribute System:** Used as attribute keys
- **Class System:** Define class relationships
- **Object References:** Named object access
- **Global Namespace:** Available throughout system

## Use Cases

### Attribute Definition
- **Custom Attributes:** Define new object properties
- **Metadata Keys:** Create keys for object metadata
- **Relationship Types:** Define relationship classifications

### System Extension
- **New Classes:** Symbols for class definitions
- **Constants:** Named constant references
- **Configuration:** System configuration keys

### Development Support
- **Debug Symbols:** Named references for debugging
- **Test Fixtures:** Named objects for testing
- **Prototype Systems:** Experimental symbol definitions

## Build and Deployment

### Compilation
```bash
cd $REFPERSYS_TOPDIR
make plugins_dir/rpsplug_createsymbol.so
/bin/ln -svf $(/bin/pwd)/plugins_dir/rpsplug_createsymbol.so /tmp/
```

### Runtime Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_createsymbol.so \
            --plugin-arg=rpsplug_createsymbol:my_attribute \
            --extra=comment='Custom attribute for my objects' \
            --extra=constant=1 \
            --batch --dump=.
```

## Example Usage

### Creating Attribute Symbols
```bash
# Priority attribute
./refpersys --plugin-after-load=/tmp/rpsplug_createsymbol.so \
            --plugin-arg=rpsplug_createsymbol:include_priority \
            --extra=comment='attribute for C++ include priority (an integer)' \
            --extra=rooted=0 \
            --extra=constant=1 \
            --batch --dump=.

# Color attribute
./refpersys --plugin-after-load=/tmp/rpsplug_createsymbol.so \
            --plugin-arg=rpsplug_createsymbol:color \
            --extra=comment='color attribute for visual objects' \
            --extra=constant=1 \
            --batch --dump=.
```

### Creating Class Symbols
```bash
# Widget class symbol
./refpersys --plugin-after-load=/tmp/rpsplug_createsymbol.so \
            --plugin-arg=rpsplug_createsymbol:widget_class \
            --extra=comment='symbol for widget class definition' \
            --extra=rooted=1 \
            --batch --dump=.
```

## Performance Characteristics

### Execution Time
- **Validation:** Fast string parsing and uniqueness checking
- **Symbol Creation:** Efficient symbol table operations
- **Registration:** Minimal system integration overhead

### Memory Usage
- **Symbol Object:** Standard object overhead
- **Symbol Table Entry:** Hash table storage
- **Attribute Storage:** Name and comment strings
- **Registration:** Minimal additional system state

## Relationship to Other Plugins

### Complementary Plugins
- **Class Creation:** Use symbols for class definitions
- **Attribute Plugins:** Create symbols for attributes
- **Object Creation:** Use symbols for naming

### Symbol Ecosystem
```
Symbols ← Attributes ← Objects ← Classes
    ↓         ↓          ↓        ↓
Names    Properties   Instances  Types
```

## Thread Safety

### Synchronization
Plugin uses RefPerSys's symbol system which provides appropriate synchronization for thread-safe symbol creation and registration.

## Error Handling

### Validation Failures
- **Missing Name:** Requires symbol name
- **Invalid Name:** Must follow identifier rules
- **Name Conflicts:** Cannot reuse existing names

### Fatal Errors
```cpp
RPS_FATALOUT("without argument; should be some non-empty name");
RPS_FATALOUT("with bad name " << Rps_QuotedC_String(plugarg));
```

## Known Issues

### Bug Report
```cpp
#warning "is buggy at commit 80608c7722cf"
/*** buggy invocation at commit 80608c7722cf Sun Jun 30 06:22 PM CEST 2024
./refpersys --plugin-after-load=/tmp/rpsplug_createsymbol.so \
  --plugin-arg=rpsplug_createsymbol:include_priority \
  --extra=comment='attribute for C++ include priority (an integer)' \
  --extra=rooted=0  --extra=constant=1  --batch --dump=.
***/
```

**Issue:** Known bug at specific commit affecting symbol creation

## File Status

**Status:** Active/With Known Issues
**Date:** 2023-2025
**Purpose:** Symbol creation and registration

## Summary

The `rpsplug_createsymbol.cc` plugin provides essential functionality for extending RefPerSys's symbol system. By creating new symbol objects with configurable lifecycles (root, constant, or plain), it enables dynamic extension of the system's naming and attribute infrastructure. Despite a known bug at a specific commit, the plugin successfully creates symbols and integrates them into the appropriate system collections. Symbols created by this plugin serve as the foundation for attributes, class relationships, and named references throughout the RefPerSys system, making this plugin crucial for system extension and customization. The plugin demonstrates the sophisticated lifecycle management capabilities of RefPerSys's object system.