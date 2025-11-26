# C++ Types Metadata Plugin (`plugins_dir/rpsplug_cplusplustypes.cc`)

**File Path:** `plugins_dir/rpsplug_cplusplustypes.cc`

## Overview

This RefPerSys plugin populates C++ primitive type objects with essential metadata including size and alignment information. The plugin uses compile-time `sizeof` and `alignof` operators to gather platform-specific type information and stores this data in RefPerSys type objects for use by code generation backends.

## Plugin Purpose

### Type Metadata Population
The plugin enables type-aware code generation by:

- **Size Information:** Recording byte sizes of C++ types
- **Alignment Data:** Storing alignment requirements
- **Type Mapping:** Connecting RefPerSys types to C++ types
- **Platform Adaptation:** Using actual platform type characteristics
- **Code Generation Support:** Providing type metadata for backends

## Command-Line Interface

### Basic Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_cplusplustypes.so \
            --batch --dump=.
```

### Arguments
- **No plugin arguments required** - operates on predefined type objects
- Uses batch mode for system updates
- Typically followed by dump for persistence

## Plugin Architecture

### Entry Point
```cpp
void rps_do_plugin(const Rps_Plugin* plugin)
```

**Execution Flow:**
1. **Type Processing:** Iterate through predefined type mappings
2. **Metadata Population:** Add size and alignment to type objects
3. **Validation:** Ensure types exist and are not already populated
4. **Information Logging:** Report successful updates

## Type Processing Function

### Metadata Population
```cpp
void rpscplusplustype(Rps_CallFrame* callframe, const char* obid,
                     const char* cppname, int size, int align)
```

**Parameters:**
- `obid`: RefPerSys object ID string
- `cppname`: C++ type name
- `size`: Byte size from `sizeof()`
- `align`: Alignment from `alignof()`

**Processing Steps:**
1. **Object Lookup:** Find type object by ID
2. **Type Validation:** Ensure it's a C++ primitive type
3. **Duplicate Check:** Skip if already populated
4. **Metadata Storage:** Set C++ name, size, and alignment

## Type Mappings

### Predefined Types
```cpp
#define RPS_TYPE_CPLUSPLUS(Id,Cnam) \
    rpscplusplustype(&_, #Id, #Cnam, sizeof(Cnam), alignof(Cnam))

RPS_TYPE_CPLUSPLUS(_4V1oeUOvmxo041XLTm, intptr_t);
RPS_TYPE_CPLUSPLUS(_4nZ0jIKUbGr01OixPV, int);
RPS_TYPE_CPLUSPLUS(_3NYlqvmSuTm024LDuD, long);
```

**Mapped Types:**
- **`intptr_t`**: Integer type capable of holding pointers
- **`int`**: Standard integer type
- **`long`**: Long integer type

## Metadata Storage

### C++ Name Attribute
```cpp
_f.str = Rps_Value(std::string(cppname));
_f.obty->put_attr(rpskob_0fx0GtCX90Z03VI9mo, _f.str);
```

**Type Identification:** Maps RefPerSys object to C++ type name

### Size Attribute
```cpp
if (size > 0)
  _f.obty->put_attr(rpskob_8IRzlYX53kN00tC3fG,
                   Rps_Value::make_tagged_int(size));
```

**Size Information:** Stores byte size as tagged integer

### Alignment Attribute
```cpp
if (align > 0)
  _f.obty->put_attr(rpskob_6EsfxShTuwH00tC3fG,
                   Rps_Value::make_tagged_int(align));
```

**Alignment Requirements:** Stores alignment as tagged integer

## Integration with Code Generation

### Backend Usage
Created metadata integrates with:

- **GNU Lightning:** `lightgen_rps.cc` - Dynamic code generation
- **libgccjit:** `gccjit_rps.cc` - GCC JIT compilation
- **Static C++:** `cppgen_rps.cc` - Static code generation
- **Type System:** Platform-specific type handling

### Metadata Access
```cpp
// Code generators can access:
auto cppName = typeObj->get_attr(cplusplus_name_attr);
auto byteSize = typeObj->get_attr(byte_size_attr);
auto alignment = typeObj->get_attr(byte_alignment_attr);
// Use for generating correct C++ type declarations
```

## Use Cases

### Platform-Specific Code
- **Cross-Platform:** Generate code that adapts to platform types
- **Size-Aware:** Create data structures with correct sizes
- **Alignment-Safe:** Ensure proper memory alignment
- **Type Portability:** Abstract platform differences

### Code Generation Scenarios
- **Struct Layout:** Generate C++ structs with correct field sizes
- **Memory Management:** Allocate memory with proper alignment
- **Type Casting:** Generate safe type conversions
- **ABI Compliance:** Ensure platform-specific ABI requirements

## Build and Deployment

### Compilation
```bash
cd $REFPERSYS_TOPDIR
./do-build-refpersys-plugin -v \
  -i plugins_dir/rpsplug_cplusplustypes.cc \
  -o plugins_dir/rpsplug_cplusplustypes.so \
  -L /tmp/rpsplug_cplusplustypes.so
```

### Runtime Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_cplusplustypes.so \
            --batch --dump=.
```

## Example Output

### Type Information Population
```
updated
object cplusplus_primitive_type #4V1oeUOvmxo041XLTm (native_intptr_type)
  cplusplus_name: "intptr_t"
  byte_size: 8
  byte_alignment: 8

updated
object cplusplus_primitive_type #4nZ0jIKUbGr01OixPV (native_int_type)
  cplusplus_name: "int"
  byte_size: 4
  byte_alignment: 4

updated
object cplusplus_primitive_type #3NYlqvmSuTm024LDuD (native_long_type)
  cplusplus_name: "long"
  byte_size: 8
  byte_alignment: 8
```

## Performance Characteristics

### Execution Time
- **Type Lookup:** Fast object ID resolution
- **Attribute Setting:** Minimal metadata storage operations
- **Validation:** Quick existence and type checks

### Memory Usage
- **Attribute Storage:** Small additional metadata per type
- **String Storage:** C++ type name strings
- **Integer Storage:** Tagged integers for size/alignment

## Relationship to Other Plugins

### Complementary Plugins
- **`rpsplug_create_cplusplus_primitive_type.cc`**: Creates the type objects
- **Code Generation Plugins:** Use populated metadata
- **Loader Integration:** `rps_set_native_data_in_loader` uses this data

### Type System Workflow
```
Create Type Objects → Populate Metadata → Use in Code Gen
     ↓                      ↓                    ↓
Type Registry        Size/Align Data      Generated Code
```

## Thread Safety

### Synchronization
Plugin uses RefPerSys's existing object access patterns which include appropriate locking for thread-safe attribute setting.

## Error Handling

### Validation Checks
- **Object Existence:** Type objects must exist
- **Type Validation:** Must be C++ primitive type instances
- **Duplicate Prevention:** Skip already populated types

### Warning for Duplicates
```cpp
if (_f.obty->get_physical_attr(cplusplus_name_attr)) {
  RPS_WARNOUT("already filled obty: " << RPS_OBJECT_DISPLAY(_f.obty));
  return;
}
```

**Idempotent Operation:** Safe to run multiple times

## Extensibility

### Adding New Types
To add more C++ types, extend the macro usage:

```cpp
RPS_TYPE_CPLUSPLUS(_new_type_id, char);
RPS_TYPE_CPLUSPLUS(_another_id, double);
// etc.
```

**Type Expansion:** Easy to add new primitive types

## File Status

**Status:** Active/Production
**Date:** 2025
**Purpose:** C++ primitive type metadata population

## Summary

The `rpsplug_cplusplustypes.cc` plugin provides essential functionality for populating C++ primitive type objects with platform-specific metadata. By using compile-time `sizeof` and `alignof` operators, it captures accurate size and alignment information for various C++ types and stores this data in RefPerSys objects. This metadata is crucial for code generation backends that need to generate correct, platform-specific C++ code. The plugin serves as a bridge between RefPerSys's abstract type system and the concrete realities of C++ type characteristics, enabling sophisticated code generation that respects platform-specific requirements for size, alignment, and type representation. This plugin represents a critical component in RefPerSys's code generation infrastructure, ensuring that generated C++ code is not only syntactically correct but also adheres to platform-specific ABI and memory layout requirements.