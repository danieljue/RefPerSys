# Named Selector Creation Plugin (`plugins_dir/rpsplug_createnamedselector.cc`)

**File Path:** `plugins_dir/rpsplug_createnamedselector.cc`

## Overview

This RefPerSys plugin creates new named selector objects in the system. Named selectors are specialized constructs used for method dispatch and message sending in RefPerSys's object-oriented system. Selectors provide a way to identify and invoke methods on objects, similar to method names in traditional object-oriented languages. The plugin enables dynamic extension of the system's method dispatch vocabulary.

## Plugin Purpose

### Method Dispatch Extension
The plugin enables dynamic extension of RefPerSys's method system by:

- **Selector Creation:** Instantiating new named selector objects
- **Symbol Association:** Creating symbolic names for method dispatch
- **Message Sending:** Enabling method invocation by name
- **Dynamic Dispatch:** Supporting runtime method resolution
- **System Integration:** Making selectors available for object messaging

## Implementation Notes

### Code Reuse
```cpp
// lots of code from similar rpsplug_createnamedattribute.cc
```

**Inheritance:** Plugin shares significant code structure with the named attribute plugin, adapted for selector-specific functionality.

## Command-Line Interface

### Basic Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_createnamedselector.so \
            --plugin-arg=rpsplug_createnamedselector:new_selector_name \
            --extra=comment='some comment' \
            --extra=rooted=0 \
            --extra=constant=0 \
            --batch --dump=.
```

### Required Arguments
- **`--plugin-arg`**: The selector name to create

### Optional Arguments
- **`--extra=comment`**: Documentation comment for the selector
- **`--extra=rooted`**: Make selector a root object (true/false/yes/no/1/0)
- **`--extra=constant`**: Make selector a constant object (true/false/yes/no/1/0)

## Plugin Architecture

### Entry Point
```cpp
void rps_do_plugin(const Rps_Plugin* plugin)
```

**Execution Flow:**
1. **Name Validation:** Check selector name format and uniqueness
2. **Lifecycle Parsing:** Interpret rooted/constant flags with flexible syntax
3. **Symbol Creation:** Instantiate associated symbol object
4. **Selector Creation:** Create named selector object
5. **Bidirectional Linking:** Connect symbol and selector
6. **Registration:** Add to appropriate system collections

## Selector Name Validation

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

### Selector Creation
```cpp
_f.obnamedselector = Rps_ObjectRef::make_object(&_,
    RPS_ROOT_OB(_0cSUtWqTYdZ00mjeNR), // named_selector∈class
    Rps_ObjectRef::root_space());
```

**Selector Properties:**
- Instance of `named_selector` class
- Placed in root object space
- Gets unique object ID

## Bidirectional Linking

### Symbol to Selector
```cpp
paylsymb->symbol_put_value(_f.obnamedselector);
```

**Symbol Resolution:** Symbol resolves to selector object

### Selector to Symbol
```cpp
_f.obnamedselector->put_attr(RPS_ROOT_OB(_3Q3hJsSgCDN03GTYW5), _f.obsymbol);
```

**Reverse Reference:** Selector references its symbol

## Metadata Configuration

### Name Attributes
```cpp
_f.namestr = Rps_Value{std::string(plugarg)};
_f.obsymbol->put_attr(RPS_ROOT_OB(_1EBVGSfW2m200z18rx), _f.namestr);
_f.obnamedselector->put_attr(RPS_ROOT_OB(_1EBVGSfW2m200z18rx), _f.namestr);
```

**Self-Naming:** Both symbol and selector reference the name

### Comment Attribute
```cpp
if (comment) {
  _f.commentstr = Rps_StringValue(comment);
  _f.obnamedselector->put_attr(RPS_ROOT_OB(_0jdbikGJFq100dgX1n), _f.commentstr);
}
```

**Documentation:** Selectors can be documented for future reference

## Registration System

### Root Registration
```cpp
if (isrooted) {
  rps_add_root_object(_f.obnamedselector);
  RPS_INFORMOUT("added new root named selector " << _f.obnamedselector);
}
```

**Root Benefits:** Survives garbage collection, always accessible

### Constant Registration
```cpp
else if (isconstant) {
  rps_add_constant_object(&_, _f.obnamedselector);
  RPS_INFORMOUT("added new constant named selector " << _f.obnamedselector);
}
```

**Constant Benefits:** Immutable, system-wide availability

### Plain Selector
```cpp
else {
  RPS_INFORMOUT("added new named selector " << RPS_OBJECT_DISPLAY(_f.obnamedselector));
}
```

**Plain Selectors:** Subject to garbage collection

## Integration with Method Dispatch

### Message Sending
Created selectors integrate with:
- **Method Dispatch:** Used for dynamic method invocation
- **Message Passing:** Enable object messaging by selector name
- **Polymorphism:** Support method overriding and inheritance
- **Dynamic Binding:** Runtime method resolution

## Use Cases

### Object Messaging
- **Method Calls:** Define method names for invocation
- **Message Patterns:** Create messaging protocols
- **Interface Definition:** Define method contracts
- **Dynamic Dispatch:** Enable polymorphic method calls

### System Extension
- **Custom Methods:** Define domain-specific operations
- **Protocol Definition:** Create messaging interfaces
- **Behavior Extension:** Add new object behaviors
- **API Design:** Define callable interfaces

## Build and Deployment

### Compilation
```bash
cd $REFPERSYS_TOPDIR
make plugins_dir/rpsplug_createnamedselector.so
/bin/ln -svf $(/bin/pwd)/plugins_dir/rpsplug_createnamedselector.so /tmp/
```

### Runtime Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_createnamedselector.so \
            --plugin-arg=rpsplug_createnamedselector:draw \
            --extra=comment='selector for drawing operations' \
            --extra=constant=1 \
            --batch --dump=.
```

## Example Usage

### Creating Method Selectors
```bash
# Draw method selector
./refpersys --plugin-after-load=/tmp/rpsplug_createnamedselector.so \
            --plugin-arg=rpsplug_createnamedselector:draw \
            --extra=comment='selector for drawing operations' \
            --extra=constant=1 \
            --batch --dump=.

# Calculate method selector
./refpersys --plugin-after-load=/tmp/rpsplug_createnamedselector.so \
            --plugin-arg=rpsplug_createnamedselector:calculate \
            --extra=comment='selector for calculation operations' \
            --extra=rooted=1 \
            --batch --dump=.
```

### Creating Protocol Selectors
```bash
# Initialize selector
./refpersys --plugin-after-load=/tmp/rpsplug_createnamedselector.so \
            --plugin-arg=rpsplug_createnamedselector:initialize \
            --extra=comment='selector for object initialization' \
            --extra=constant=1 \
            --batch --dump=.

# Cleanup selector
./refpersys --plugin-after-load=/tmp/rpsplug_createnamedselector.so \
            --plugin-arg=rpsplug_createnamedselector:cleanup \
            --extra=comment='selector for object cleanup' \
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
- **Selector Object:** Additional object for the selector
- **Bidirectional Links:** Reference storage
- **Metadata:** Name and comment strings

## Relationship to Other Plugins

### Complementary Plugins
- **Symbol Creation:** Creates symbols used by selectors
- **Class Creation:** Uses selectors for method definitions
- **Message System:** Uses selectors for method dispatch

### Method System Ecosystem
```
Symbols ← Named Selectors ← Method Dispatch
    ↓              ↓              ↓
Names       Message Names     Method Calls
```

## Thread Safety

### Synchronization
Plugin uses appropriate locking for object creation and attribute manipulation, ensuring thread-safe operation in the RefPerSys environment.

## Error Handling

### Validation Failures
- **Missing Name:** Requires selector name
- **Invalid Name:** Must follow identifier rules
- **Name Conflicts:** Cannot reuse existing names

### Warning for Invalid Flags
```cpp
RPS_WARNOUT("is ignoring rooted=" << Rps_QuotedC_String(rooted));
```

**Graceful Degradation:** Warns about invalid flag values but continues

## File Status

**Status:** Active/Production
**Date:** 2024-2025
**Purpose:** Named selector creation and registration

## Summary

The `rpsplug_createnamedselector.cc` plugin provides essential functionality for extending RefPerSys's method dispatch system. By creating named selector objects with associated symbols and configurable lifecycles, it enables dynamic extension of the system's messaging and method invocation framework. The plugin shares significant code structure with the named attribute plugin but specializes in selector creation for method dispatch. Named selectors created by this plugin serve as the foundation for message passing, method calls, and polymorphic dispatch throughout the RefPerSys system, making this plugin crucial for implementing object-oriented behaviors and extensible interfaces. The sophisticated lifecycle management and validation make it a robust tool for system administrators and developers extending RefPerSys's method system.