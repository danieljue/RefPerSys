# Root Object Installation Plugin (`plugins_dir/rpsplug_installrootoid.cc`)

**File Path:** `plugins_dir/rpsplug_installrootoid.cc`

## Overview

This RefPerSys plugin installs an existing object as a root object in the system. Root objects are protected from garbage collection and serve as permanent, always-accessible references in the RefPerSys object graph. The plugin takes an object ID (OID) as input, validates it, and promotes the corresponding object to root status.

## Plugin Purpose

### Root Object Management
The plugin enables dynamic root object management by:

- **Object Promotion:** Elevating existing objects to root status
- **GC Protection:** Preventing garbage collection of important objects
- **System Persistence:** Ensuring critical objects remain accessible
- **Reference Stability:** Creating permanent object references
- **System Administration:** Managing the root object set

## Command-Line Interface

### Basic Usage
```bash
./refpersys --plugin-after-load=/tmp/rpsplug_installrootoid.so \
            --plugin-arg=rpsplug_installrootoid:someoid \
            --batch --dump=.
```

### Required Arguments
- **`--plugin-arg`**: Object ID (OID) of existing object to make root

## Plugin Architecture

### Entry Point
```cpp
void rps_do_plugin(const Rps_Plugin* plugin)
```

**Execution Flow:**
1. **Argument Validation:** Check for OID parameter
2. **OID Parsing:** Validate object ID format
3. **Object Lookup:** Find object by OID
4. **Root Registration:** Add object to root set

## Object ID Validation

### OID Format Checking
```cpp
Rps_Id oid(plugarg, &idend, &goodid);
if (!oid || !idend || *idend != (char)0 || !goodid)
  RPS_FATALOUT("with invalid argument - not a valid objid");
```

**OID Requirements:**
- Valid RefPerSys object identifier format
- Complete OID string (no extra characters)
- Successfully parsed by Rps_Id constructor

## Object Lookup and Validation

### Existence Verification
```cpp
_f.obnewroot = Rps_ObjectRef::find_object_or_fail_by_oid(&_, oid);
RPS_ASSERT(_f.obnewroot);
```

**Lookup Requirements:**
- Object must exist in the system
- OID must correspond to a valid object
- Fatal error if object not found

## Root Registration

### Root Set Addition
```cpp
rps_add_root_object(_f.obnewroot);
```

**Root Benefits:**
- Protected from garbage collection
- Always accessible via OID
- Permanent system reference
- Survives system restarts (when persisted)

## Use Cases

### System Administration
- **Critical Objects:** Make essential system objects permanent
- **Configuration Objects:** Protect configuration data
- **Shared Resources:** Ensure shared objects remain accessible
- **API Endpoints:** Keep interface objects available

### Development and Testing
- **Test Fixtures:** Create permanent test objects
- **Debug Objects:** Make debugging aids persistent
- **Prototype Objects:** Protect experimental objects
- **Reference Objects:** Maintain object examples

## Build and Deployment

### Compilation
```bash
cd $REFPERSYS_TOPDIR
./do-build-refpersys-plugin -i plugins_dir/rpsplug_installrootoid.cc \
                           -o /tmp/rpsplug_installrootoid.so
```

### Runtime Usage
```bash
# Make an existing object a root
./refpersys --plugin-after-load=/tmp/rpsplug_installrootoid.so \
            --plugin-arg=rpsplug_installrootoid:_1aGtWm38Vw701jDhZn \
            --batch --dump=.
```

## Example Usage

### Promoting System Objects
```bash
# Make the agenda a root object
./refpersys --plugin-after-load=/tmp/rpsplug_installrootoid.so \
            --plugin-arg=rpsplug_installrootoid:_1aGtWm38Vw701jDhZn \
            --batch --dump=.

# Make a class object permanent
./refpersys --plugin-after-load=/tmp/rpsplug_installrootoid.so \
            --plugin-arg=rpsplug_installrootoid:_2wdmxJecnFZ02VGGFK \
            --batch --dump=.
```

## Performance Characteristics

### Execution Time
- **OID Parsing:** Fast string validation
- **Object Lookup:** Direct OID-based lookup
- **Root Registration:** Efficient set addition

### Memory Usage
- **No Additional Memory:** Only updates existing data structures
- **Root Set Storage:** Minimal additional references
- **No Object Duplication:** Works with existing objects

## Relationship to Other Plugins

### Complementary Plugins
- **Object Creation Plugins:** Create objects to make root
- **Root Management:** Work with root object set
- **Persistence Plugins:** Save root object status

### Object Lifecycle
```
Create Objects → Install as Root → Use Persistently
     ↓              ↓                ↓
Object Creation   Root Protection   Permanent Access
```

## Thread Safety

### Synchronization
Plugin uses RefPerSys's existing root object management which includes appropriate locking for thread-safe root set modifications.

## Error Handling

### Validation Failures
- **Missing Argument:** Requires OID parameter
- **Invalid OID:** Must be valid object identifier format
- **Object Not Found:** OID must correspond to existing object

### Fatal Errors
```cpp
RPS_FATALOUT("without argument; should be some existing objid");
RPS_FATALOUT("with invalid argument - not a valid objid");
```

## Integration with Persistence

### Root Object Persistence
Root objects are automatically included in system dumps and loads:
- **Dump Inclusion:** Root objects are always serialized
- **Load Restoration:** Root status is preserved across restarts
- **System Consistency:** Root objects maintain their status

## File Status

**Status:** Active/Production
**Date:** 2022-2025
**Purpose:** Root object installation and management

## Summary

The `rpsplug_installrootoid.cc` plugin provides essential functionality for managing RefPerSys's root object set. By taking an existing object ID and promoting that object to root status, it ensures that critical system objects remain protected from garbage collection and permanently accessible. The plugin serves as an important system administration tool, allowing developers and administrators to designate important objects as permanent fixtures in the RefPerSys object graph. Its simple interface and robust validation make it a reliable component for managing object lifecycle and persistence in complex RefPerSys applications. The plugin demonstrates the importance of explicit lifecycle management in a system where objects can be created dynamically but need to be preserved for long-term system operation.