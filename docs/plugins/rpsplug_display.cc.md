# Object Display Plugin (`plugins_dir/rpsplug_display.cc`)

**File Path:** `plugins_dir/rpsplug_display.cc`

## Overview

This RefPerSys plugin displays detailed information about a specified object. The plugin takes an object name or object ID (OID) as input and outputs comprehensive object information including its class, attributes, and metadata. It's a diagnostic and inspection tool for examining the state of objects in the RefPerSys system.

## Plugin Purpose

### Object Inspection and Debugging
The plugin enables system introspection by:

- **Object Lookup:** Finding objects by name or OID
- **Detailed Display:** Showing complete object information
- **Debugging Support:** Inspecting object state during development
- **System Analysis:** Examining object relationships and attributes
- **Diagnostic Tool:** Troubleshooting object-related issues

## Command-Line Interface

### Basic Usage
```bash
./refpersys --plugin-after-load=plugins_dir/rpsplug_display.so \
            --plugin-arg=rpsplug_display:<object-name-or-oid>
```

### Required Arguments
- **`--plugin-arg`**: Object name or OID to display

### Optional Arguments
- **`--extra=from`**: Source file name for context
- **`--extra=lineno`**: Line number for context

## Plugin Architecture

### Entry Point
```cpp
void rps_do_plugin(const Rps_Plugin* plugin)
```

**Execution Flow:**
1. **Argument Validation:** Check for object identifier
2. **Object Lookup:** Find object by name or OID
3. **Context Processing:** Handle optional source location info
4. **Object Display:** Output comprehensive object information

## Object Lookup

### Name Resolution
```cpp
_f.ob = Rps_ObjectRef::find_object_or_null_by_string(&_, std::string(plugarg));
```

**Lookup Methods:**
- **By Name:** Symbol resolution for named objects
- **By OID:** Direct object ID lookup
- **Null Result:** Graceful handling of missing objects

## Display Output

### Standard Display
```cpp
RPS_INFORMOUT("object " << plugarg << " is:" << RPS_OBJECT_DISPLAY(_f.ob));
```

**Display Content:**
- Object class and inheritance
- Object attributes and values
- Physical attributes
- Component relationships
- Metadata and properties

### Contextual Display
```cpp
RPS_INFORMOUT("from " << from << ":" << lineno
              << " object " << plugarg << " is:" << RPS_OBJECT_DISPLAY(_f.ob));
```

**Context Information:**
- Source file and line number
- Build timestamp and git information
- Plugin version details

## Error Handling

### Missing Arguments
```cpp
if (!plugarg || !plugarg[0])
  RPS_FATALOUT("without argument - expecting an object name or oid");
```

**Validation:** Requires object identifier

### Object Not Found
```cpp
if (!_f.ob)
  RPS_WARNOUT(plugarg << " dont name any object");
```

**Warning:** Non-fatal for missing objects

## Build Information

### Version Tracking
```cpp
const char rpsplug_display_shortgit[] = RPS_SHORTGIT;
const char rpsplug_display_buildtimestamp[] = __DATE__ "@" __TIME__;
```

**Metadata:**
- Git commit hash (short)
- Build timestamp
- Compilation information

## Use Cases

### Development and Debugging
- **Object Inspection:** Examine object state and attributes
- **System Analysis:** Understand object relationships
- **Debugging:** Troubleshoot object-related issues
- **Testing:** Verify object creation and configuration

### System Administration
- **Object Verification:** Confirm object existence and properties
- **Configuration Check:** Validate system object setup
- **Data Integrity:** Inspect object consistency
- **Performance Analysis:** Examine object complexity

## Build and Deployment

### Compilation
```bash
cd $REFPERSYS_TOPDIR
./do-build-refpersys-plugin -v \
  -i plugins_dir/rpsplug_display.cc \
  -o plugins_dir/rpsplug_display.so \
  -L /tmp/rpsplug_display.so
```

### Runtime Usage
```bash
# Display a named object
./refpersys --plugin-after-load=/tmp/rpsplug_display.so \
            --plugin-arg=rpsplug_display:RefPerSys_system

# Display with context
./refpersys --plugin-after-load=/tmp/rpsplug_display.so \
            --plugin-arg=rpsplug_display:_1aGtWm38Vw701jDhZn \
            --extra=from=my_script.rps \
            --extra=lineno=42
```

## Example Output

### Object Display
```
object RefPerSys_system is:
object class #2wdmxJecnFZ02VGGFK (repl_delimiter)
  name: "RefPerSys_system"
  class: repl_delimiter∈class
  space: root
  attributes:
    name∈named_attribute: "RefPerSys_system"
    symbol∈symbol: symbol#RefPerSys_system
  components: (none)
```

### Contextual Display
```
from my_script.rps:42 object _1aGtWm38Vw701jDhZn is:
in git a1b2c3d build Dec 26 2025@14:30:25 object _1aGtWm38Vw701jDhZn is:
object agenda #1aGtWm38Vw701jDhZn (the_agenda)
  class: agenda∈class
  space: root
  attributes:
    name∈named_attribute: "the_agenda"
    symbol∈symbol: symbol#the_agenda
  components: (tasklets...)
```

## Performance Characteristics

### Execution Time
- **Object Lookup:** Fast symbol table or OID resolution
- **Display Generation:** Proportional to object complexity
- **Output:** Depends on object size and attributes

### Memory Usage
- **Lookup Operations:** Minimal additional memory
- **Display Buffering:** Temporary string construction
- **No Persistent Changes:** Read-only operation

## Relationship to Other Plugins

### Complementary Plugins
- **Creation Plugins:** Create objects to inspect
- **Modification Plugins:** Change objects to verify
- **Analysis Plugins:** Work with displayed information

### Development Workflow
```
Create Objects → Modify Objects → Display Objects → Analyze Results
     ↓              ↓                ↓              ↓
Object Creation   State Changes    Inspection     Validation
```

## Thread Safety

### Synchronization
Plugin uses RefPerSys's existing object access patterns which include appropriate locking for thread-safe object lookup and attribute access.

## Integration with Development Tools

### Scripting Integration
```bash
# In scripts or automated testing
./refpersys --plugin-after-load=/tmp/rpsplug_display.so \
            --plugin-arg=rpsplug_display:$OBJECT_NAME \
            --extra=from=$0 \
            --extra=lineno=$LINENO
```

**Scripting:** Can be called from shell scripts with context

### IDE Integration
- **Editor Integration:** Called from development environments
- **Debug Sessions:** Inspect objects during debugging
- **Test Suites:** Automated object verification

## File Status

**Status:** Active/Production
**Date:** 2025
**Purpose:** Object inspection and debugging tool

## Summary

The `rpsplug_display.cc` plugin provides essential functionality for inspecting and displaying RefPerSys objects. By taking an object name or OID as input and producing detailed, formatted output about the object's class, attributes, components, and metadata, it serves as a crucial diagnostic tool for developers and system administrators. The plugin includes contextual information like source location and build metadata, making it valuable for debugging and system analysis. Its simple interface and comprehensive output make it an indispensable tool for understanding the state of objects within the RefPerSys system, supporting development, testing, and system administration tasks. The plugin demonstrates the importance of introspection capabilities in a complex, reflective system like RefPerSys.