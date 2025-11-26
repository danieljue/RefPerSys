# Root to Constant Conversion Plugin (`plugins_dir/rpsplug_root2const.cc`)

**File Path:** `plugins_dir/rpsplug_root2const.cc`

## Overview

This RefPerSys plugin converts a root object to a constant object. Root objects are protected from garbage collection but are mutable, while constant objects are also protected from GC but are immutable. The plugin moves an object from the system's root set to its constant set, changing its lifecycle status from mutable root to immutable constant.

## Plugin Purpose

### Object Lifecycle Management
The plugin enables object lifecycle transitions by:

- **Root to Constant:** Converting mutable roots to immutable constants
- **Immutability Enforcement:** Making objects permanently unchangeable
- **System Stability:** Protecting critical objects from modification
- **Lifecycle Control:** Managing object mutability over time
- **System Administration:** Controlling object modification permissions

## Command-Line Interface

### Basic Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_root2const.so \
            --plugin-arg=rpsplug_root2const:oidorname \
            --extra=comment='some comment' \
            --batch --dump=.
```

### Required Arguments
- **`--plugin-arg`**: Object name or OID of root object to convert

### Optional Arguments
- **`--extra=comment`**: Documentation comment for the constant
- **`--extra=dump`**: Directory to dump system state after conversion

## Plugin Architecture

### Entry Point
```cpp
void rps_do_plugin(const Rps_Plugin* plugin)
```

**Execution Flow:**
1. **Argument Validation:** Check object identifier format
2. **Object Lookup:** Find object by name or OID
3. **Sacred Object Check:** Prevent conversion of critical system objects
4. **Constant Registration:** Add to system's constant set
5. **Root Removal:** Remove from root set
6. **Optional Dump:** Save system state if requested

## Object Identification

### Multiple Lookup Methods
```cpp
if (isalpha(plugarg[0])) {
  // Named object lookup
  _f.oboldroot = Rps_PayloadSymbol::find_named_object(std::string(plugarg));
} else if (plugarg[0] == '_' && isalnum(plugarg[1])) {
  // OID lookup
  _f.oboldroot = Rps_ObjectRef::find_object_or_null_by_oid(&_, Rps_Id(plugarg));
} else {
  RPS_FATALOUT("bad argument - expect name or objid");
}
```

**Supported Formats:**
- **Named Objects:** Alphabetic names (e.g., "my_object")
- **Object IDs:** OID format starting with underscore (e.g., "_1aGtWm38Vw701jDhZn")

## Sacred Object Protection

### Critical System Objects
The plugin prevents conversion of essential system objects:

```cpp
if (false
    || _f.oboldroot == RPS_ROOT_OB(_2i66FFjmS7n03HNNBx) // space∈class
    || _f.oboldroot == RPS_ROOT_OB(_10YXWeY7lYc01RpQTA) // the_system_class∈class
    || _f.oboldroot == RPS_ROOT_OB(_1Io89yIORqn02SXx4p) // RefPerSys_system∈the_system_class
    // ... many more critical objects
   ) {
  RPS_FATALOUT("refering to sacred root object");
}
```

**Protected Objects Include:**
- Core classes (object, class, symbol, etc.)
- System infrastructure (agenda, spaces, etc.)
- Fundamental types (int, string, value, etc.)
- Essential attributes and sets

## Constant Registration

### System Constant Set
```cpp
_f.obsystem = RPS_ROOT_OB(_1Io89yIORqn02SXx4p); // RefPerSys_system
_f.oldsetv = _f.obsystem->get_physical_attr(RPS_ROOT_OB(_2aNcYqKwdDR01zp0Xp)); // "constant"∈named_attribute
_f.newsetv = Rps_SetValue{_f.oldsetv, Rps_Value(_f.oboldroot)};
_f.obsystem->put_attr(RPS_ROOT_OB(_2aNcYqKwdDR01zp0Xp), _f.newsetv);
```

**Constant Benefits:**
- Protected from garbage collection
- Immutable (cannot be modified)
- Permanent system reference
- Type-safe constant access

## Root Removal

### Lifecycle Transition
```cpp
if (!rps_remove_root_object(_f.oboldroot))
  RPS_WARNOUT("failed to remove non-root object " << _f.oboldroot);
```

**Transition Effects:**
- Object loses root protection
- Maintains constant protection
- Becomes immutable
- Changes from mutable root to immutable constant

## Metadata Enhancement

### Comment Addition
```cpp
if (comment && comment[0]) {
  _f.commentstrv = Rps_StringValue(comment);
  _f.oboldroot->put_attr(RPS_ROOT_OB(_0jdbikGJFq100dgX1n), _f.commentstrv);
  _f.oboldroot->touch_now();
}
```

**Documentation:** Constants can be documented for future reference

## Optional System Dump

### State Persistence
```cpp
if (dumpdir) {
  rps_dump_into(std::string(dumpdir), &_);
}
```

**Dump Benefits:**
- Preserves conversion in persistent storage
- Creates backup of system state
- Enables rollback if needed

## Use Cases

### System Hardening
- **Configuration Freezing:** Make configuration objects immutable
- **API Stabilization:** Convert interface objects to constants
- **Security Hardening:** Prevent modification of security-critical objects
- **System Stabilization:** Lock down essential system components

### Development Workflow
- **Prototype to Production:** Convert development objects to constants
- **Version Stabilization:** Make released objects immutable
- **API Finalization:** Convert API objects to constants
- **Data Integrity:** Protect reference data from modification

## Build and Deployment

### Compilation
```bash
cd $REFPERSYS_TOPDIR
./do_build_refpersys_plugin plugins_dir/rpsplug_root2const.cc \
                           -o /tmp/rpsplug_root2const.so
```

### Runtime Usage
```bash
# Convert named root to constant
./refpersys --plugin-after-load=/tmp/rpsplug_root2const.so \
            --plugin-arg=rpsplug_root2const:my_root_object \
            --extra=comment='Now immutable for system stability' \
            --batch --dump=.

# Convert OID root to constant with dump
./refpersys --plugin-after-load=/tmp/rpsplug_root2const.so \
            --plugin-arg=rpsplug_root2const:_1aGtWm38Vw701jDhZn \
            --extra=dump=/tmp/system_backup \
            --batch
```

## Performance Characteristics

### Execution Time
- **Object Lookup:** Fast name or OID resolution
- **Set Operations:** Efficient set modification
- **Attribute Updates:** Minimal metadata changes
- **Optional Dump:** Proportional to system size

### Memory Usage
- **Set Modification:** Minimal additional storage
- **Attribute Addition:** Small comment storage
- **No Object Duplication:** Works with existing objects

## Relationship to Other Plugins

### Complementary Plugins
- **`rpsplug_installrootoid.cc`**: Creates root objects
- **Creation Plugins:** Create objects to convert
- **Dump/Load Plugins:** Persist conversion results

### Object Lifecycle
```
Create Objects → Make Root → Convert to Constant
     ↓              ↓              ↓
Object Creation   GC Protection   Immutability
```

## Thread Safety

### Synchronization
Plugin uses appropriate locking for system object modifications, ensuring thread-safe constant set updates and root object removal.

## Error Handling

### Validation Failures
- **Missing Argument:** Requires object identifier
- **Invalid Format:** Must be valid name or OID
- **Object Not Found:** Must reference existing object
- **Sacred Object:** Cannot convert critical system objects

### Fatal Errors
```cpp
RPS_FATALOUT("without argument; should be some name or root objectid");
RPS_FATALOUT("refering to sacred root object");
```

## Integration with System Architecture

### Constant vs Root Objects
- **Root Objects:** Protected from GC, mutable, temporary status
- **Constant Objects:** Protected from GC, immutable, permanent status
- **Lifecycle Flow:** Objects can be root → constant, not vice versa

## File Status

**Status:** Active/Production
**Date:** 2023-2024
**Purpose:** Object lifecycle management and system hardening

## Summary

The `rpsplug_root2const.cc` plugin provides essential functionality for managing object lifecycle transitions in RefPerSys. By converting root objects to constants, it enables system administrators and developers to harden their applications by making critical objects immutable. The plugin includes sophisticated protection against converting essential system objects, ensuring system stability while providing flexibility for application-specific hardening. Its integration with the system's constant management and optional dumping capabilities make it a powerful tool for system administration and deployment preparation. The plugin demonstrates the importance of lifecycle management in complex systems where objects need different levels of protection and mutability at different stages of system evolution.