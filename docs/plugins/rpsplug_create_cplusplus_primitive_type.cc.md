# C++ Primitive Type Creation Plugin (`plugins_dir/rpsplug_create_cplusplus_primitive_type.cc`)

**File Path:** `plugins_dir/rpsplug_create_cplusplus_primitive_type.cc`

## Overview

This RefPerSys plugin creates C++ primitive type objects that serve as metadata for code generation systems. These type objects represent fundamental C++ types and provide the necessary information for generating correct C++ code through various backends like GNU Lightning JIT, libgccjit, and static C++ code generation.

## Plugin Purpose

### Code Generation Type System
The plugin enables type-aware code generation by:

- **Type Representation:** Creating objects that represent C++ primitive types
- **Code Generation Support:** Providing type metadata for JIT and static compilation
- **Type Safety:** Ensuring generated code uses correct C++ types
- **Backend Integration:** Supporting multiple code generation backends
- **Type Registry:** Maintaining a catalog of available C++ types

## Implementation Status

### Current State
```cpp
#warning incomplete plugins_dir/rpsplug_create_cplusplus_primitive_type.cc
```

**Status:** Functional but marked as incomplete

### Design Intent
- **Type Objects:** Create RefPerSys objects representing C++ types
- **Code Generation:** Support for multiple compilation backends
- **Type Metadata:** Store type information for code emission
- **System Integration:** Make types available to generation systems

## Command-Line Interface

### Basic Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_create_cplusplus_primitive_type.so \
            --plugin-arg=rpsplug_create_cplusplus_primitive_type:native_int_type \
            --extra=comment='the native int type' \
            --batch --dump=.
```

### Required Arguments
- **`--plugin-arg`**: The type object name to create

### Optional Arguments
- **`--extra=comment`**: Documentation comment for the type
- **`--extra=cplusplus`**: The actual C++ type name (e.g., "int", "long")

## Plugin Architecture

### Entry Point
```cpp
void rps_do_plugin(const Rps_Plugin* plugin)
```

**Execution Flow:**
1. **Name Validation:** Check type object name format and uniqueness
2. **Type Creation:** Instantiate C++ primitive type object
3. **Metadata Setup:** Configure name, comment, and C++ type name
4. **Symbol Creation:** Create associated symbol for referencing
5. **Constant Registration:** Make type object immutable

## Type Name Validation

### Naming Rules
```cpp
bool goodplugarg = isalpha(plugarg[0]);
for (const char* pa = &plugarg[0]; goodplugarg && *pa; pa++)
  goodplugarg = isalnum(*pa) || *pa == '_';

if (!goodplugarg)
  RPS_FATALOUT("bad name");
```

**Requirements:**
- Must start with alphabetic character
- Can contain alphanumeric characters and underscores
- Must not conflict with existing object names

## Core Type Object Creation

### Type Instantiation
```cpp
_f.obcpptype = Rps_ObjectRef::make_object(&_,
    _f.obcppprimtypclass,  // cplusplus_primitive_type∈class
    Rps_ObjectRef::root_space());
```

**Type Properties:**
- Instance of `cplusplus_primitive_type` class
- Placed in root object space
- Gets unique object ID

## Metadata Configuration

### Name Attribute
```cpp
_f.namestr = Rps_Value{std::string(plugarg)};
_f.obcpptype->put_attr(RPS_ROOT_OB(_1EBVGSfW2m200z18rx), _f.namestr);
```

**Self-Naming:** Type object references its own name

### Comment Attribute
```cpp
if (comment) {
  _f.commentstr = Rps_StringValue(comment);
  _f.obcpptype->put_attr(RPS_ROOT_OB(_0jdbikGJFq100dgX1n), _f.commentstr);
}
```

**Documentation:** Types can be documented for future reference

### C++ Type Name Attribute
```cpp
if (cplusplusname && isalpha(cplusplusname[0])) {
  _f.cplusplusstr = Rps_Value{std::string(cplusplusname)};
  _f.obcpptype->put_attr(rpskob_0fx0GtCX90Z03VI9mo, _f.cplusplusstr);
}
```

**Type Mapping:** Maps RefPerSys type object to actual C++ type name

## Symbol Association

### Symbol Creation
```cpp
_f.obsymbol = Rps_ObjectRef::make_new_strong_symbol(&_, std::string{plugarg});
paylsymb->symbol_put_value(_f.obcpptype);
_f.obcpptype->put_attr(RPS_ROOT_OB(_3Q3hJsSgCDN03GTYW5), _f.obsymbol);
```

**Bidirectional Linking:**
- Symbol resolves to type object
- Type object references its symbol

## Constant Registration

### Immutability
```cpp
rps_add_constant_object(&_, _f.obcpptype);
```

**Constant Benefits:** Type objects become immutable system constants

## Integration with Code Generation

### Backend Support
Created type objects integrate with:

- **GNU Lightning:** `lightgen_rps.cc` - Dynamic code generation
- **libgccjit:** `gccjit_rps.cc` - GCC JIT compilation
- **Static C++:** `cppgen_rps.cc` - Static code generation
- **Type System:** Native data loading in `load_rps.cc`

### Type Information Usage
```cpp
// Code generators can access:
auto cppTypeName = typeObj->get_attr(cplusplus_name_attr);
auto typeComment = typeObj->get_attr(comment_attr);
// Use for generating correct C++ type declarations
```

## Use Cases

### Code Generation Types
- **Primitive Types:** int, long, char, float, double, etc.
- **Pointer Types:** void*, char*, int*, etc.
- **Platform Types:** size_t, ptrdiff_t, uintptr_t, etc.
- **Custom Types:** Domain-specific primitive types

### Backend-Specific Types
- **JIT Types:** Types needed for dynamic code generation
- **Static Types:** Types for compiled code output
- **Platform Types:** Architecture-specific type representations

## Build and Deployment

### Compilation
```bash
cd $REFPERSYS_TOPDIR
make plugins_dir/rpsplug_create_cplusplus_primitive_type.so
/bin/ln -sfv $(/bin/pwd)/plugins_dir/rpsplug_create_cplusplus_primitive_type.so /tmp/
```

### Runtime Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_create_cplusplus_primitive_type.so \
            --plugin-arg=rpsplug_create_cplusplus_primitive_type:native_int_type \
            --extra=comment='the native int type' \
            --extra=cplusplus=int \
            --batch --dump=.
```

## Example Usage

### Creating Basic Types
```bash
# Integer type
./refpersys --plugin-after-load=/tmp/rpsplug_create_cplusplus_primitive_type.so \
            --plugin-arg=rpsplug_create_cplusplus_primitive_type:int_type \
            --extra=comment='32-bit signed integer' \
            --extra=cplusplus=int \
            --batch --dump=.

# Pointer type
./refpersys --plugin-after-load=/tmp/rpsplug_create_cplusplus_primitive_type.so \
            --plugin-arg=rpsplug_create_cplusplus_primitive_type:void_ptr_type \
            --extra=comment='generic pointer type' \
            --extra=cplusplus='void*' \
            --batch --dump=.
```

### Creating Platform Types
```bash
# Size type
./refpersys --plugin-after-load=/tmp/rpsplug_create_cplusplus_primitive_type.so \
            --plugin-arg=rpsplug_create_cplusplus_primitive_type:size_type \
            --extra=comment='size_t type for sizes' \
            --extra=cplusplus=size_t \
            --batch --dump=.
```

## Performance Characteristics

### Execution Time
- **Validation:** Fast string parsing and uniqueness checking
- **Object Creation:** Standard RefPerSys instantiation
- **Symbol Operations:** Hash table operations
- **Registration:** Minimal system integration overhead

### Memory Usage
- **Type Object:** Standard object overhead
- **Symbol Object:** Additional symbol structure
- **Metadata:** Name, comment, and C++ type name strings
- **Constant Registration:** Efficient system integration

## Relationship to Other Plugins

### Complementary Plugins
- **`rpsplug_cplusplustypes.cc`**: Populates type metadata (sizes, alignments)
- **Code Generation Plugins:** Use type objects for code emission
- **Class Creation Plugins:** Create classes that use these types

### Type System Workflow
```
Create Type Objects → Populate Metadata → Use in Code Gen
     ↓                      ↓                    ↓
Type Registry        Size/Align Data      Generated Code
```

## Thread Safety

### Synchronization
Plugin uses appropriate locking for object creation and attribute manipulation, ensuring thread-safe operation in the RefPerSys environment.

## Error Handling

### Validation Failures
- **Missing Name:** Requires type object name
- **Invalid Name:** Must follow identifier rules
- **Name Conflicts:** Cannot reuse existing names

### Fatal Errors
```cpp
RPS_FATALOUT("without argument; should be some non-empty name");
RPS_FATALOUT("naming an existing object");
```

## File Status

**Status:** Active/Incomplete
**Date:** 2024-2025
**Purpose:** C++ primitive type object creation

## Summary

The `rpsplug_create_cplusplus_primitive_type.cc` plugin provides essential functionality for creating C++ primitive type objects that serve as the foundation for RefPerSys's code generation capabilities. By creating type objects with associated symbols and metadata, it enables type-aware code generation across multiple backends. The plugin creates the structural elements needed for type-safe code generation, though it notes that additional metadata population (sizes, alignments) is handled by complementary plugins. This plugin represents a crucial component in RefPerSys's code generation infrastructure, providing the type system foundation that enables sophisticated compilation and JIT code generation capabilities. The integration with multiple code generation backends makes it a key enabler for RefPerSys's ability to generate efficient, type-correct C++ code through various compilation strategies.