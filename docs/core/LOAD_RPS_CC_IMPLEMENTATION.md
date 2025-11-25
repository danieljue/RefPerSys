# Persistence Deserialization Engine Implementation Analysis (`load_rps.cc`)

## Overview

The `load_rps.cc` file implements the core **persistence deserialization engine** for the Reflective Persistent System (RefPerSys), providing the ability to restore the entire object graph from JSON format persistent storage. This file serves as the counterpart to the dumping functionality (`dump_rps.cc`), enabling RefPerSys to maintain state across executions and achieve true persistence.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2019-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// JSON parsing and validation
#include <json/json.h>

// Dynamic loading for plugins
#include <dlfcn.h>

// File system operations
#include <filesystem>
#include <sys/stat.h>

// Git versioning information
extern "C" const char rps_load_gitid[];
extern "C" const char rps_load_date[];
extern "C" const char rps_load_shortgitid[];
extern "C" const char rps_load_timestamp[];

// Global loaded directory
extern "C" char rps_loaded_directory[rps_path_byte_size];
```

### Key Dependencies
- **JSON-CPP**: JSON parsing and deserialization library
- **dlfcn.h**: Dynamic loading library for plugin management
- **RefPerSys core**: Object system, value types, and payload infrastructure
- **System libraries**: File operations, process management, and threading

## Rps_Loader Class

The `Rps_Loader` class is the central orchestrator for the persistence loading process, managing the complete restoration of RefPerSys state from persistent storage.

### Class Structure
```cpp
class Rps_Loader {
    std::string ld_topdir;                    // Top-level directory for loading
    double ld_startclock;                     // Loading start time
    std::recursive_mutex ld_mtx;              // Thread safety for plugin loading

    // Space and root object management
    std::set<Rps_Id> ld_spaceset;             // Set of space IDs to load
    std::set<Rps_Id> ld_globrootsidset;       // Set of global root object IDs

    // Plugin management
    std::map<Rps_Id, void*> ld_pluginsmap;    // Plugin ID to dlopen handle mapping

    // Object management
    std::map<Rps_Id, Rps_ObjectRef> ld_mapobjects; // Loaded objects by ID

    // Deferred initialization
    struct todo_st {
        double todo_addtime;                  // Time when todo was added
        std::function<void(Rps_Loader*)> todo_fun; // Deferred function
    };
    std::deque<struct todo_st> ld_todoque;    // Todo queue
    unsigned ld_todocount;                    // Total todos processed
    static constexpr unsigned ld_maxtodo = 1<<20; // Maximum todos

    // Payload loader caching
    std::map<std::string, rpsldpysig_t*> ld_payloadercache; // Cached payload loaders

public:
    // Core loading operations
    Rps_Loader(const std::string& topdir);
    ~Rps_Loader();

    // Manifest and configuration
    void parse_manifest_file(void);
    void parse_user_manifest(const std::string& path);

    // Two-pass loading
    void first_pass_space(Rps_Id spacid);
    void second_pass_space(Rps_Id spacid);
    void load_all_state_files(void);

    // Root object management
    void initialize_root_objects(void);
    void initialize_constant_objects(void);
    void load_install_roots(void);

    // Utility functions
    std::string string_of_loaded_file(const std::string& relpath);
    std::string space_file_path(Rps_Id spacid);
    std::string load_real_path(const std::string& path);
    Rps_ObjectRef find_object_by_oid(Rps_Id oid);

    // Deferred initialization
    void add_todo(const std::function<void(Rps_Loader*)>& todofun);
    int run_some_todo_functions(void);

    // Primitive type configuration
    void set_primitive_type_size_and_align(Rps_ObjectRef primtypob,
                                          unsigned sizeby, unsigned alignby);

    // Statistics
    unsigned nb_loaded_objects(void) const;

private:
    // Internal helpers
    bool is_object_starting_line(Rps_Id spacid, unsigned lineno,
                                const std::string& linbuf, Rps_Id* pobid);
    Rps_ObjectRef fetch_one_constant_at(const char* oidstr, int lin);
    void parse_json_buffer_second_pass(Rps_Id spacid, unsigned lineno,
                                      Rps_Id objid, const std::string& objbuf,
                                      unsigned count);
};
```

### Constructor and Initialization
```cpp
Rps_Loader::Rps_Loader(const std::string& topdir)
    : ld_topdir(topdir),
      ld_startclock(rps_wallclock_real_time()),
      ld_mtx(),
      ld_spaceset(),
      ld_globrootsidset(),
      ld_pluginsmap(),
      ld_mapobjects(),
      ld_todoque(),
      ld_todocount(0),
      ld_payloadercache()
{
    // Logging and initialization
}
```
**Purpose**: Initialize loader with top-level directory and timing
- **Directory Setup**: Establishes base path for all loading operations
- **Timing**: Records start time for performance measurement
- **Thread Safety**: Initializes mutex for plugin loading operations

## JSON Parsing and Validation

### String to JSON Conversion
```cpp
Json::Value rps_load_string_to_json(const std::string& str,
                                   const char* filnam, int lineno)
{
    Json::CharReaderBuilder jsonreaderbuilder;
    std::unique_ptr<Json::CharReader> pjsonreader(jsonreaderbuilder.newCharReader());
    Json::Value jv;
    JSONCPP_STRING errstr;

    if (!pjsonreader->parse(str.c_str(), str.c_str() + str.size(), &jv, &errstr)) {
        if (filnam != nullptr && lineno > 0) {
            RPS_WARNOUT("JSON parse failure (loading) at " << filnam << ":" << lineno
                        << std::endl << str);
        }
        throw std::runtime_error(std::string("JSON parsing error:") + errstr);
    }
    return jv;
}
```
**Purpose**: Parse JSON strings with comprehensive error reporting
- **JSON-CPP Integration**: Uses JsonCpp library for parsing
- **Error Context**: Includes file name and line number in error messages
- **Exception Safety**: Throws runtime_error on parsing failures

## Two-Pass Loading Algorithm

### First Pass: Object Allocation
```cpp
void Rps_Loader::first_pass_space(Rps_Id spacid)
{
    auto spacepath = load_real_path(space_file_path(spacid));
    std::ifstream ins(spacepath);

    // Parse space prologue (JSON header)
    Json::Value prologjson = rps_load_string_to_json(prologstr);

    // Validate format and version compatibility
    // Extract expected object count

    // Scan for object markers and allocate objects
    for (std::string linbuf; std::getline(ins, linbuf); ) {
        Rps_Id curobjid;
        if (is_object_starting_line(spacid, lincnt, linbuf, &curobjid)) {
            Rps_ObjectRef obref(Rps_ObjectZone::make_loaded(curobjid, this));
            ld_mapobjects.insert({curobjid, obref});
        }
    }
}
```
**Purpose**: Allocate all objects in a space without initializing them
- **Space File Parsing**: Reads JSON header and validates format
- **Object Discovery**: Scans for `//+ob_` markers to identify objects
- **Memory Allocation**: Creates object zones for all discovered objects
- **ID Mapping**: Builds object ID to reference mapping

### Second Pass: Object Initialization
```cpp
void Rps_Loader::second_pass_space(Rps_Id spacid)
{
    // Parse space file line by line
    for (std::string linbuf; std::getline(ins, linbuf); ) {
        Rps_Id curobjid;
        if (is_object_starting_line(spacid, lincnt, linbuf, &curobjid)) {
            // Collect JSON content for this object
            std::string objbuf = linbuf + '\n';
            // ... accumulate lines until next object ...

            // Parse and initialize object
            parse_json_buffer_second_pass(spacid, prevlin, prevoid, objbuf, obcnt);
        }
    }
}
```
**Purpose**: Initialize all objects with their data and relationships
- **JSON Parsing**: Converts accumulated JSON strings to object representations
- **Attribute Setting**: Restores object attributes, components, and metadata
- **Payload Loading**: Delegates to specialized payload loaders
- **Relationship Restoration**: Reconnects object references and values

### Object Starting Line Detection
```cpp
bool Rps_Loader::is_object_starting_line(Rps_Id spacid, unsigned lineno,
                                        const std::string& linbuf, Rps_Id* pobid)
{
    // Pattern: //+ob__XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    if (linbuf.substr(0, 7) != "//+ob_")
        return false;

    // Extract and validate object ID
    std::string oidstr = linbuf.substr(7, Rps_Id::nbchars);
    Rps_Id oid(oidstr);

    if (pobid) *pobid = oid;
    return oid.valid();
}
```
**Purpose**: Identify lines that mark the beginning of object definitions
- **Pattern Matching**: Recognizes `//+ob_` comment markers
- **ID Extraction**: Parses object IDs from fixed-width format
- **Validation**: Ensures extracted IDs are valid

## Plugin Loading and Dynamic Linking

### Plugin Discovery and Loading
```cpp
// Parse plugins from manifest
for (int ix = 0; ix < (int)sizeplugins; ix++) {
    std::string curpluginidstr = pluginsjson[ix].asString();
    Rps_Id curpluginid(curpluginidstr);

    // Build plugin paths
    std::string pluginsopath = load_real_path(std::string{"plugins/rps"}
                                             + curpluginid.to_string() + "-mod.so");
    std::string pluginsrcpath = load_real_path(std::string{"generated/rps"}
                                              + curpluginid.to_string() + "-mod.cc");

    // Build plugin if necessary
    if (need_rebuild) {
        std::string buildcmdstr = build_command_for_plugin(pluginsrcpath, pluginsopath);
        int notok = system(buildcmdstr.c_str());
        if (notok) RPS_FATALOUT("failed to build plugin");
    }

    // Load plugin with dlopen
    void* dlh = dlopen(pluginsopath.c_str(), RTLD_NOW | RTLD_GLOBAL);
    if (!dlh) RPS_FATAL("failed to load plugin");

    ld_pluginsmap.insert({curpluginid, dlh});
}
```
**Purpose**: Load and manage dynamic plugins during system initialization
- **Manifest Parsing**: Discovers plugins from JSON configuration
- **Build System Integration**: Automatically rebuilds outdated plugins
- **Dynamic Loading**: Uses `dlopen()` with global symbol visibility
- **Error Handling**: Comprehensive error reporting for build/load failures

## Space-Based Organization

### Space File Structure
```
persistore/sp_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX-rps.json
├── JSON Prologue (format, version, object count)
├── Object 1: //+ob__XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
│   └── JSON content for object data
├── Object 2: //+ob__XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
│   └── JSON content for object data
└── ...
```

### Space File Path Generation
```cpp
std::string Rps_Loader::space_file_path(Rps_Id spacid)
{
    return std::string{"persistore/sp"} + spacid.to_string() + "-rps.json";
}
```
**Purpose**: Generate standardized file paths for space persistence files
- **Naming Convention**: `sp_{spaceid}-rps.json` format
- **Directory Structure**: All spaces stored in `persistore/` directory

## Todo-Based Deferred Initialization

### Todo System Architecture
```cpp
struct todo_st {
    double todo_addtime;                          // Timestamp for ordering
    std::function<void(Rps_Loader*)> todo_fun;    // Deferred function
};

std::deque<struct todo_st> ld_todoque;            // FIFO queue
unsigned ld_todocount;                            // Processed count
static constexpr unsigned ld_maxtodo = 1<<20;    // Safety limit
```

### Todo Execution
```cpp
int Rps_Loader::run_some_todo_functions(void)
{
    // Process todos in time order with batching
    while (count < dosteps && elapsed_time < doelaps) {
        // Execute next todo function
        td.todo_fun(this);
        count++;
    }
    return remaining_todos;
}
```
**Purpose**: Execute deferred initialization functions in controlled batches
- **Time-Based Batching**: Limits execution time per batch
- **Count Limits**: Prevents infinite loops
- **Progress Tracking**: Returns remaining work count

### Todo Addition
```cpp
void Rps_Loader::add_todo(const std::function<void(Rps_Loader*)>& todofun)
{
    std::lock_guard<std::recursive_mutex> gu(ld_mtx);
    ld_todoque.push_back(todo_st{rps_elapsed_real_time(), todofun});
}
```
**Purpose**: Queue functions for deferred execution
- **Timestamp Ordering**: Maintains chronological order
- **Thread Safety**: Protected by loader mutex

## Payload Loaders

### Class Information Loader
```cpp
void rpsldpy_classinfo(Rps_ObjectZone* obz, Rps_Loader* ld,
                      const Json::Value& jv, Rps_Id spacid, unsigned lineno)
{
    auto paylclainf = obz->put_new_plain_payload<Rps_PayloadClassInfo>();

    // Set superclass
    auto obsuperclass = Rps_ObjectRef(jv["class_super"], ld);
    paylclainf->put_superclass(obsuperclass);

    // Set symbol name if present
    if (jv.isMember("class_symb")) {
        auto obsymb = Rps_ObjectRef(jv["class_symb"], ld);
        paylclainf->loader_put_symbname(obsymb, ld);
    }

    // Load method dictionary
    Json::Value jvmethodict = jv["class_methodict"];
    for (int methix = 0; methix < (int)jvmethodict.size(); methix++) {
        auto obsel = Rps_ObjectRef(jvmethodict[methix]["methosel"], ld);
        auto valclo = Rps_Value(jvmethodict[methix]["methclos"], ld);
        paylclainf->put_own_method(obsel, valclo);
    }

    // Load attribute set
    if (jv.isMember("class_attrset")) {
        auto valaset = Rps_Value(jv["class_attrset"], ld);
        paylclainf->loader_put_attrset(valaset.as_set(), ld);
    }
}
```
**Purpose**: Restore class information including inheritance, methods, and attributes
- **Superclass Restoration**: Reconnects inheritance hierarchy
- **Method Dictionary**: Restores method implementations
- **Attribute Set**: Reconstructs class attribute definitions

### Vector Payload Loaders
```cpp
void rpsldpy_vectob(Rps_ObjectZone* obz, Rps_Loader* ld,
                   const Json::Value& jv, Rps_Id spacid, unsigned lineno)
{
    Json::Value jvectob = jv["vectob"];
    auto paylvectob = obz->put_new_plain_payload<Rps_PayloadVectOb>();

    for (int elemix = 0; elemix < (int)jvectob.size(); elemix++) {
        auto obelem = Rps_ObjectRef(jvectob[elemix], ld);
        paylvectob->push_back(obelem);
    }
}

void rpsldpy_vectval(Rps_ObjectZone* obz, Rps_Loader* ld,
                    const Json::Value& jv, Rps_Id spacid, unsigned lineno)
{
    Json::Value jvectval = jv["vectval"];
    auto paylvectval = obz->put_new_plain_payload<Rps_PayloadVectVal>();

    for (int compix = 0; compix < (int)jvectval.size(); compix++) {
        auto compv = Rps_Value(jvectval[compix], ld);
        paylvectval->push_back(compv);
    }
}
```
**Purpose**: Restore vector payloads containing objects or values
- **Object Vectors**: Reconstruct sequences of object references
- **Value Vectors**: Restore sequences of arbitrary values
- **Capacity Management**: Pre-allocate appropriate sizes

### Set Payload Loader
```cpp
void rpsldpy_setob(Rps_ObjectZone* obz, Rps_Loader* ld,
                  const Json::Value& jv, Rps_Id spacid, unsigned lineno)
{
    Json::Value jsetob = jv["setob"];
    auto paylsetob = obz->put_new_plain_payload<Rps_PayloadSetOb>();

    for (int elemix = 0; elemix < (int)jsetob.size(); elemix++) {
        auto obelem = Rps_ObjectRef(jsetob[elemix], ld);
        if (obelem) paylsetob->add(obelem);
    }
}
```
**Purpose**: Restore set payloads containing unique object references
- **Deduplication**: Automatically handles set semantics
- **Null Filtering**: Ignores null object references

### Symbol Payload Loader
```cpp
void rpsldpy_symbol(Rps_ObjectZone* obz, Rps_Loader* ld,
                   const Json::Value& jv, Rps_Id spacid, unsigned lineno)
{
    const char* name = jv["symb_name"].asCString();
    bool weak = jv["symb_weak"].asBool();

    if (!Rps_PayloadSymbol::valid_name(name))
        RPS_FATALOUT("invalid symbol name");

    auto paylsymb = obz->put_new_plain_payload<Rps_PayloadSymbol>();
    paylsymb->load_register_name(std::string{name}, ld, weak);

    if (!jv["symb_val"].isNull())
        paylsymb->symbol_put_value(Rps_Value(jv["symb_val"], ld));
}
```
**Purpose**: Restore symbol objects with names and optional values
- **Name Validation**: Ensures symbol names meet RefPerSys requirements
- **Weak Symbol Support**: Handles weak symbol semantics
- **Value Association**: Restores symbol values if present

## Value Reconstruction

### JSON to Value Conversion
```cpp
Rps_Value::Rps_Value(const Json::Value& jv, Rps_Loader* ld)
{
    // Handle primitive types
    if (jv.isInt64()) {
        *this = Rps_Value(jv.asInt64(), Rps_IntTag{});
    }
    else if (jv.isDouble()) {
        *this = Rps_Value(jv.asDouble(), Rps_DoubleTag{});
    }
    else if (jv.isString()) {
        // Check for object ID strings
        if (is_object_id_string(jv.asString())) {
            *this = Rps_ObjectValue(Rps_ObjectRef(jv, ld));
        } else {
            *this = Rps_StringValue(jv.asString());
        }
    }
    // Handle complex types (sets, tuples, instances, closures)
    else if (jv.isObject() && jv.isMember("vtype")) {
        std::string str = jv["vtype"].asString();
        if (str == "set") {
            // Reconstruct set from elements
        }
        else if (str == "instance") {
            *this = Rps_InstanceZone::load_from_json(ld, jv);
        }
        else if (str == "closure") {
            // Reconstruct closure with environment
        }
    }
}
```
**Purpose**: Convert JSON representations back to RefPerSys values
- **Primitive Types**: Direct conversion for numbers and strings
- **Object References**: Resolve object IDs to references
- **Complex Types**: Reconstruct sets, instances, closures, and tuples
- **Type Dispatch**: Uses `vtype` field for complex type discrimination

## Root Object and Symbol Installation

### Root Object Installation
```cpp
void Rps_Loader::load_install_roots(void)
{
    // Install manifest-specified global roots
    for (Rps_Id curootid : ld_globrootsidset) {
        Rps_ObjectRef curootobr = find_object_by_oid(curootid);
        rps_add_root_object(curootobr);
    }

    // Install hardcoded root objects
    #define RPS_INSTALL_ROOT_OB(Oid) \
        RPS_ROOT_OB(Oid) = find_object_by_oid(Rps_Id(#Oid));
    #include "generated/rps-roots.hh"
}
```
**Purpose**: Install root objects for garbage collection protection
- **Manifest Roots**: Installs roots specified in manifest files
- **Hardcoded Roots**: Installs system-defined root objects
- **GC Protection**: Ensures root objects are never collected

### Symbol Installation
```cpp
#define RPS_INSTALL_NAMED_ROOT_OB(Oid, Name) \
    RPS_SYMB_OB(Name) = find_object_by_oid(Rps_Id(#Oid));
#include "generated/rps-names.hh"
```
**Purpose**: Install named symbol objects for system use
- **Global Symbols**: Makes symbols accessible via `RPS_SYMB_OB()` macros
- **Name Resolution**: Enables symbol lookup by name

## Native Data Configuration

### Primitive Type Configuration
```cpp
void Rps_Loader::set_primitive_type_size_and_align(Rps_ObjectRef primtypob,
                                                  unsigned sizeby, unsigned alignby)
{
    primtypob->loader_put_attr(this, rpskob_6EsfxShTuwH02waeLE, // byte_alignment
                              Rps_Value((intptr_t)sizeby));
    primtypob->loader_put_attr(this, rpskob_8IRzlYX53kN00tC3fG, // byte_size
                              Rps_Value((intptr_t)alignby));
}

void rps_set_native_data_in_loader(Rps_Loader* ld)
{
    // Configure machine-specific primitive types
    ld->set_primitive_type_size_and_align(rpskob_4V1oeUOvmxo041XLTm, // intptr_t
                                         sizeof(intptr_t), alignof(intptr_t));
    ld->set_primitive_type_size_and_align(rpskob_4nZ0jIKUbGr01OixPV, // int
                                         sizeof(int), alignof(int));
    // ... more primitive types
}
```
**Purpose**: Configure primitive types with machine-specific sizes and alignments
- **Cross-Platform**: Ensures correct sizes on different architectures
- **Runtime Configuration**: Uses `sizeof()` and `alignof()` for accuracy
- **Attribute Setting**: Stores size and alignment as object attributes

## Main Loading Entry Point

### Load from Directory
```cpp
void rps_load_from(const std::string& dirpath)
{
    Rps_Loader loader(dirpath);

    try {
        // Parse manifest and user manifest
        loader.parse_manifest_file();
        if (user_manifest_exists)
            loader.parse_user_manifest(user_manifest_path);

        // Load all spaces
        loader.load_all_state_files();

        // Initialize system state
        loader.load_install_roots();
        rps_initialize_roots_after_loading(&loader);
        rps_initialize_symbols_after_loading(&loader);
        rps_set_native_data_in_loader(&loader);

        // Process deferred initialization
        while (loader.run_some_todo_functions() > 0) {
            usleep(20); // Small delay between batches
        }

        // Report statistics
        RPS_INFORMOUT("loaded " << loader.nb_loaded_objects() << " objects");

    } catch (const std::exception& exc) {
        RPS_FATALOUT("failed to load " << dirpath << ": " << exc.what());
    }
}
```
**Purpose**: Main entry point for loading RefPerSys state from persistent storage
- **Error Handling**: Comprehensive exception catching and reporting
- **Progress Reporting**: Performance statistics and object counts
- **Deferred Processing**: Handles complex initialization dependencies
- **System Integration**: Coordinates with other system components

## Implementation Status and TODOs

### Completed Features
- ✅ **Two-Pass Loading**: Complete allocation and initialization phases
- ✅ **JSON Parsing**: Full JSON-to-object conversion with validation
- ✅ **Plugin System**: Dynamic loading with automatic building
- ✅ **Space Organization**: Multi-space persistence support
- ✅ **Payload Loaders**: Comprehensive payload type restoration
- ✅ **Value Reconstruction**: All RefPerSys value types supported
- ✅ **Root Management**: Root object and symbol installation
- ✅ **Deferred Initialization**: Todo-based dependency resolution
- ✅ **Thread Safety**: Proper synchronization for concurrent operations
- ✅ **Error Handling**: Detailed error reporting and recovery

### Incomplete Implementations
- ❌ **Instance Loading**: `Rps_InstanceZone::fill_loaded_instance_from_json()` incomplete
- ❌ **Native Data Setup**: `rps_set_native_data_in_loader()` incomplete
- ❌ **Object Reference Loading**: `Rps_ObjectRef::Rps_ObjectRef(const Json::Value&)` incomplete
- ❌ **Value Loading**: `Rps_Value::Rps_Value(const Json::Value&)` incomplete
- ❌ **JSON Zone Loading**: `Rps_JsonZone::load_from_json()` incomplete

### Known Warnings and TODOs
- **Line 1130**: Incomplete `Rps_Value::Rps_Value(const Json::Value&)` constructor
- **Line 1309**: Partly unimplemented `Rps_ObjectRef::Rps_ObjectRef(const Json::Value&)`
- **Line 1925**: Incomplete `rps_set_native_data_in_loader` function
- **Line 1950**: Warning about incomplete native data setup

### Future Enhancements
- Complete instance reconstruction with attribute validation
- Full JSON zone restoration
- Enhanced error recovery and validation
- Parallel loading for multiple spaces
- Incremental loading for large systems
- Better memory usage optimization
- Support for additional payload types
- Enhanced debugging and diagnostics

## Usage Patterns

### Basic Loading
```cpp
// Load RefPerSys state from directory
rps_load_from("/path/to/persistore");

// System is now restored with all objects, values, and relationships
```

### Manifest Structure
```json
{
  "format": "RefPerSysFormat2024A",
  "rpsmajorversion": 0,
  "rpsminorversion": 6,
  "spaceset": ["_8J6vNYtP5E800eCr5q", "_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"],
  "globalroots": ["_1aGtWm38Vw701jDhZn", "_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"],
  "plugins": ["_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"]
}
```

### Space File Structure
```
// JSON prologue with metadata
{
  "format": "RefPerSysFormat2024A",
  "spaceid": "_8J6vNYtP5E800eCr5q",
  "nbobjects": 42,
  "rpsmajorversion": 0,
  "rpsminorversion": 6
}
//+ob__XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
{
  "oid": "_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
  "class": "_1pxQkOj5Fby02nBJWN",
  "mtime": 1234567890.123,
  "comps": ["_obj1", "_obj2"],
  "attrs": [{"at": "_attr1", "va": "value1"}],
  "payload": "classinfo",
  "class_super": "_superclass",
  "class_methodict": [{"methosel": "_selector", "methclos": "_closure"}]
}
```

### Plugin Loading
```cpp
// Plugins are automatically loaded from manifest
// Source: generated/rps_pluginid-mod.cc
// Binary: plugins/rps_pluginid-mod.so
// Loaded with: dlopen(binary_path, RTLD_NOW | RTLD_GLOBAL)
```

## Design Rationale

### Two-Pass Loading Architecture
- **Memory Efficiency**: Allocates all objects before initializing relationships
- **Dependency Resolution**: Handles forward references correctly
- **Error Recovery**: Allows partial loading and better error diagnosis

### Space-Based Organization
- **Modularity**: Allows loading subsets of the object graph
- **Scalability**: Supports very large object graphs through partitioning
- **Versioning**: Each space can evolve independently

### Deferred Initialization
- **Dependency Management**: Resolves complex initialization ordering
- **Performance**: Batches work to maintain responsiveness
- **Safety**: Prevents infinite recursion in object initialization

### Plugin Architecture
- **Extensibility**: Allows runtime extension of the system
- **Build Integration**: Automatic rebuilding of outdated plugins
- **Symbol Management**: Global symbol visibility for inter-plugin communication

### JSON-Based Persistence
- **Human Readable**: JSON format allows inspection and debugging
- **Standard Format**: Widely supported with good tooling
- **Type Safety**: Explicit type information prevents corruption

This implementation provides a robust foundation for RefPerSys's persistence system, enabling the reflective system to maintain state across executions while supporting complex object graphs, dynamic plugins, and incremental loading capabilities.