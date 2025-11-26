# C++ Code Class Creation Plugin (`plugins_dir/rpsplug_create_cplusplus_code_class.cc`)

**File Path:** `plugins_dir/rpsplug_create_cplusplus_code_class.cc`

## Overview

This RefPerSys plugin creates C++ code classes that serve as templates for code generation. These classes inherit from specified superclasses and are designed to be used by RefPerSys's code generation backends to produce C++ code. The plugin enables the creation of class hierarchies that can be translated into C++ class definitions.

## Plugin Purpose

### Code Generation Class System
The plugin enables structured code generation by:

- **Class Hierarchy Creation:** Building inheritance-based class structures
- **Code Generation Templates:** Creating classes that generate C++ code
- **Type System Integration:** Connecting with C++ type objects
- **Backend Support:** Providing classes for multiple code generation systems
- **Class Registry:** Maintaining catalogs of generatable classes

## Command-Line Interface

### Basic Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_create_cplusplus_code_class.so \
            --plugin-arg=rpsplug_create_cplusplus_code_class:new_class_name \
            --extra=super=superclass \
            --extra=comment='some comment' \
            --batch --dump=.
```

### Required Arguments
- **`--plugin-arg`**: The class name to create
- **`--extra=super`**: The superclass name to inherit from

### Optional Arguments
- **`--extra=comment`**: Documentation comment for the class
- **`--extra=instance`**: Mark as instance class (true/yes/1)

## Plugin Architecture

### Entry Point
```cpp
void rps_do_plugin(const Rps_Plugin* plugin)
```

**Execution Flow:**
1. **Name Validation:** Check class name format and uniqueness
2. **Superclass Resolution:** Find and validate superclass
3. **Class Creation:** Instantiate new class with inheritance
4. **Metadata Setup:** Configure name, comment, and symbol
5. **Registry Integration:** Add to class registries
6. **Constant Registration:** Make class immutable

## Class Name Validation

### Naming Rules
```cpp
bool goodplugarg = isalpha(plugarg[0]);
for (const char* pa = &plugarg[0]; goodplugarg && *pa; pa++)
  goodplugarg = isalnum(*pa) || *pa == '_';

if (!goodplugarg)
  RPS_FATALOUT("bad class name");
```

**Requirements:**
- Must start with alphabetic character
- Can contain alphanumeric characters and underscores
- Must not conflict with existing object names

## Superclass Resolution

### Inheritance Setup
```cpp
bool goodsupername = isalpha(supername[0]) || supername[0] == '_';
for (const char* pn = supername; goodsupername && *pn; pn++)
  goodsupername = isalnum(*pn) || *pn == '_';

if (!goodsupername)
  RPS_FATALOUT("bad superclass name");

_f.obsuperclass = Rps_ObjectRef::find_object_or_null_by_string(&_, std::string(supername));
if (!_f.obsuperclass)
  RPS_FATALOUT("unknown superclass name");

if (!_f.obsuperclass->is_class())
  RPS_FATALOUT("super name not naming a class");
```

**Superclass Requirements:**
- Must exist in the system
- Must be a valid class object
- Must be accessible by name

## Core Class Creation

### Class Instantiation
```cpp
_f.obnewclass = Rps_ObjectRef::make_named_class(&_,
    _f.obsuperclass, std::string{plugarg});
```

**Class Properties:**
- Inherits from specified superclass
- Gets unique object ID
- Placed in appropriate object space
- Becomes part of class hierarchy

## Metadata Configuration

### Name Attribute
```cpp
_f.namestr = Rps_Value{std::string(plugarg)};
_f.obnewclass->put_attr(RPS_ROOT_OB(_1EBVGSfW2m200z18rx), _f.namestr);
```

**Self-Naming:** Class references its own name

### Comment Attribute
```cpp
if (comment) {
  _f.commentstr = Rps_StringValue(comment);
  _f.obnewclass->put_attr(RPS_ROOT_OB(_0jdbikGJFq100dgX1n), _f.commentstr);
}
```

**Documentation:** Classes can be documented for future reference

## Symbol Association

### Symbol Creation
```cpp
_f.obsymbol = Rps_ObjectRef::make_new_strong_symbol(&_, std::string{plugarg});
paylsymb->symbol_put_value(_f.obnewclass);
_f.obnewclass->put_attr(RPS_ROOT_OB(_3Q3hJsSgCDN03GTYW5), _f.obsymbol);
```

**Bidirectional Linking:**
- Symbol resolves to class object
- Class references its symbol

## Registry Integration

### Mutable Class Set
```cpp
_f.obmutsetclass = RPS_ROOT_OB(_4DsQEs8zZf901wT1LH); // "the_mutable_set_of_classes"
Rps_PayloadSetOb* paylsetob = _f.obmutsetclass->get_dynamic_payload<Rps_PayloadSetOb>();
paylsetob->add(_f.obnewclass);
```

**Class Registry:** Adds new class to the global class set

### C++ Code Class Components
```cpp
_f.obcppcodeclass = RPS_ROOT_OB(_14J2l2ZPGtp00WLhIu); // cplusplus_code∈class
_f.obcppcodeclass->append_comp1(_f.obnewclass);
rps_add_constant_object(&_, _f.obnewclass);
```

**Code Generation Integration:**
- Adds class as component of C++ code class
- Makes class a constant for immutability

## Instance Class Support

### Instance Marking
```cpp
if (isinstanced) {
  // Handle instance class creation
  RPS_WARNOUT("added new instance class " << _f.obnewclass);
} else {
  RPS_INFORMOUT("added new class " << _f.obnewclass);
}
```

**Instance Classes:** Special handling for instance-based classes

## Integration with Code Generation

### Backend Support
Created classes integrate with:

- **GNU Lightning:** `lightgen_rps.cc` - Dynamic code generation
- **libgccjit:** `gccjit_rps.cc` - GCC JIT compilation
- **Static C++:** `cppgen_rps.cc` - Static code generation
- **Class System:** Inheritance and method dispatch

### Code Generation Usage
```cpp
// Code generators can access:
auto className = classObj->get_attr(name_attr);
auto superClass = classObj->get_superclass();
auto classComment = classObj->get_attr(comment_attr);
// Use for generating C++ class declarations
```

## Use Cases

### Code Generation Classes
- **Data Classes:** Classes representing data structures
- **Function Classes:** Classes containing methods
- **Type Classes:** Classes representing complex types
- **Interface Classes:** Abstract base classes

### Inheritance Hierarchies
- **Base Classes:** Fundamental code generation classes
- **Derived Classes:** Specialized implementations
- **Mixin Classes:** Composable class functionality
- **Template Classes:** Parameterized class definitions

## Build and Deployment

### Compilation
```bash
cd $REFPERSYS_TOPDIR
make plugins_dir/rpsplug_create_cplusplus_code_class.so
/bin/ln -svf $(/bin/pwd)/plugins_dir/rpsplug_create_cplusplus_code_class.so /tmp/
```

### Runtime Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_create_cplusplus_code_class.so \
            --plugin-arg=rpsplug_create_cplusplus_code_class:my_data_class \
            --extra=super=object \
            --extra=comment='Data class for my application' \
            --batch --dump=.
```

## Example Usage

### Creating Data Classes
```bash
# Person data class
./refpersys --plugin-after-load=/tmp/rpsplug_create_cplusplus_code_class.so \
            --plugin-arg=rpsplug_create_cplusplus_code_class:person_class \
            --extra=super=object \
            --extra=comment='Class representing a person entity' \
            --batch --dump=.

# Shape class hierarchy
./refpersys --plugin-after-load=/tmp/rpsplug_create_cplusplus_code_class.so \
            --plugin-arg=rpsplug_create_cplusplus_code_class:shape_class \
            --extra=super=object \
            --extra=comment='Base class for geometric shapes' \
            --batch --dump=.

# Circle class inheriting from shape
./refpersys --plugin-after-load=/tmp/rpsplug_create_cplusplus_code_class.so \
            --plugin-arg=rpsplug_create_cplusplus_code_class:circle_class \
            --extra=super=shape_class \
            --extra=comment='Circle shape implementation' \
            --batch --dump=.
```

## Performance Characteristics

### Execution Time
- **Validation:** Fast string parsing and uniqueness checking
- **Superclass Lookup:** Symbol table operations
- **Class Creation:** Standard RefPerSys class instantiation
- **Registry Updates:** Set and component operations

### Memory Usage
- **Class Object:** Standard class overhead
- **Symbol Object:** Additional symbol structure
- **Registry Entries:** Minimal additional storage
- **Metadata:** Name and comment strings

## Relationship to Other Plugins

### Complementary Plugins
- **`rpsplug_create_cplusplus_primitive_type.cc`**: Creates types used by classes
- **Code Generation Plugins:** Use classes for code emission
- **Method Plugins:** Add methods to generated classes

### Code Generation Workflow
```
Create Classes → Add Methods → Generate Code
     ↓             ↓            ↓
Class Registry   Method Set    C++ Output
```

## Thread Safety

### Synchronization
Plugin uses appropriate locking for class creation, registry updates, and attribute manipulation, ensuring thread-safe operation in the RefPerSys environment.

## Error Handling

### Validation Failures
- **Missing Name:** Requires class name
- **Invalid Name:** Must follow identifier rules
- **Missing Superclass:** Requires superclass specification
- **Invalid Superclass:** Must be existing class
- **Name Conflicts:** Cannot reuse existing names

### Fatal Errors
```cpp
RPS_FATALOUT("without argument; should be some non-empty name");
RPS_FATALOUT("without super extra name");
RPS_FATALOUT("unknown superclass name");
RPS_FATALOUT("super name not naming a class");
```

## File Status

**Status:** Active/Production
**Date:** 2024-2025
**Purpose:** C++ code class creation and registration

## Summary

The `rpsplug_create_cplusplus_code_class.cc` plugin provides essential functionality for creating C++ code classes that form the foundation of RefPerSys's code generation capabilities. By creating class objects with inheritance relationships and integrating them into the system's class registries, it enables structured code generation across multiple backends. The plugin creates the class hierarchy framework needed for sophisticated C++ code generation, supporting inheritance, method dispatch, and type relationships. This plugin represents a crucial component in RefPerSys's code generation infrastructure, providing the class system foundation that enables the creation of complex, inheritance-based C++ code through various compilation strategies. The integration with multiple code generation backends and the class registry system makes it a key enabler for RefPerSys's ability to generate sophisticated, object-oriented C++ code.