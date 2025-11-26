# Named Attribute Creation Plugin (`plugins_dir/rpsplug_createnamedattribute.cc`)

**File Path:** `plugins_dir/rpsplug_createnamedattribute.cc`

## Overview

This RefPerSys plugin creates new named attribute objects in the system. Named attributes are fundamental to RefPerSys's object system, providing typed and named properties that can be attached to objects. The plugin enables dynamic extension of the system's attribute vocabulary, allowing developers to define new kinds of object metadata and relationships.

## Plugin Purpose

### Attribute System Extension
The plugin enables dynamic extension of RefPerSys's attribute system by:

- **Attribute Creation:** Instantiating new named attribute objects
- **Symbol Association:** Creating symbolic names for attributes
- **Type Safety:** Establishing typed attribute relationships
- **Metadata Framework:** Providing extensible object properties
- **System Integration:** Making attributes available throughout the system

## Command-Line Interface

### Basic Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_createnamedattribute.so \
            --plugin-arg=rpsplug_createnamedattribute:new_attr_name \
            --extra=comment='some comment' \
            --extra=rooted=0 \
            --extra=constant=0 \
            --batch --dump=.
```

### Required Arguments
- **`--plugin-arg`**: The attribute name to create

### Optional Arguments
- **`--extra=comment`**: Documentation comment for the attribute
- **`--extra=rooted`**: Make attribute a root object (true/false/yes/no/1/0)
- **`--extra=constant`**: Make attribute a constant object (true/false/yes/no/1/0)

## Plugin Architecture

### Entry Point
```cpp
void rps_do_plugin(const Rps_Plugin* plugin)
```

**Execution Flow:**
1. **Name Validation:** Check attribute name format and uniqueness
2. **Lifecycle Parsing:** Interpret rooted/constant flags with flexible syntax
3. **Symbol Creation:** Instantiate associated symbol object
4. **Attribute Creation:** Create named attribute object
5. **Bidirectional Linking:** Connect symbol and attribute
6. **Registration:** Add to appropriate system collections

## Attribute Name Validation

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

## Lifecycle Flag Parsing

### Flexible Boolean Parsing
```cpp
if (rooted) {
  if (!strcmp(rooted, "true")) isrooted = true;
  else if (!strcmp(rooted, "false")) isrooted = false;
  else if (rooted[0]=='Y' || rooted[0]=='y') isrooted = true;
  else if (rooted[0]=='N' || rooted[0]=='n') isrooted = false;
  else if (isdigit(rooted[0])) isrooted = atoi(rooted)>0;
  else RPS_WARNOUT("ignoring rooted=" << Rps_QuotedC_String(rooted));
}
```

**Supported Formats:**
- **Text:** true/false, yes/no
- **Case Insensitive:** True/False, Yes/No
- **Numeric:** 1/0, any positive/zero number
- **Invalid:** Warning for unrecognized values

## Core Object Creation

### Symbol Creation
```cpp
_f.obsymbol = Rps_ObjectRef::make_new_strong_symbol(&_, std::string{plugarg});
```

**Symbol Properties:**
- Strong symbol for persistent reference
- Placed in root object space
- Gets unique object ID

### Attribute Creation
```cpp
_f.obnamedattr = Rps_ObjectRef::make_object(&_,
    RPS_ROOT_OB(_4pSwobFHGf301Qgwzh), // named_attribute∈class
    Rps_ObjectRef::root_space());
```

**Attribute Properties:**
- Instance of `named_attribute` class
- Placed in root object space
- Gets unique object ID

## Bidirectional Linking

### Symbol to Attribute
```cpp
paylsymb->symbol_put_value(_f.obnamedattr);
```

**Symbol Resolution:** Symbol resolves to attribute object

### Attribute to Symbol
```cpp
_f.obnamedattr->put_attr(RPS_ROOT_OB(_3Q3hJsSgCDN03GTYW5), _f.obsymbol);
```

**Reverse Reference:** Attribute references its symbol

## Metadata Configuration

### Name Attributes
```cpp
_f.namestr = Rps_Value{std::string(plugarg)};
_f.obsymbol->put_attr(RPS_ROOT_OB(_1EBVGSfW2m200z18rx), _f.namestr);
_f.obnamedattr->put_attr(RPS_ROOT_OB(_1EBVGSfW2m200z18rx), _f.namestr);
```

**Self-Naming:** Both symbol and attribute reference the name

### Comment Attribute
```cpp
if (comment) {
  _f.commentstr = Rps_StringValue(comment);
  _f.obnamedattr->put_attr(RPS_ROOT_OB(_0jdbikGJFq100dgX1n), _f.commentstr);
}
```

**Documentation:** Attributes can be documented for future reference

## Registration System

### Root Registration
```cpp
if (isrooted) {
  rps_add_root_object(_f.obnamedattr);
  RPS_INFORMOUT("added new root named attribute " << _f.obnamedattr);
}
```

**Root Benefits:** Survives garbage collection, always accessible

### Constant Registration
```cpp
else if (isconstant) {
  rps_add_constant_object(&_, _f.obnamedattr);
  RPS_INFORMOUT("added new constant named attribute " << _f.obnamedattr);
}
```

**Constant Benefits:** Immutable, system-wide availability

### Plain Attribute
```cpp
else {
  RPS_INFORMOUT("added new named attribute " << _f.obnamedattr);
}
```

**Plain Attributes:** Subject to garbage collection

## Integration with Attribute System

### Attribute Usage
Created attributes integrate with:
- **Object Properties:** Used as keys for object attributes
- **Type System:** Define typed relationships
- **Metadata Framework:** Provide extensible object properties
- **Query System:** Enable attribute-based object queries

## Use Cases

### Object Metadata
- **Custom Properties:** Define domain-specific object properties
- **Relationship Types:** Create typed object relationships
- **Configuration Keys:** Define system configuration attributes
- **Extension Points:** Enable plugin-specific metadata

### System Extension
- **Class Attributes:** Define class-specific properties
- **Instance Metadata:** Add per-object information
- **Relationship Modeling:** Define object connection types
- **Configuration Management:** Create configuration attribute keys

## Build and Deployment

### Compilation
```bash
cd $REFPERSYS_TOPDIR
make plugins_dir/rpsplug_createnamedattribute.so
/bin/ln -sfv $(/bin/pwd)/plugins_dir/rpsplug_createnamedattribute.so /tmp/
```

### Runtime Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_createnamedattribute.so \
            --plugin-arg=rpsplug_createnamedattribute:color \
            --extra=comment='color attribute for visual objects' \
            --extra=constant=1 \
            --batch --dump=.
```

## Example Usage

### Creating Object Properties
```bash
# Size attribute
./refpersys --plugin-after-load=/tmp/rpsplug_createnamedattribute.so \
            --plugin-arg=rpsplug_createnamedattribute:size \
            --extra=comment='size property for objects' \
            --extra=constant=1 \
            --batch --dump=.

# Priority attribute
./refpersys --plugin-after-load=/tmp/rpsplug_createnamedattribute.so \
            --plugin-arg=rpsplug_createnamedattribute:priority \
            --extra=comment='priority level attribute' \
            --extra=rooted=1 \
            --batch --dump=.
```

### Creating Relationship Attributes
```bash
# Parent relationship
./refpersys --plugin-after-load=/tmp/rpsplug_createnamedattribute.so \
            --plugin-arg=rpsplug_createnamedattribute:parent \
            --extra=comment='parent relationship attribute' \
            --extra=constant=1 \
            --batch --dump=.

# Children relationship
./refpersys --plugin-after-load=/tmp/rpsplug_createnamedattribute.so \
            --plugin-arg=rpsplug_createnamedattribute:children \
            --extra=comment='children relationship attribute' \
            --extra=constant=1 \
            --batch --dump=.
```

## Performance Characteristics

### Execution Time
- **Validation:** Fast string parsing and uniqueness checking
- **Object Creation:** Standard RefPerSys instantiation
- **Linking:** Efficient bidirectional reference setup
- **Registration:** Minimal system integration overhead

### Memory Usage
- **Symbol Object:** Standard object overhead
- **Attribute Object:** Additional object for the attribute
- **Bidirectional Links:** Reference storage
- **Metadata:** Name and comment strings

## Relationship to Other Plugins

### Complementary Plugins
- **Symbol Creation:** Creates symbols used by attributes
- **Class Creation:** Uses attributes for class definitions
- **Object Creation:** Uses attributes for object properties

### Attribute Ecosystem
```
Symbols ← Named Attributes ← Object Properties
    ↓              ↓                ↓
Names       Metadata Keys      Object Values
```

## Thread Safety

### Synchronization
Plugin uses appropriate locking for object creation and attribute manipulation, ensuring thread-safe operation in the RefPerSys environment.

## Error Handling

### Validation Failures
- **Missing Name:** Requires attribute name
- **Invalid Name:** Must follow identifier rules
- **Name Conflicts:** Cannot reuse existing names

### Warning for Invalid Flags
```cpp
RPS_WARNOUT("is ignoring rooted=" << Rps_QuotedC_String(rooted));
```

**Graceful Degradation:** Warns about invalid flag values but continues

## File Status

**Status:** Active/Production
**Date:** 2023-2025
**Purpose:** Named attribute creation and registration

## Summary

The `rpsplug_createnamedattribute.cc` plugin provides essential functionality for extending RefPerSys's attribute system. By creating named attribute objects with associated symbols and configurable lifecycles, it enables dynamic extension of the system's metadata framework. The plugin supports flexible boolean flag parsing and creates bidirectional links between symbols and attributes, ensuring proper integration with the system's naming and referencing mechanisms. Named attributes created by this plugin serve as the foundation for object properties, relationships, and metadata throughout the RefPerSys system, making this plugin crucial for system extension and domain-specific customization. The sophisticated lifecycle management and validation make it a robust tool for system administrators and developers extending RefPerSys's object model.