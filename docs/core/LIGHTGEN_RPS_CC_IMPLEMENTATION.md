# GNU Lightning JIT Code Generation Implementation Analysis (`lightgen_rps.cc`)

## Overview

The `lightgen_rps.cc` file implements **GNU Lightning** just-in-time (JIT) code generation for the Reflective Persistent System (RefPerSys), providing dynamic machine code generation capabilities. This file serves as one of RefPerSys's code generation backends, complementing the C++ code generation (`cppgen_rps.cc`) and GCC JIT (`gccjit_rps.cc`) implementations.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2023-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// GNU Lightning library (C API)
extern "C" {
#include "lightning.h"
};

// Git versioning information
extern "C" const char rps_lightgen_gitid[];
extern "C" const char rps_lightgen_date[];
extern "C" const char rps_lightgen_shortgitid[];
extern "C" const char rps_lightgen_timestamp[];
```

### Key Dependencies
- **GNU Lightning**: Dynamic code generation library (version 2.2.2+ required)
- **RefPerSys core**: Object system, payload infrastructure, and call frames
- **GCC infrastructure**: Leverages GCC's compilation pipeline

### Version Requirements
```cpp
// Requires GNU Lightning after 2.2.2 (commit 3b0fff9206a458d7e11db of August 21, 2023)
```

## Rps_PayloadLightningCodeGen Class

The `Rps_PayloadLightningCodeGen` class is the core payload for GNU Lightning JIT code generation, managing JIT state and node mappings.

### Class Structure
```cpp
class Rps_PayloadLightningCodeGen : public Rps_Payload {
public:
    typedef long lightnodenum_t;  // Node numbering type

    jit_state_t* lightg_jist;     // GNU Lightning JIT state
    std::map<jit_node*, lightnodenum_t> lightg_nod2num_map;  // Node to number mapping
    std::map<lightnodenum_t, jit_node*> lightg_num2nod_map;  // Number to node mapping

    // Convenience macro for JIT state access
    #define _jit this->lightg_jist

    // Thread safety macros
    #define RPSJITLIGHTPAYLOAD_LOCKGUARD_AT(Lin) \
      std::lock_guard<std::recursive_mutex> gu##Lin(*(this->owner()->objmtxptr()));
    #define RPSJITLIGHTPAYLOAD_LOCKGUARD() RPSJITLIGHTPAYLOAD_LOCKGUARD_AT(__LINE__)

public:
    // Core payload methods
    virtual void gc_mark(Rps_GarbageCollector& gc) const;
    virtual void dump_scan(Rps_Dumper* du) const;
    virtual void dump_json_content(Rps_Dumper* du, Json::Value& jv) const;
    virtual const std::string payload_type_name(void) const;
    virtual uint32_t wordsize(void) const;
    virtual bool is_erasable(void) const;
    virtual ~Rps_PayloadLightningCodeGen();

    // JIT lifecycle methods
    void rpsjit_prolog(void);
    void rpsjit_epilog(void);
    void rpsjit_realize(void);
    bool rpsjit_is_frozen() const;

    // Node management
    lightnodenum_t rpsjit_register_node(jit_node_t* jn);
    jit_node_t* rpsjit_node_of_num(lightnodenum_t num) const;
    lightnodenum_t rpsjit_num_of_node(jit_node_t* nd) const;

    // Object creation
    Rps_ObjectRef make_lightgen_code_object(Rps_CallFrame* callframe,
                                           Rps_ObjectRef classarg,
                                           Rps_ObjectRef spacearg);

    // Output
    virtual void output_payload(std::ostream& out, unsigned depth, unsigned maxdepth) const;
};
```

### Constructor and Initialization
```cpp
Rps_PayloadLightningCodeGen::Rps_PayloadLightningCodeGen(Rps_ObjectZone* owner)
    : Rps_Payload(Rps_Type::PaylLightCodeGen, owner),
      lightg_jist(nullptr),
      lightg_nod2num_map(),
      lightg_num2nod_map()
{
    lightg_jist = jit_new_state();
    RPSJITLIGHTPAYLOAD_LOCKGUARD();
}
```
**Purpose**: Initialize payload with new GNU Lightning JIT state
- **JIT State Creation**: `jit_new_state()` allocates fresh JIT context
- **Thread Safety**: Constructor includes locking for thread safety
- **Mapping Initialization**: Empty bidirectional node mappings

### Destructor and Cleanup
```cpp
Rps_PayloadLightningCodeGen::~Rps_PayloadLightningCodeGen()
{
    RPSJITLIGHTPAYLOAD_LOCKGUARD();
    // Validate mappings before cleanup
    for (auto itnod : lightg_nod2num_map) {
        jit_node_t* curnod = itnod.first;
        lightnodenum_t curnum = itnod.second;
        RPS_ASSERT(curnod != nullptr);
        RPS_ASSERT(curnum > 0);
        RPS_ASSERT(lightg_num2nod_map.find(curnum) != lightg_num2nod_map.end());
    }
    lightg_nod2num_map.clear();
    lightg_num2nod_map.clear();
    _jit_destroy_state(lightg_jist);  // Destroys all nodes
    lightg_jist = nullptr;
}
```
**Purpose**: Clean up JIT state and validate consistency
- **Mapping Validation**: Ensures bidirectional mapping consistency
- **Resource Cleanup**: Clears all mappings and destroys JIT state
- **Node Destruction**: `_jit_destroy_state()` frees all JIT nodes

## GNU Lightning Integration

### JIT State Management
The payload uses GNU Lightning's `jit_state_t` as the central JIT context, with convenience macros for access:

```cpp
#define _jit this->lightg_jist  // Direct access to JIT state
```

### JIT Lifecycle Methods
```cpp
void rpsjit_prolog(void) {
    RPSJITLIGHTPAYLOAD_LOCKGUARD();
    jit_prolog();
}

void rpsjit_epilog(void) {
    RPSJITLIGHTPAYLOAD_LOCKGUARD();
    jit_epilog();
}

void rpsjit_realize(void) {
    RPSJITLIGHTPAYLOAD_LOCKGUARD();
    jit_realize();
}

bool rpsjit_is_frozen() const {
    RPSJITLIGHTPAYLOAD_LOCKGUARD();
    return lightg_jist == nullptr;
}
```
**Purpose**: Wrapper methods for GNU Lightning JIT lifecycle
- **Prolog/Epilog**: Function entry/exit code generation
- **Realize**: Finalize code generation and prepare for execution
- **Frozen Check**: Detect when JIT state has been destroyed

## Node Numbering and Mapping System

### Node Registration
```cpp
lightnodenum_t rpsjit_register_node(jit_node_t* jn) {
    lightnodenum_t num = 0;
    RPSJITLIGHTPAYLOAD_LOCKGUARD();
    RPS_ASSERT(jn != nullptr);

    size_t nbnod = lightg_nod2num_map.size();
    RPS_ASSERT(nbnod == lightg_num2nod_map.size());

    auto numit = lightg_nod2num_map.find(jn);
    if (numit != lightg_nod2num_map.end()) {
        num = numit->second;
        RPS_ASSERT(num > 0);
        RPS_ASSERT(lightg_num2nod_map.find(num) != lightg_num2nod_map.end());
        return num;
    }

    num = (lightnodenum_t)(nbnod + 1);
    lightg_nod2num_map.insert({jn, num});
    lightg_num2nod_map.insert({num, jn});
    return num;
}
```
**Purpose**: Assign persistent numbers to JIT nodes for serialization
- **Deduplication**: Returns existing number if node already registered
- **Bidirectional Mapping**: Maintains both node→number and number→node mappings
- **Consistency Checks**: Validates mapping integrity

### Node Lookup Methods
```cpp
jit_node_t* rpsjit_node_of_num(lightnodenum_t num) const {
    if (num == 0) return nullptr;
    jit_node_t* nd = nullptr;
    RPSJITLIGHTPAYLOAD_LOCKGUARD();
    auto numit = lightg_num2nod_map.find(num);
    if (numit != lightg_num2nod_map.end()) {
        nd = numit->second;
        RPS_ASSERT(lightg_nod2num_map.find(nd) != lightg_nod2nod_map.end());
    }
    return nd;
}

lightnodenum_t rpsjit_num_of_node(jit_node_t* nd) const {
    if (nd == nullptr) return 0;
    RPSJITLIGHTPAYLOAD_LOCKGUARD();
    lightnodenum_t num = 0;
    auto nodit = lightg_nod2num_map.find(nd);
    if (nodit != lightg_nod2num_map.end()) {
        num = nodit->second;
        RPS_ASSERT(lightg_num2nod_map.find(num) != lightg_num2nod_map.end());
    }
    return num;
}
```
**Purpose**: Bidirectional lookup between nodes and their persistent numbers
- **Null Handling**: Returns 0/null for invalid inputs
- **Thread Safety**: Protected by payload mutex
- **Consistency Validation**: Cross-checks mappings

## Code Generation Function

### Main Code Generation Entry Point
```cpp
bool rps_generate_lightning_code(Rps_CallFrame* callerframe,
                                Rps_ObjectRef argobmodule,
                                Rps_Value arggenparam)
```
**Purpose**: Generate JIT code from a RefPerSys module using GNU Lightning
- **Module Processing**: Iterates through module components
- **Generator Creation**: Creates `midend_lightning_code_generator` object
- **Component Handling**: Applies closures or sends messages to components
- **Result Collection**: Gathers results from component processing

### Module Processing Algorithm
1. **Generator Creation**: Create generator object with Lightning payload
2. **Attribute Setup**: Set code_module and generate_code attributes
3. **Component Iteration**: Process each component of the module
4. **Closure/Message Dispatch**: Apply closures or send `lightning_generate_code` messages
5. **Result Collection**: Store results in vector for GC safety

### Component Processing
```cpp
// For closure components
if (_f.elemv.is_closure()) {
    Rps_TwoValues apres = Rps_ClosureValue(_f.elemv).apply4(&_,
        _f.obgenerator, _f.genparamv, _f.obmodule,
        Rps_Value::make_tagged_int(mix));
    _f.mainv = apres.mainv();
    _f.xtrav = apres.xtrav();
}

// For non-closure components
else {
    Rps_TwoValues snres = _f.elemv.send4(&_,
        RPS_ROOT_OB(_6GiKCsHJDCi04m74XV), // lightning_generate_code
        _f.obgenerator, _f.genparamv, _f.obmodule,
        Rps_Value::make_tagged_int(mix));
    _f.mainv = snres.mainv();
    _f.xtrav = snres.xtrav();
}
```
**Purpose**: Process individual module components
- **Closure Application**: Direct function call with generator context
- **Message Sending**: Dynamic dispatch to `lightning_generate_code` selector
- **Result Handling**: Capture main and extra return values

## Object Creation and Management

### Code Object Creation
```cpp
Rps_ObjectRef make_lightgen_code_object(Rps_CallFrame* callframe,
                                       Rps_ObjectRef obclassarg,
                                       Rps_ObjectRef obspacearg)
```
**Purpose**: Create new objects with Lightning code generation payloads
- **Class Validation**: Ensures class is subclass of `$lightning_code_object`
- **Default Class**: Uses `$lightning_code_object` if none specified
- **Payload Attachment**: Creates and attaches Lightning payload
- **Space Assignment**: Places object in specified space

### Generator Object Creation
```cpp
_f.obgenerator = Rps_ObjectRef::make_object(&_,
    RPS_ROOT_OB(_6SM7PykipQW01HVClH) // midend_lightning_code_generator
);
Rps_PayloadLightningCodeGen* paylgen =
    _f.obgenerator->put_new_plain_payload<Rps_PayloadLightningCodeGen>();
```
**Purpose**: Create generator objects for code generation process
- **Class Instantiation**: Creates `midend_lightning_code_generator` instance
- **Payload Setup**: Attaches Lightning code generation payload
- **Attribute Configuration**: Sets module and parameter attributes

## Persistence Integration

### Garbage Collection
```cpp
void gc_mark(Rps_GarbageCollector& gc) const {
    // Currently empty - no RefPerSys objects directly referenced
}
```
**Purpose**: Mark objects referenced by the payload during GC
- **Status**: Currently no-op (no direct object references)
- **Future**: May need to mark referenced objects

### Dumping Support
```cpp
void dump_scan(Rps_Dumper* du) const {
    RPS_ASSERT(du);
    RPS_POSSIBLE_BREAKPOINT();
    // Currently incomplete
}

void dump_json_content(Rps_Dumper* du, Json::Value& jv) const {
    RPS_ASSERT(du);
    RPS_POSSIBLE_BREAKPOINT();
    #warning incomplete Rps_PayloadLightningCodeGen::dump_json_content
    RPS_WARNOUT("incomplete Rps_PayloadLightningCodeGen::dump_json_content owner="
                << RPS_OBJECT_DISPLAY(owner()));
}
```
**Purpose**: Support persistence of Lightning code generation state
- **Status**: Implementation incomplete
- **Requirements**: Serialize node mappings and JIT state

### Loading Support
```cpp
void rpsldpy_lightning_code_generator(Rps_ObjectZone* obz, Rps_Loader* ld,
                                    const Json::Value& jv, Rps_Id spacid,
                                    unsigned lineno) {
    // Create payload
    auto payl = obz->put_new_plain_payload<Rps_PayloadLightningCodeGen>();
    #warning unimplemented rpsldpy_lightning_code_generator
    RPS_WARNOUT("unimplemented rpsldpy_lightning_code_generator jv=" << jv
                << " spacid=" << spacid << " lineno=" << lineno
                << " obz=" << RPS_OBJECT_DISPLAY(obz));
}
```
**Purpose**: Restore Lightning code generation state from persistence
- **Status**: Implementation incomplete
- **Requirements**: Deserialize node mappings and reconstruct JIT state

## Thread Safety Architecture

### Synchronization Patterns
- **Owner Mutex**: All operations lock the payload owner's object mutex
- **Recursive Mutex**: Allows nested locking within the same thread
- **Lock Macros**: Convenience macros for consistent locking
- **GC Safety**: Additional mutex for result vector during garbage collection

### Lock Hierarchy
```cpp
#define RPSJITLIGHTPAYLOAD_LOCKGUARD_AT(Lin) \
  std::lock_guard<std::recursive_mutex> gu##Lin(*(this->owner()->objmtxptr()));
```
- **Owner Lock**: Protects payload state and JIT operations
- **Module Lock**: Protects module being processed
- **Generator Lock**: Protects generator object
- **Result Lock**: Protects result collection vector

## Output and Debugging

### Payload Display
```cpp
void output_payload(std::ostream& out, unsigned depth, unsigned maxdepth) const {
    bool ontty = (&out == &std::cout) ? isatty(STDOUT_FILENO)
              : (&out == &std::cerr) ? isatty(STDERR_FILENO) : false;
    if (rps_without_terminal_escape) ontty = false;

    const char* BOLD_esc = (ontty ? RPS_TERMINAL_BOLD_ESCAPE : "");
    const char* NORM_esc = (ontty ? RPS_TERMINAL_NORMAL_ESCAPE : "");

    std::lock_guard<std::recursive_mutex> guown(*(owner()->objmtxptr()));
    out << BOLD_esc << "* GNU lightning code generator for "
        << lightg_num2nod_map.size() << " nodes*"
        << NORM_esc << std::endl;
}
```
**Purpose**: Display payload information with terminal formatting
- **Terminal Detection**: Adapts output based on terminal capabilities
- **Node Count**: Shows number of registered JIT nodes
- **Formatting**: Uses bold/normal escape sequences

## Implementation Status and TODOs

### Completed Features
- ✅ **Payload Infrastructure**: Complete class structure with proper inheritance
- ✅ **JIT State Management**: GNU Lightning integration with lifecycle methods
- ✅ **Node Mapping System**: Bidirectional node numbering for persistence
- ✅ **Thread Safety**: Comprehensive mutex usage and lock ordering
- ✅ **Object Creation**: Code object and generator creation methods
- ✅ **Module Processing**: Framework for processing module components
- ✅ **Basic GC Integration**: Payload participates in garbage collection
- ✅ **Output Support**: Terminal-aware payload display

### Incomplete Implementations
- ❌ **Code Generation**: Main `rps_generate_lightning_code()` function incomplete
- ❌ **Persistence**: Dump/scan and JSON content methods incomplete
- ❌ **Loading**: `rpsldpy_lightning_code_generator()` unimplemented
- ❌ **JIT Operations**: No actual code generation (prolog/epilog/etc. wrappers only)
- ❌ **Node Processing**: No processing of individual JIT nodes
- ❌ **Result Handling**: Incomplete result collection and processing
- ❌ **Error Handling**: Limited error recovery and diagnostics

### Known Warnings and TODOs
- **Line 233**: Incomplete `dump_json_content` method
- **Line 246**: Unimplemented `rpsldpy_lightning_code_generator`
- **Line 313**: TODO about better attribute for generator
- **Line 391**: Incomplete `rps_generate_lightning_code` function
- **Line 398**: Warning about incomplete code generation
- **Line 432**: Probably incomplete `make_lightgen_code_object`
- **Line 437**: Incomplete entire file

### Future Enhancements
- Complete code generation pipeline
- Implement full JIT node processing
- Add persistence support for node mappings
- Enhance error handling and diagnostics
- Support for different target architectures
- Integration with RefPerSys debugging facilities
- Optimization and code analysis features
- Support for function calls and data references

## Usage Patterns

### Basic Payload Creation
```cpp
// Create a Lightning code generator payload
auto payload = object->put_new_plain_payload<Rps_PayloadLightningCodeGen>();
```

### JIT Lifecycle
```cpp
// Initialize function generation
payload->rpsjit_prolog();

// ... generate code ...

// Finalize function
payload->rpsjit_epilog();

// Prepare for execution
payload->rpsjit_realize();
```

### Node Registration
```cpp
// Register a JIT node and get its persistent number
lightnodenum_t num = payload->rpsjit_register_node(jit_node);

// Later retrieve the node
jit_node_t* node = payload->rpsjit_node_of_num(num);
```

### Code Generation
```cpp
// Generate code from a module (currently incomplete)
bool success = rps_generate_lightning_code(callframe, module_object, params);
```

### Object Creation
```cpp
// Create a new Lightning code object
Rps_ObjectRef code_obj = payload->make_lightgen_code_object(
    callframe, target_class, target_space);
```

## Design Rationale

### GNU Lightning Choice
- **Lightweight**: Minimal runtime dependencies
- **Portable**: Supports multiple architectures
- **Dynamic**: True JIT compilation capabilities
- **C Integration**: Easy integration with existing C code

### Payload-Based Architecture
- **Object Integration**: JIT state tied to RefPerSys objects
- **Persistence**: Framework for state serialization
- **Modularity**: Each generator manages its own JIT context

### Node Numbering System
- **Persistence**: Enables serialization of JIT state
- **Debugging**: Provides stable identifiers for nodes
- **Memory Efficiency**: Avoids storing large node pointers

### Thread Safety Design
- **Owner Locking**: Protects payload state and operations
- **Hierarchical**: Consistent lock ordering prevents deadlocks
- **GC Safety**: Additional synchronization for garbage collection

### Incomplete Implementation
- **Framework First**: Provides infrastructure for future development
- **Incremental**: Allows gradual implementation of features
- **Research**: Serves as foundation for JIT compilation research

This implementation provides a solid foundation for JIT code generation in RefPerSys using GNU Lightning, with clear pathways for completing the code generation pipeline and enabling dynamic compilation capabilities.