# Persistence Serialization Engine Analysis (`dump_rps.cc`)

## Overview

The `dump_rps.cc` file implements the core **persistence serialization engine** for the Reflective Persistent System (RefPerSys), providing the ability to serialize the entire object graph to JSON format for persistent storage. This file serves as the serialization counterpart to the loading engine (`load_rps.cc`), enabling RefPerSys to save its state and restore it across executions.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2019-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// GNU lightning for JIT code generation metadata
extern "C" {
#include "lightning.h";
}

// Git versioning information
extern "C" const char rps_dump_gitid[];
extern "C" const char rps_dump_date[];
extern "C" const char rps_dump_shortgitid[];
extern "C" const char rps_dump_timestamp[];
```

### Key Dependencies
- `refpersys.hh`: Main header containing core system definitions
- **JSON-CPP library**: For JSON serialization/deserialization
- **GNU lightning**: For JIT code generation metadata (`JIT_R_NUM`, `JIT_V_NUM`, `JIT_F_NUM`)
- Standard C++ filesystem and I/O libraries
- System libraries for file operations and process management

## Core Classes

### Rps_Dumper Class

The `Rps_Dumper` class is the central orchestrator for the persistence dumping process, managing the entire serialization pipeline from object graph traversal to file generation.

#### Class Structure
```cpp
class Rps_Dumper {
public:
    // Directory and file management
    std::string du_topdir;                    // Top-level dump directory
    std::string du_curworkdir;                // Current working directory
    Json::StreamWriterBuilder du_jsonwriterbuilder; // JSON formatting

    // Thread safety
    std::recursive_mutex du_mtx;              // Main mutex for thread safety

    // Object graph management
    std::unordered_map<Rps_Id, Rps_ObjectRef, Rps_Id::Hasher> du_mapobjects;
    std::deque<Rps_ObjectRef> du_scanque;     // Objects to be scanned

    // File management
    std::string du_tempsuffix;                // Temporary file suffix
    std::set<std::string> du_openedpathset;   // Opened temporary files

    // Statistics and timing
    long du_newobcount;                       // Count of new objects dumped
    double du_startelapsedtime;               // Timing information
    double du_startprocesstime;
    double du_startwallclockrealtime;
    double du_startmonotonictime;

    // Space management
    std::map<Rps_ObjectRef, std::shared_ptr<du_space_st>> du_spacemap;

    // Object categorization
    std::set<Rps_ObjectRef> du_pluginobset;   // Plugin objects
    std::set<Rps_ObjectRef> du_constantobset; // Constant objects

    // Internal space structure
    struct du_space_st {
        Rps_Id sp_id;                         // Space identifier
        std::set<Rps_ObjectRef> sp_setob;     // Objects in this space
    };
};
```

#### Constructor and Initialization
```cpp
Rps_Dumper::Rps_Dumper(const std::string& topdir, Rps_CallFrame* callframe)
```
**Purpose**: Initializes the dumper with directory paths and timing information
- Resolves real paths using `realpath()`
- Determines if dumping into the source directory
- Sets up JSON writer with compact formatting
- Records start times for performance tracking
- Validates main thread execution

**Key Initialization Steps**:
1. Resolve and validate dump directory path
2. Get current working directory
3. Configure JSON writer (no comments, space indentation)
4. Record timing information
5. Set up temporary file suffix with PID

## Object Graph Scanning

### Root Object Scanning
```cpp
void Rps_Dumper::scan_roots(void)
```
**Purpose**: Begins the object graph traversal from root objects
- Iterates through all root objects using `rps_each_root_object()`
- Adds each root to the scan queue via `scan_object()`
- Logs the total number of root objects found

### Object Scanning Process
```cpp
void Rps_Dumper::scan_object(const Rps_ObjectRef obr)
```
**Purpose**: Registers an object for dumping and tracks new objects
- Checks for already scanned objects using `du_mapobjects`
- Validates object has a space (not transient)
- Records modification time to identify new objects
- Adds to scan queue for content processing

**New Object Detection**:
- Compares object modification time with system start time
- Increments `du_newobcount` for newly created objects
- Logs new object information for debugging

### Scan Loop Processing
```cpp
void Rps_Dumper::scan_loop_pass(void)
Rps_ObjectRef Rps_Dumper::pop_object_to_scan(void)
void Rps_Dumper::scan_object_contents(Rps_ObjectRef obr)
```
**Purpose**: Processes the scan queue to traverse the entire object graph
- `scan_loop_pass()`: Main processing loop, pops and scans objects
- `pop_object_to_scan()`: Thread-safe queue management
- `scan_object_contents()`: Delegates to object's `dump_scan_contents()` method

**Traversal Algorithm**:
1. Pop object from scan queue
2. Call object's content scanning method
3. Add object's space to space map
4. Repeat until queue is empty

### Value Scanning
```cpp
void Rps_Dumper::scan_value(const Rps_Value val, unsigned depth)
```
**Purpose**: Recursively scans values for referenced objects
- Handles pointer values by calling their `dump_scan()` method
- Prevents infinite recursion with depth parameter
- Thread-safe with mutex protection

## JSON Serialization

### Value Serialization
```cpp
Json::Value Rps_Dumper::json_value(Rps_Value val)
```
**Purpose**: Converts RefPerSys values to JSON representation
- **Integers**: Converted to `Json::Int64`
- **Pointers**: Delegates to object's `dump_json()` method
- **Null/Empty**: Returns `Json::nullValue`

### Object Reference Serialization
```cpp
Json::Value Rps_Dumper::json_objectref(Rps_ObjectRef obr)
```
**Purpose**: Serializes object references as JSON strings
- Returns object ID as string value
- Returns `null` for invalid objects

### Dumpability Checks
```cpp
bool Rps_Dumper::is_dumpable_objref(const Rps_ObjectRef obr)
bool Rps_Dumper::is_dumpable_objattr(const Rps_ObjectRef obr)
bool Rps_Dumper::is_dumpable_value(const Rps_Value val)
```
**Purpose**: Determines if objects/values can be serialized
- `is_dumpable_objref()`: Checks if object is in dump map and has space
- `is_dumpable_objattr()`: Currently stub implementation (marked incomplete)
- `is_dumpable_value()`: Validates different value types (closures, instances, objects)

**Value Dumpability Logic**:
- **Closures**: Must not be transient and have dumpable connector
- **Instances**: Must not be transient and have dumpable connector
- **Objects**: Must be dumpable object references

## File Generation System

### Generated File Types

#### Roots File (`generated/rps-roots.hh`)
```cpp
void Rps_Dumper::write_generated_roots_file(void)
```
**Purpose**: Generates C++ header with root object definitions
- Creates `RPS_INSTALL_ROOT_OB(Oid)` macros for each root object
- Includes object class information and hash values
- Groups objects with progress indicators

**Format Example**:
```cpp
RPS_INSTALL_ROOT_OB(_1Io89yIORqn02SXx4p) //RefPerSys_system∈the_system_class h:12345
```

#### Names File (`generated/rps-names.hh`)
```cpp
void Rps_Dumper::write_generated_names_file(void)
```
**Purpose**: Generates named symbol definitions
- Creates `RPS_INSTALL_NAMED_ROOT_OB(Oid,Name)` macros
- Only includes objects with non-weak symbol payloads
- Provides symbol name mappings

#### Constants File (`generated/rps-constants.hh`)
```cpp
void Rps_Dumper::write_generated_constants_file(void)
```
**Purpose**: Generates constant object definitions
- Includes objects from system constants and source file scanning
- Creates `RPS_INSTALL_CONSTANT_OB(Oid)` macros
- Provides class and name information where available

#### Data File (`generated/rpsdata-*.h`)
```cpp
void Rps_Dumper::write_generated_data_file(void)
```
**Purpose**: Generates system-specific data definitions
- Creates platform-specific header files (`rpsdata_OS_ARCH.h`)
- Includes size and alignment information for all C++ types
- Contains system configuration constants
- Generates symlink to `rpsdata.h`

**Included Information**:
- **Sizes**: `sizeof()` for all fundamental and RefPerSys types
- **Alignments**: `alignof()` for memory layout information
- **System Constants**: Page size, argument limits, POSIX versions
- **GNU Lightning**: JIT register counts (`JIT_R_NUM`, `JIT_V_NUM`, `JIT_F_NUM`)

#### RGB Colors File (`generated/rps-rgb-colors.hh`)
```cpp
void Rps_Dumper::write_generated_rgb_colors_file(void)
```
**Purpose**: Generates color definitions from source code macros
- Parses color definitions from header files
- Creates `RPS_RGB_COLOR(R,G,B,Name)` macros
- Tracks color name width for formatting

#### Parser Files (Incomplete)
```cpp
void Rps_Dumper::write_generated_parser_decl_file(Rps_CallFrame*, Rps_ObjectRef)
void Rps_Dumper::write_generated_parser_impl_file(Rps_CallFrame*, Rps_ObjectRef)
```
**Status**: Stub implementations marked as incomplete
- Intended for parser code generation
- Currently emit warning messages only

## Space and Manifest Files

### Space Files (`persistore/sp_*.json`)
```cpp
void Rps_Dumper::write_space_file(Rps_ObjectRef spacobr)
```
**Purpose**: Serializes objects belonging to a specific space
- Creates JSON files for each space with objects
- Includes prologue with space metadata
- Serializes each object's content with comments

**File Structure**:
```json
{
  "format": "RefPerSysFormat2024A",
  "spaceid": "space_oid",
  "nbobjects": 42,
  "rpsmajorversion": 0,
  "rpsminorversion": 6
}
//+ob object_oid:name
{
  "oid": "object_oid",
  "mtime": 123.45,
  "class": "class_oid",
  // ... object content ...
}
//-ob object_oid:name
```

### Manifest File (`rps_manifest.json`)
```cpp
void Rps_Dumper::write_manifest_file(void)
```
**Purpose**: Creates the main manifest describing the dump
- Contains global metadata and object lists
- References all space files and generated files
- Includes version information and timestamps

**Manifest Contents**:
- Format version and JSON-CPP version
- Major/minor version numbers
- Global root objects array
- Space set with space IDs
- Constant objects set
- Named symbols mapping
- Program metadata (name, timestamp, MD5 sum)

## Plugin and Constant Discovery

### Source File Scanning
```cpp
int Rps_Dumper::scan_source_file_for_constants(const std::string& relfilename)
void Rps_Dumper::scan_every_source_file_for_constants(void)
```
**Purpose**: Discovers constant objects defined in source files
- Scans C++ source files for `RPS_CONSTANTOBJ_PREFIX` patterns
- Parses object IDs following the prefix
- Validates found objects and adds to constant set

**Scanning Process**:
1. Read source file line by line
2. Search for constant object prefix
3. Extract and validate object IDs
4. Add valid objects to `du_constantobset`

### System Constants
```cpp
void Rps_Dumper::add_constants_known_from_RefPerSys_system(void)
```
**Purpose**: Adds constants from the system object
- Accesses `RefPerSys_system` object's constant attribute
- Iterates through the constant set
- Adds all system-known constants to dump set

### Plugin Discovery
```cpp
void Rps_Dumper::scan_code_addr(const void* ad)
```
**Purpose**: Discovers plugin objects from loaded code addresses
- Uses `dladdr()` to resolve code addresses to shared libraries
- Matches plugin naming patterns (`__rps_OS_ARCH_mod.so`)
- Extracts plugin object IDs and registers them

**Plugin Pattern Matching**:
- Parses filename for plugin OID using `sscanf()`
- Validates plugin object existence
- Adds to plugin object set for serialization

## File Management

### Temporary File System
```cpp
std::string Rps_Dumper::make_temporary_suffix(void)
std::string rps_dumper_temporary_path(Rps_Dumper* du, std::string shortpath)
```
**Purpose**: Manages temporary files during dump process
- Creates unique suffixes with random ID and PID
- Generates temporary file paths for atomic writes
- Validates path safety (no leading dots, proper characters)

### File Operations
```cpp
std::unique_ptr<std::ofstream> Rps_Dumper::open_output_file(const std::string& relpath)
void Rps_Dumper::rename_opened_files(void)
```
**Purpose**: Safe file writing with atomic operations
- Opens files with temporary suffixes
- Tracks opened files in `du_openedpathset`
- Renames temporary files to final names atomically

**Atomic Rename Process**:
1. Write to temporary file (`.temp`)
2. Backup existing file (`.backup`)
3. Rename temporary to final name
4. Clean up backup files

## Space Management

### Space Registration
```cpp
void Rps_Dumper::scan_space_component(Rps_ObjectRef obrspace, Rps_ObjectRef obrcomp)
```
**Purpose**: Associates objects with their containing spaces
- Creates space entries in `du_spacemap` if needed
- Adds component objects to space's object set
- Thread-safe with mutex protection

### Space File Generation
```cpp
void Rps_Dumper::write_all_space_files(void)
```
**Purpose**: Generates all space files for the dump
- Iterates through all spaces in `du_spacemap`
- Calls `write_space_file()` for each space
- Reports total number of space files created

## Main Dump Function

### Entry Point
```cpp
void rps_dump_into(std::string dirpath, Rps_CallFrame* callframe)
```
**Purpose**: Main entry point for the dumping process
- Validates and creates dump directory
- Initializes `Rps_Dumper` instance
- Orchestrates the complete dump pipeline

**Dump Pipeline**:
1. **Setup**: Create directories (`persistore/`, `generated/`)
2. **Scanning**: Roots → constants → source files → object graph traversal
3. **Writing**: Space files → generated files → manifest
4. **Finalization**: Rename temporary files, sync to disk

**Error Handling**:
- Comprehensive exception catching
- Detailed error messages with backtraces
- Directory creation and permission validation

## Object Serialization Methods

### Tuple Serialization
```cpp
void Rps_TupleOb::dump_scan(Rps_Dumper* du, unsigned) const
Json::Value Rps_TupleOb::dump_json(Rps_Dumper* du) const
```
**Purpose**: Serializes tuple objects
- Scans all tuple elements for referenced objects
- Creates JSON with `vtype: "tuple"` and `comp` array

### Set Serialization
```cpp
void Rps_SetOb::dump_scan(Rps_Dumper* du, unsigned) const
Json::Value Rps_SetOb::dump_json(Rps_Dumper* du) const
```
**Purpose**: Serializes set objects
- Scans all set elements for referenced objects
- Creates JSON with `vtype: "set"` and `elem` array

### Closure Serialization
```cpp
void Rps_ClosureZone::dump_scan(Rps_Dumper* du, unsigned depth) const
Json::Value Rps_ClosureZone::dump_json(Rps_Dumper* du) const
```
**Purpose**: Serializes closure objects
- Scans connector object and environment values
- Includes metadata (metaobject, metarank) if present
- Creates JSON with function reference and environment array

### Instance Serialization
```cpp
void Rps_InstanceZone::dump_scan(Rps_Dumper* du, unsigned depth) const
Json::Value Rps_InstanceZone::dump_json(Rps_Dumper* du) const
```
**Purpose**: Serializes instance objects
- Scans all attribute values and components
- Handles both named attributes and indexed components
- Creates complex JSON structure with class, attributes, and components

### Object Reference Serialization
```cpp
void Rps_ObjectZone::dump_scan(Rps_Dumper* du, unsigned) const
Json::Value Rps_ObjectZone::dump_json(Rps_Dumper* du) const
```
**Purpose**: Basic object serialization
- Delegates scanning to dumper
- Returns object ID as JSON string

## Integration with GNU Lightning

### JIT Metadata
The file includes GNU lightning constants for runtime code generation metadata:
```cpp
#define RPS_LIGHTNING_JIT_R_NUM JIT_R_NUM    // Integer registers
#define RPS_LIGHTNING_JIT_V_NUM JIT_V_NUM    // Vector registers
#define RPS_LIGHTNING_JIT_F_NUM JIT_F_NUM    // Float registers
```

**Purpose**: Provides JIT compilation information for generated data files
- Used in cross-compilation scenarios
- Enables runtime code generation metadata persistence

## Performance and Statistics

### Timing Information
The dumper tracks multiple time sources:
- **Elapsed Time**: Wall clock time since dump start
- **Process Time**: CPU time used by the process
- **Wall Clock Time**: Real time reference point
- **Monotonic Time**: System monotonic clock

### Statistics Tracking
- **New Object Count**: Objects created since system start
- **Space Count**: Number of spaces processed
- **File Counts**: Generated files and space files
- **Constant Counts**: Discovered constants and plugins

### Performance Logging
```cpp
RPS_INFORMOUT("dump into " << dumper.get_top_dir()
              << " completed in " << (endelapsed-startelapsed) << " wallclock, "
              << (endcputime-startcputime) << " cpu seconds"
              << " with " << dumper.du_newobcount << " new objects dumped");
```

## Implementation Status and TODOs

### Completed Features
- ✅ Complete object graph traversal and scanning
- ✅ JSON serialization for all value types
- ✅ Space-based file organization
- ✅ Generated C++ header files (roots, names, constants, data)
- ✅ Manifest file generation
- ✅ Plugin and constant discovery
- ✅ Temporary file management with atomic operations
- ✅ Comprehensive error handling and logging

### Incomplete Implementations
- ❌ `is_dumpable_objattr()`: Stub implementation
- ❌ Parser file generation: Empty implementations with warnings
- ❌ `copy_one_source_file()`: Fatal error for unimplemented feature
- ❌ `make_source_directory()`: Fatal error for unimplemented feature
- ❌ Code generation integration: `rpsapply_5Q5E0Lw9v4f046uAKZ` is stub

### Known Warnings and TODOs
- **Line 243**: Temporary dump object suggestion
- **Line 439**: Incomplete `is_dumpable_objattr` implementation
- **Line 467**: Partial `is_dumpable_value` implementation
- **Line 1017**: Unimplemented source file copying
- **Line 1032**: Unimplemented directory creation
- **Line 1479**: TODO for symlink creation
- **Line 1569**: Parser declaration file needs implementation
- **Line 1588**: Parser implementation file needs implementation
- **Line 2010**: Temporary dumper object suggestion
- **Line 2133**: Unimplemented code generation method

### Future Enhancements
- Complete parser code generation
- Source file copying for full persistence
- Enhanced attribute dumpability checks
- Code generation integration completion

## Usage Patterns

### Basic Dumping
```cpp
// Dump current system state to directory
rps_dump_into("/path/to/dump/dir", callframe);
```

### Accessing Dump Information
```cpp
// Get timing information
double elapsed = rps_dump_start_elapsed_time(dumper);
double process = rps_dump_start_process_time(dumper);

// Check dumpability
bool can_dump = rps_is_dumpable_objref(dumper, my_object);
bool can_dump_val = rps_is_dumpable_value(dumper, my_value);

// Get JSON representation
Json::Value json_obj = rps_dump_json_objectref(dumper, my_object);
Json::Value json_val = rps_dump_json_value(dumper, my_value);
```

### Custom JSON Formatting
```cpp
// Configure JSON writer
Json::StreamWriterBuilder builder;
builder["commentStyle"] = "None";
builder["indentation"] = " ";

// Create JSON string
std::string json_str = rps_dump_json_to_string(json_value);
```

## Design Rationale

### Space-Based Organization
- **Modularity**: Objects grouped by conceptual spaces
- **Load Efficiency**: Allows selective loading of object subsets
- **Scalability**: Supports large object graphs through partitioning

### Temporary File Strategy
- **Atomicity**: Prevents partial dumps from corruption
- **Safety**: Backup and restore mechanism for existing files
- **Performance**: Minimizes I/O conflicts during dump process

### Comprehensive Scanning
- **Completeness**: Ensures all reachable objects are serialized
- **Reference Integrity**: Maintains object relationships through ID references
- **Incremental Awareness**: Tracks new vs. existing objects

### Generated File Architecture
- **Bootstrap**: Generated headers enable system self-description
- **Compilation**: C++ headers provide compile-time knowledge
- **Platform Adaptation**: Data files capture system-specific information

This implementation provides a robust persistence layer for RefPerSys, enabling the system to maintain state across executions while supporting complex object graphs, plugin architectures, and generated code integration.