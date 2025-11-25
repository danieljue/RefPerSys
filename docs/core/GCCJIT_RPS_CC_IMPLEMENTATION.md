# GNU libgccjit Integration Implementation Analysis (`gccjit_rps.cc`)

## Overview

The `gccjit_rps.cc` file implements the **GNU libgccjit** integration for machine code generation in the Reflective Persistent System (RefPerSys), providing just-in-time (JIT) compilation capabilities. This file serves as the JIT compilation backend for RefPerSys, enabling runtime code generation and execution through GCC's JIT compilation infrastructure.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2023-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"
#include "libgccjit.h"

// Git versioning information
extern "C" const char rps_gccjit_gitid[];
extern "C" const char rps_gccjit_date[];
extern "C" const char rps_gccjit_shortgitid[];
extern "C" const char rps_gccjit_timestamp[];
```

### Key Dependencies
- **libgccjit**: GNU JIT compilation library (C API)
- **RefPerSys core**: Object system and payload infrastructure
- **GCC infrastructure**: Leverages GCC's compilation pipeline

### Global State
```cpp
// Top-level GCCJIT context (shared across the system)
extern "C" struct gcc_jit_context* rps_gccjit_top_ctxt;

// Naming prefixes for generated code
extern "C" const std::string rps_gccjit_prefix_struct;
extern "C" const std::string rps_gccjit_prefix_field;
```

## Rps_PayloadGccjit Class

The `Rps_PayloadGccjit` class is the core payload for JIT code generation, managing GCCJIT contexts, type systems, and object-to-JIT mappings.

### Class Structure
```cpp
class Rps_PayloadGccjit : public Rps_Payload {
    struct gcc_jit_context* _gji_ctxt;  // Child context for code generation
    std::map<Rps_ObjectRef, struct gcc_jit_object*> _gji_rpsobj2jit; // Object mappings

public:
    // Core payload methods
    virtual void gc_mark(Rps_GarbageCollector&gc) const;
    virtual void dump_scan(Rps_Dumper*du) const;
    virtual void dump_json_content(Rps_Dumper*du, Json::Value&jv) const;
    virtual const std::string payload_type_name(void) const;
    virtual uint32_t wordsize(void) const;
    virtual bool is_erasable(void) const;
    virtual ~Rps_PayloadGccjit();

    // Location management
    struct gcc_jit_location* make_csrc_location(const char*filename, int line, int col);
    struct gcc_jit_location* make_string_src_location(const std::string&filen, int line, int col);
    struct gcc_jit_location* no_src_location(void) const;
    struct gcc_jit_location* json_to_src_location(const Json::Value &jv);
    Json::Value src_location_to_json(struct gcc_jit_location*loc) const;
    struct gcc_jit_location* make_rpsobj_location(Rps_ObjectRef ob, int line, int col=0);

    // Object registration
    void locked_register_object_jit(Rps_ObjectRef ob, struct gcc_jit_object* jit);
    void locked_unregister_object_jit(Rps_ObjectRef ob);

protected:
    // Internal methods
    void load_jit_json(Rps_Loader*ld, Rps_Id spacid, unsigned lineno, Json::Value&jseq);
    void raw_register_object_jit(Rps_ObjectRef ob, struct gcc_jit_object* jit);
    void raw_unregister_object_jit(Rps_ObjectRef ob);

    // Type creation methods (detailed below)
    // ...
};
```

### Constructor and Initialization
```cpp
Rps_PayloadGccjit::Rps_PayloadGccjit(Rps_ObjectZone* owner)
    : Rps_Payload(Rps_Type::PaylGccjit, owner),
      _gji_ctxt(rps_gccjit_top_ctxt),
      _gji_rpsobj2jit()
```
**Purpose**: Initialize payload with shared top-level context
- **Thread Safety**: Constructor lacks proper synchronization (marked incomplete)
- **Context Sharing**: Uses global `rps_gccjit_top_ctxt` for all payloads
- **Mapping Initialization**: Empty object-to-JIT mapping

## GCCJIT Context Management

### Global Context Initialization
```cpp
void rps_gccjit_initialize(void)
```
**Purpose**: Set up the global GCCJIT context at system startup
- **Thread Safety**: Must be called from main thread
- **Context Acquisition**: `gcc_jit_context_acquire()` creates top-level context
- **Lifetime**: Context persists for entire program execution

### Context Finalization
```cpp
void rps_gccjit_finalize(void)
```
**Purpose**: Clean up GCCJIT resources at program termination
- **Thread Safety**: Uses atomic flag to prevent multiple finalizations
- **Resource Release**: `gcc_jit_context_release()` frees all resources
- **Registration**: Called via `atexit()` mechanism

## Type System Integration

### Builtin Types
```cpp
struct gcc_jit_type* raw_get_gccjit_builtin_type(enum gcc_jit_types gcty)
struct gcc_jit_type* locked_get_gccjit_builtin_type(enum gcc_jit_types gcty)
```
**Purpose**: Access GCCJIT's builtin types (int, float, void, etc.)
- **Raw vs Locked**: Raw methods don't lock, locked methods do
- **Thread Safety**: Locked variants protect against concurrent access
- **GCC Types**: Maps to `enum gcc_jit_types` (GCC_JIT_TYPE_INT, etc.)

### Derived Types

#### Pointer Types
```cpp
struct gcc_jit_type* raw_get_gccjit_pointer_type(struct gcc_jit_type* srcty)
struct gcc_jit_type* locked_get_gccjit_pointer_type(struct gcc_jit_type* srcty)
```
**Purpose**: Create pointer types from base types
- **GCC API**: Uses `gcc_jit_type_get_pointer()`
- **Memory Model**: Follows GCC's pointer semantics

#### Qualified Types
```cpp
struct gcc_jit_type* raw_get_gccjit_const_type(struct gcc_jit_type* srcty)
struct gcc_jit_type* locked_get_gccjit_const_type(struct gcc_jit_type* srcty)
struct gcc_jit_type* raw_get_gccjit_volatile_type(struct gcc_jit_type* srcty)
struct gcc_jit_type* locked_get_gccjit_volatile_type(struct gcc_jit_type* srcty)
```
**Purpose**: Create const and volatile qualified types
- **Type Qualifiers**: Standard C type qualifiers
- **GCC API**: `gcc_jit_type_get_const()`, `gcc_jit_type_get_volatile()`

#### Aligned Types
```cpp
struct gcc_jit_type* raw_get_gccjit_aligned_type(struct gcc_jit_type* srcty, size_t alignment)
struct gcc_jit_type* locked_get_gccjit_aligned_type(struct gcc_jit_type* srcty, size_t alignment)
```
**Purpose**: Create types with specific alignment requirements
- **Alignment**: Must be power of two
- **GCC API**: `gcc_jit_type_get_aligned()`
- **Memory Layout**: Controls structure field alignment

### Array Types
```cpp
struct gcc_jit_type* raw_new_gccjit_array_type(struct gcc_jit_type* elemtype, int nbelem, struct gcc_jit_location* loc = nullptr)
struct gcc_jit_type* locked_new_gccjit_array_type(struct gcc_jit_type* elemtype, int nbelem, struct gcc_jit_location* loc = nullptr)
```
**Purpose**: Create array types with element type and size
- **Dimensions**: Single dimension arrays
- **Source Location**: Optional debugging information
- **GCC API**: `gcc_jit_context_new_array_type()`

### Struct Types

#### Opaque Structs
```cpp
struct gcc_jit_struct* raw_new_gccjit_opaque_struct(const std::string& strname, struct gcc_jit_location* loc = nullptr)
struct gcc_jit_struct* locked_new_gccjit_opaque_struct(const std::string& strname, struct gcc_jit_location* loc = nullptr)
```
**Purpose**: Create forward-declared struct types
- **Naming**: String-based struct names
- **Deferred Definition**: Structs can be defined later
- **GCC API**: `gcc_jit_context_new_opaque_struct()`

#### Object-Based Structs
```cpp
struct gcc_jit_struct* raw_new_gccjit_opaque_struct(const Rps_ObjectRef ob, struct gcc_jit_location* loc = nullptr)
struct gcc_jit_struct* locked_new_gccjit_opaque_struct(const Rps_ObjectRef ob, struct gcc_jit_location* loc = nullptr)
```
**Purpose**: Create structs named after RefPerSys objects
- **Naming Convention**: `rps_gccjit_prefix_struct + oid.to_string()`
- **Object Registration**: Locked version registers the object-JIT mapping
- **Identity Preservation**: Links RefPerSys objects to GCC types

### Field Creation
```cpp
struct gcc_jit_field* raw_new_gccjit_field(struct gcc_jit_type* type, const std::string& name, struct gcc_jit_location* loc = nullptr)
struct gcc_jit_field* locked_new_gccjit_field(struct gcc_jit_type* type, const std::string& name, struct gcc_jit_location* loc = nullptr)
struct gcc_jit_field* raw_new_gccjit_field(struct gcc_jit_type* type, const Rps_ObjectRef obf, struct gcc_jit_location* loc = nullptr)
struct gcc_jit_field* locked_new_gccjit_field(struct gcc_jit_type* type, const Rps_ObjectRef obf, struct gcc_jit_location* loc = nullptr)
```
**Purpose**: Create struct fields with types and names
- **Naming Variants**: String names or object-based names
- **Object Fields**: Uses `rps_gccjit_prefix_field + oid.to_string()`
- **GCC API**: `gcc_jit_context_new_field()`

## Object-to-JIT Mapping

### Registration System
```cpp
void raw_register_object_jit(Rps_ObjectRef ob, struct gcc_jit_object* jit)
void locked_register_object_jit(Rps_ObjectRef ob, struct gcc_jit_object* jit)
```
**Purpose**: Associate RefPerSys objects with GCCJIT objects
- **Mapping Storage**: `_gji_rpsobj2jit` map maintains associations
- **Thread Safety**: Locked version protects concurrent access
- **Object Lifecycle**: Tracks JIT objects corresponding to RefPerSys objects

### Unregistration
```cpp
void raw_unregister_object_jit(Rps_ObjectRef ob)
void locked_unregister_object_jit(Rps_ObjectRef ob)
```
**Purpose**: Remove object-JIT associations
- **Cleanup**: Removes entries from mapping
- **Thread Safety**: Locked version with proper synchronization
- **Resource Management**: Called during object cleanup

## Source Location Management

### Location Creation
```cpp
struct gcc_jit_location* make_csrc_location(const char* filename, int line, int col)
struct gcc_jit_location* make_string_src_location(const std::string& filen, int line, int col)
struct gcc_jit_location* make_rpsobj_location(Rps_ObjectRef ob, int line, int col = 0)
```
**Purpose**: Create source location information for debugging
- **File-based**: Standard filename/line/column locations
- **Object-based**: Locations tied to RefPerSys objects
- **GCC API**: `gcc_jit_context_new_location()`

### JSON Serialization
```cpp
struct gcc_jit_location* json_to_src_location(const Json::Value& jv)
Json::Value src_location_to_json(struct gcc_jit_location* loc) const
```
**Purpose**: Convert between JSON and GCCJIT location representations
- **JSON Format**: `{"srcloc_file": "filename", "srcloc_line": 123, "srcloc_col": 45}`
- **GCC API**: Location creation and inspection
- **Status**: `src_location_to_json()` is unimplemented (fatal error)

## Persistence Integration

### Garbage Collection
```cpp
void gc_mark(Rps_GarbageCollector& gc) const
```
**Purpose**: Mark RefPerSys objects referenced by JIT mappings
- **Object Iteration**: Traverses `_gji_rpsobj2jit` map
- **GC Integration**: Marks all referenced objects
- **Status**: Marked as incomplete

### Dumping Support
```cpp
void dump_scan(Rps_Dumper* du) const
void dump_json_content(Rps_Dumper* du, Json::Value& jv) const
```
**Purpose**: Support persistence of JIT state
- **Object Scanning**: Registers all mapped objects for dumping
- **JSON Export**: Creates array of object references
- **Status**: Both methods marked as incomplete

### Loading Support
```cpp
void load_jit_json(Rps_Loader* ld, Rps_Id spacid, unsigned lineno, Json::Value& jseq)
void rpsldpy_gccjit(Rps_ObjectZone* obz, Rps_Loader* ld, const Json::Value& jv, Rps_Id spacid, unsigned lineno)
```
**Purpose**: Restore JIT state from persistent storage
- **JSON Processing**: Parse JIT sequence from dump
- **Object Recreation**: Rebuild GCCJIT objects and mappings
- **Status**: Both functions are unimplemented (fatal errors)

## Naming Conventions

### Prefix Definitions
```cpp
const std::string rps_gccjit_prefix_struct = "_rps_STRUCT";
const std::string rps_gccjit_prefix_field = "_rps_FIELD";
```

### Generated Names
- **Struct Names**: `_rps_STRUCT` + object OID string
- **Field Names**: `_rps_FIELD` + object OID string
- **Location Names**: Object OID as filename for object-based locations

## Thread Safety Architecture

### Synchronization Patterns
- **Owner Mutex**: All operations lock the payload owner's mutex
- **Object Mutex**: Operations on specific objects lock their mutex
- **Raw vs Locked**: Raw methods assume external synchronization
- **Locked Methods**: Provide internal synchronization

### Context Sharing
- **Global Context**: `rps_gccjit_top_ctxt` shared across all payloads
- **Child Contexts**: Each payload could have its own context (not implemented)
- **Thread Safety**: Context operations may require synchronization

## Implementation Status and TODOs

### Completed Features
- ✅ GCCJIT context management (initialize/finalize)
- ✅ Type system integration (builtins, pointers, arrays, structs)
- ✅ Object-to-JIT mapping and registration
- ✅ Source location creation and JSON conversion (partial)
- ✅ Basic payload structure and lifecycle
- ✅ Naming conventions and prefixes
- ✅ Thread safety infrastructure

### Incomplete Implementations
- ❌ **Source Location JSON**: `src_location_to_json()` fatal error
- ❌ **JIT JSON Loading**: `load_jit_json()` fatal error
- ❌ **Persistence Loader**: `rpsldpy_gccjit()` incomplete
- ❌ **GC Marking**: `gc_mark()` incomplete
- ❌ **Dump Scanning**: `dump_scan()` incomplete
- ❌ **JSON Content Dump**: `dump_json_content()` incomplete
- ❌ **Payload Constructor**: Thread safety issues
- ❌ **Context Management**: Child context creation
- ❌ **Code Generation**: Actual JIT compilation and execution
- ❌ **Struct Definition**: Setting struct fields after creation
- ❌ **Function Creation**: JIT function generation
- ❌ **Variable Management**: JIT variable handling
- ❌ **Compilation**: Converting contexts to executable code

### Known Warnings and TODOs
- **Line 87**: TODO to document GCCJIT code representation
- **Line 183**: Incomplete payload constructor (thread safety)
- **Line 451**: Unimplemented `src_location_to_json`
- **Line 468**: Unimplemented `load_jit_json`
- **Line 482**: Incomplete GC marking
- **Line 494**: Incomplete dump scanning
- **Line 514**: Incomplete JSON content dumping
- **Line 556**: Incomplete persistence loader
- **Line 575**: Incomplete finalization

### Future Enhancements
- Complete source location JSON serialization
- Implement full JIT state persistence and loading
- Add struct field definition capabilities
- Implement function and variable creation
- Add compilation and code execution
- Support for GCCJIT expressions and statements
- Integration with RefPerSys object system for code objects
- Debugging and optimization support
- Plugin generation for dlopen-able modules

## Usage Patterns

### Basic Payload Creation
```cpp
// Create a GCCJIT payload on an object
auto payload = object->put_new_plain_payload<Rps_PayloadGccjit>();
```

### Type Creation
```cpp
// Get builtin types
auto int_type = payload->locked_get_gccjit_builtin_type(GCC_JIT_TYPE_INT);

// Create derived types
auto ptr_type = payload->locked_get_gccjit_pointer_type(int_type);
auto arr_type = payload->locked_new_gccjit_array_type(int_type, 10);

// Create struct types
auto struct_type = payload->locked_new_gccjit_opaque_struct("my_struct");
auto obj_struct = payload->locked_new_gccjit_opaque_struct(my_object);
```

### Object Registration
```cpp
// Register a RefPerSys object with its JIT counterpart
payload->locked_register_object_jit(my_object, jit_object);

// Later unregister
payload->locked_unregister_object_jit(my_object);
```

### Source Locations
```cpp
// Create file-based location
auto file_loc = payload->make_csrc_location("myfile.c", 42, 10);

// Create object-based location
auto obj_loc = payload->make_rpsobj_location(my_object, 1, 5);
```

### JSON Location Handling
```cpp
// Convert JSON to location
Json::Value jloc = R"({"srcloc_file": "test.c", "srcloc_line": 10})"_json;
auto loc = payload->json_to_src_location(jloc);

// Convert location to JSON (unimplemented)
Json::Value json_loc = payload->src_location_to_json(loc); // Fatal error
```

## Design Rationale

### C API Usage
- **Stability**: C API less likely to change than C++ API
- **Compatibility**: Avoids deprecation issues with C++ API
- **Control**: Direct access to GCCJIT internals

### Payload-Based Architecture
- **Object Integration**: JIT state tied to RefPerSys objects
- **Persistence**: Payloads participate in GC and dumping
- **Modularity**: Each payload manages its own JIT context

### Thread Safety Design
- **Owner Locking**: Payload operations lock owner object
- **Object Locking**: Operations on specific objects lock them
- **Hierarchical**: Prevents deadlocks through consistent ordering

### Naming Conventions
- **Uniqueness**: OID-based names ensure global uniqueness
- **Debugging**: Structured prefixes aid in debugging
- **Compatibility**: Valid C identifiers for generated code

This implementation provides a solid foundation for JIT code generation in RefPerSys, with clear pathways for completing the GCCJIT integration and enabling runtime code generation capabilities.