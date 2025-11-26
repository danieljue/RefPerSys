# Class Creation Plugin (`plugins_dir/rpsplug_createclass.cc`)

**File Path:** `plugins_dir/rpsplug_createclass.cc`

## Overview

This RefPerSys plugin creates new class objects within the system, providing a programmatic way to extend the RefPerSys class hierarchy. It supports creating regular classes as well as special variants like root classes and constant classes, enabling dynamic ontology extension.

## Plugin Purpose

### Dynamic Class Creation
The plugin enables runtime extension of RefPerSys's type system by:

- **Class Instantiation:** Creating new class objects with inheritance
- **Metadata Management:** Attaching names, comments, and symbols
- **Global Registration:** Adding classes to system-wide registries
- **Special Class Types:** Support for rooted and constant classes
- **Symbol Integration:** Creating associated symbol objects

## Command-Line Interface

### Basic Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_createclass.so \
            --plugin-arg=rpsplug_createclass:new_class_name \
            --extra=super=superclass \
            --extra=comment='some comment' \
            --extra=rooted=0 \
            --extra=constant=1 \
            --batch --dump=.
```

### Required Arguments
- **`--plugin-arg`**: The name of the new class to create
- **`--extra=super`**: Name of the superclass (required)

### Optional Arguments
- **`--extra=comment`**: Documentation comment for the class
- **`--extra=rooted`**: Make class a root object (true/1 vs false/0)
- **`--extra=constant`**: Make class a constant object (true/1 vs false/0)

## Plugin Architecture

### Entry Point
```cpp
void rps_do_plugin(const Rps_Plugin* plugin)
```

**Execution Flow:**
1. **Argument Parsing:** Extract plugin arguments and extras
2. **Validation:** Check class name, superclass existence, and options
3. **Class Creation:** Instantiate new class object with inheritance
4. **Metadata Setup:** Configure attributes and relationships
5. **Registration:** Add to global registries and special collections
6. **Special Handling:** Apply rooted/constant status if requested

## Argument Processing

### Plugin Arguments
```cpp
const char* plugarg = rps_get_plugin_cstr_argument(plugin);
```

**Validation Rules:**
- Must be non-empty
- Must start with alphabetic character
- Can contain alphanumeric characters and underscores
- Must not conflict with existing object names

### Extra Arguments
```cpp
const char* supername = rps_get_extra_arg("super");
const char* comment = rps_get_extra_arg("comment");
const char* rooted = rps_get_extra_arg("rooted");
const char* constant = rps_get_extra_arg("constant");
```

**Superclass Requirements:**
- Must exist in the system
- Must be a valid class object
- Name must follow identifier rules

### Special Flags
```cpp
bool isrooted = false;
bool isconstant = false;
// Parsing logic for true/false and numeric values
```

## Class Creation Process

### 1. Validation Phase
```cpp
// Check class name validity
bool goodplugarg = isalpha(plugarg[0]);
for (const char* pa = &plugarg[0]; goodplugarg && *pa; pa++)
  goodplugarg = isalnum(*pa) || *pa == '_';

// Verify uniqueness
if (auto nob = Rps_ObjectRef::find_object_or_null_by_string(&_, std::string(plugarg)))
  RPS_FATALOUT("name already exists: " << nob);
```

### 2. Superclass Resolution
```cpp
// Validate superclass name
bool goodsupername = isalpha(supername[0]) || supername[0] == '_';
for (const char* pn = supername; goodsupername && *pn; pn++)
  goodsupername = isalnum(*pn) || *pn == '_';

// Find and validate superclass
_f.obsuperclass = Rps_ObjectRef::find_object_or_null_by_string(&_, std::string(supername));
if (!_f.obsuperclass->is_class())
  RPS_FATALOUT("not a class: " << _f.obsuperclass);
```

### 3. Class Instantiation
```cpp
// Create the new class
_f.obnewclass = Rps_ObjectRef::make_named_class(&_, _f.obsuperclass, std::string{plugarg});
```

**Class Properties:**
- Inherits from specified superclass
- Gets unique object ID
- Placed in appropriate object space

## Metadata Configuration

### Basic Attributes
```cpp
// Set name attribute
_f.namestr = Rps_Value{std::string(plugarg)};
_f.obnewclass->put_attr(RPS_ROOT_OB(_1EBVGSfW2m200z18rx), _f.namestr);

// Set comment if provided
if (comment) {
  _f.commentstr = Rps_StringValue(comment);
  _f.obnewclass->put_attr(RPS_ROOT_OB(_0jdbikGJFq100dgX1n), _f.commentstr);
}
```

### Symbol Creation and Linking
```cpp
// Create associated symbol
_f.obsymbol = Rps_ObjectRef::make_new_strong_symbol(&_, std::string{plugarg});

// Link symbol to class
Rps_PayloadSymbol* paylsymb = _f.obsymbol->get_dynamic_payload<Rps_PayloadSymbol>();
paylsymb->symbol_put_value(_f.obnewclass);

// Set symbol attribute on class
_f.obnewclass->put_attr(RPS_ROOT_OB(_3Q3hJsSgCDN03GTYW5), _f.obsymbol);
```

## Global Registration

### Class Registry Integration
```cpp
// Add to global class set
_f.obmutsetclass = RPS_ROOT_OB(_4DsQEs8zZf901wT1LH); // the_mutable_set_of_classes
Rps_PayloadSetOb* paylsetob = _f.obmutsetclass->get_dynamic_payload<Rps_PayloadSetOb>();
paylsetob->add(_f.obnewclass);
```

**Purpose:** Maintains system-wide class catalog

## Special Class Types

### Root Classes
```cpp
if (isrooted) {
  rps_add_root_object(_f.obnewclass);
  // Root classes are never garbage collected
}
```

**Root Class Characteristics:**
- Survive garbage collection
- Always available in system
- Used for core system classes

### Constant Classes
```cpp
else if (isconstant) {
  rps_add_constant_object(&_, _f.obnewclass);
  // Constant classes are immutable
}
```

**Constant Class Characteristics:**
- Cannot be modified after creation
- Values are fixed at creation time
- Used for system constants and enumerations

## Error Handling

### Validation Failures
- **Empty class name:** Requires non-empty plugin argument
- **Invalid class name:** Must follow C identifier rules
- **Name conflicts:** Cannot overwrite existing objects
- **Missing superclass:** Requires valid superclass specification
- **Invalid superclass:** Must name an existing class object

### Fatal Errors
```cpp
RPS_FATALOUT("failure: plugin " << plugin->plugin_name
             << " without argument; should be some non-empty name");
```

## Logging and Monitoring

### Success Messages
```cpp
if (isrooted) {
  RPS_INFORMOUT("added new root class " << RPS_OBJECT_DISPLAY(_f.obnewclass)
                << " of hash " << _f.obnewclass->obhash() << " ...");
} else if (isconstant) {
  RPS_INFORMOUT("added new constant class " << RPS_OBJECT_DISPLAY(_f.obnewclass)
                << " of hash " << _f.obnewclass->obhash() << " ...");
} else {
  RPS_INFORMOUT("added new class " << _f.obnewclass
                << " of hash " << _f.obnewclass->obhash() << " ...");
}
```

**Information Provided:**
- Class display representation
- Object hash for identification
- Inheritance relationship
- Associated symbol

## Build and Deployment

### Compilation
```bash
cd $REFPERSYS_TOPDIR
./do-build-refpersys-plugin -v \
  -i plugins_dir/rpsplug_createclass.cc \
  -o plugins_dir/rpsplug_createclass.so \
  -L /tmp/rpsplug_createclass.so
```

### Runtime Usage
```bash
./refpersys \
  --plugin-after-load=/tmp/rpsplug_createclass.so \
  --plugin-arg=rpsplug_createclass:MyNewClass \
  --extra=super=object \
  --extra=comment='My custom class' \
  --extra=rooted=1 \
  --batch --dump=.
```

## Example Usage

### Creating a Regular Class
```bash
./refpersys \
  --plugin-after-load=/tmp/rpsplug_createclass.so \
  --plugin-arg=rpsplug_createclass:Person \
  --extra=super=object \
  --extra=comment='Represents a person entity' \
  --batch --dump=.
```

### Creating a Root Class
```bash
./refpersys \
  --plugin-after-load=/tmp/rpsplug_createclass.so \
  --plugin-arg=rpsplug_createclass:SystemClass \
  --extra=super=class \
  --extra=comment='Core system class' \
  --extra=rooted=1 \
  --batch --dump=.
```

### Creating a Constant Class
```bash
./refpersys \
  --plugin-after-load=/tmp/rpsplug_createclass.so \
  --plugin-arg=rpsplug_createclass:StatusEnum \
  --extra=super=object \
  --extra=comment='Status enumeration values' \
  --extra=constant=1 \
  --batch --dump=.
```

## Thread Safety

### Mutex Usage
```cpp
std::lock_guard<std::recursive_mutex> gunewclass(*(_f.obnewclass->objmtxptr()));
std::lock_guard<std::recursive_mutex> gusymbol(*(_f.obsymbol->objmtxptr()));
std::lock_guard<std::recursive_mutex> gumutsetclass(*(_f.obmutsetclass->objmtxptr()));
```

**Synchronization Points:**
- New class object creation
- Symbol creation and linking
- Global class registry updates

## Performance Characteristics

### Execution Time
- **Fast Validation:** String and name validation is O(n)
- **Object Creation:** Involves hash table lookups and memory allocation
- **Registry Updates:** Requires mutex acquisition and set operations

### Memory Usage
- **New Objects:** Creates 1 class object + 1 symbol object
- **Attribute Storage:** Stores name, comment, and relationship attributes
- **Registry Growth:** Adds entries to global class and root/constant sets

## Integration with RefPerSys Metamodel

### Class Hierarchy Extension
The plugin extends the core class hierarchy:
- **object:** Root of all instance objects
- **class:** Root of all class objects
- **value:** Root of all value objects

### Symbol System Integration
```cpp
// Classes get associated symbols for naming
class_name_symbol → class_object
```

**Symbol Resolution:** Enables lookup by name in REPL and code

## Relationship to Other Plugins

### Complementary Plugins
- **Creation Plugins:** Various specialized creation plugins
- **Type Plugins:** C++ type integration plugins
- **Code Generation:** Plugins that use created classes

### Plugin Ecosystem
This general class creation plugin serves as:
- **Foundation:** Basic class creation functionality
- **Template:** Pattern for specialized creation plugins
- **Integration Point:** Used by higher-level plugins

## File Status

**Status:** Active/Production
**Date:** 2023-2025
**Purpose:** Core infrastructure for dynamic class creation

## Summary

The `rpsplug_createclass.cc` plugin provides essential infrastructure for extending RefPerSys's class hierarchy at runtime. By enabling programmatic creation of classes with inheritance, metadata, and special properties like rooted and constant status, it supports dynamic ontology development and system extension. The plugin demonstrates the power of RefPerSys's metaprogramming capabilities, allowing the system to evolve and adapt its own structure during execution.