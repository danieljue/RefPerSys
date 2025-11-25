# Extended Value Types Implementation Analysis (`morevalues_rps.cc`)

## Overview

The `morevalues_rps.cc` file implements extended value types in the Reflective Persistent System (RefPerSys), providing support for complex data structures beyond the basic value types defined in `values_rps.cc`. This file introduces immutable instances, JSON values as first-class objects, deque collections, and object mapping payloads, extending RefPerSys's type system to handle sophisticated data representations.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2020-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// Git versioning information
extern "C" const char rps_morevalues_gitid[];
extern "C" const char rps_morevalues_date[];
extern "C" const char rps_morevalues_shortgitid[];
extern "C" const char rps_morevalues_timestamp[];
```

### Key Dependencies
- **RefPerSys core**: Object system, value types, and memory management
- **JSON-CPP**: JSON parsing and manipulation for Rps_JsonZone
- **Standard containers**: std::deque, std::map, std::vector for complex data structures
- **Hashing utilities**: Custom hashing functions for value comparison

## Rps_InstanceZone: Immutable Instances

### Class Overview
`Rps_InstanceZone` represents immutable instances with components and attributes, providing a structured way to create complex objects with both positional components and named attributes.

### Instance Creation Methods

#### From Components Only
```cpp
Rps_InstanceZone*
Rps_InstanceZone::make_from_components(Rps_ObjectRef classob,
                                      const std::initializer_list<Rps_Value>& valil)
{
  if (!classob) return nullptr;
  auto nbsons = valil.size();
  Rps_InstanceZone* inst = Rps_QuasiZone::rps_allocate_with_wordgap<Rps_InstanceZone,
    unsigned, Rps_ObjectRef, Rps_InstanceTag>((nbsons*sizeof(Rps_Value)/sizeof(void*)),
      (unsigned)nbsons, classob, Rps_InstanceTag{});

  int ix = 0;
  Rps_Value* sonarr = inst->raw_data_sons();
  for (auto val : valil)
    sonarr[ix++] = val;

  return inst;
}
```

**Purpose**: Create instances with positional components only
- **Memory Layout**: Allocates space for class reference, component count, and component values
- **Type Safety**: Uses template metaprogramming for proper memory alignment
- **Immutability**: Once created, instances cannot be modified

#### From Components with Attributes
```cpp
Rps_InstanceZone*
Rps_InstanceZone::make_from_attributes_components(Rps_ObjectRef classob,
    const std::vector<Rps_Value>& valvec,
    const std::map<Rps_ObjectRef, Rps_Value>& attrmap)
{
  // Validate class has attribute set
  auto clpayl = classob->get_classinfo_payload();
  auto attrset = clpayl->attributes_set();

  // Allocate space for both components and attributes
  auto nbattrs = attrset->cardinal();
  auto nbcomps = valvec.size();
  auto physiz = 2*nbattrs + nbcomps;

  res = rps_allocate_with_wordgap<Rps_InstanceZone, unsigned, Rps_ObjectRef, Rps_InstanceTag>
    ((physiz*sizeof(Rps_Value))/sizeof(void*), physiz, classob, Rps_InstanceTag{});

  // Store attributes as key-value pairs
  for (auto it : attrmap) {
    Rps_ObjectRef curat = it.first;
    int ix = attrset->element_index(curat);
    sonarr[2*ix] = curat;        // Attribute key
    sonarr[2*ix+1] = it.second;  // Attribute value
  }

  // Store components
  for (int cix=0; cix<(int)nbcomps; cix++)
    sonarr[2*nbattrs+cix] = valvec[cix];

  return res;
}
```

**Purpose**: Create instances with both components and named attributes
- **Dual Structure**: Supports both positional (components) and named (attributes) data
- **Validation**: Ensures attributes are declared in the class's attribute set
- **Memory Efficiency**: Interleaves attributes and components in a single allocation

### Instance Methods

#### Class Resolution
```cpp
Rps_ObjectRef Rps_InstanceZone::compute_class(Rps_CallFrame* callerframe) const
{
  auto k_immutable_instance = RPS_ROOT_OB(_6ulDdOP2ZNr001cqVZ); // immutable_instance∈class
  RPS_LOCALFRAME(k_immutable_instance, callerframe,
                 Rps_ObjectRef obclass;);

  RPS_ASSERT(stored_type() == Rps_Type::Instance);
  _f.obclass = get_class();
  return _f.obclass;
}
```

**Purpose**: Return the class of the instance
- **Type Safety**: Validates instance type before returning class
- **Call Frame Integration**: Uses proper garbage collection context

#### Attribute Set Access
```cpp
const Rps_SetOb* Rps_InstanceZone::set_attributes(void) const
{
  Rps_ObjectRef obclass = get_class();
  auto setat = class_attrset(obclass);
  return setat;
}
```

**Purpose**: Get the set of valid attributes for this instance's class
- **Class Delegation**: Delegates to class's attribute set
- **Thread Safety**: Uses class mutex for thread-safe access

#### Output Formatting
```cpp
void Rps_InstanceZone::val_output(std::ostream& outs, unsigned depth, unsigned maxdepth) const
{
  if (depth > maxdepth) {
    outs << "...";
    return;
  }

  outs << "inst." << compute_class(nullptr);
  outs << "*";

  if (is_transient()) outs << "¡";  // U+00A1 INVERTED EXCLAMATION MARK

  conn().output(outs);

  if (depth == 0) {
    if (metarank() != 0 || metaobject()) {
      if (is_metatransient()) outs << "‼"; // U+203C DOUBLE EXCLAMATION MARK
      else outs << "#";
      outs << metarank() << ":";
      if (metaobject()) metaobject()->val_output(outs, 0, maxdepth);
      else outs << "_";
    }
  }

  outs << "{";
  int cnt = 0;
  for (auto sonv : *this) {
    if (cnt > 0) outs << ",";
    sonv.output(outs, depth+1);
    if (cnt++ % 4 == 0) outs << std::endl;
  }
  outs << "}";
}
```

**Purpose**: Provide human-readable representation of instances
- **Depth Control**: Prevents infinite recursion in circular references
- **Unicode Symbols**: Uses special characters to indicate transient/meta status
- **Structured Output**: Shows class, components, and metadata clearly

## Rps_JsonZone: JSON Values as Objects

### Class Overview
`Rps_JsonZone` treats JSON values as first-class objects in RefPerSys, enabling JSON data to be stored, manipulated, and persisted as part of the object graph.

### JSON Value Storage
```cpp
class Rps_JsonZone : public Rps_ZoneValue {
  Json::Value _jsonval;  // The underlying JSON-CPP value

public:
  static Rps_JsonZone* make(const Json::Value& jv);
  static Rps_JsonZone* load_from_json(Rps_Loader* ld, const Json::Value& jv);
};
```

### Creation and Loading
```cpp
Rps_JsonZone* Rps_JsonZone::make(const Json::Value& jv)
{
  return Rps_QuasiZone::rps_allocate<Rps_JsonZone, const Json::Value&>(jv);
}

Rps_JsonZone* Rps_JsonZone::load_from_json(Rps_Loader* ld, const Json::Value& jv)
{
  RPS_ASSERT(ld != nullptr);
  if (!jv.isObject() || !jv.isMember("json"))
    throw RPS_RUNTIME_ERROR_OUT("Rps_JsonZone::load_from_json bad jv=" << jv);
  return make(jv["json"]);
}
```

**Purpose**: Create JSON zones from JSON data or during persistence loading
- **Memory Management**: Uses RefPerSys's allocation system
- **Validation**: Ensures proper JSON structure during loading

### Hashing and Comparison
```cpp
Rps_HashInt Rps_JsonZone::compute_hash(void) const
{
  std::uint64_t h1 = 0, h2 = 0;
  rps_recursive_hash_json(_jsonval, h1, h2, 0);
  Rps_HashInt h = (h1 * 13151) ^ (h2 * 13291);
  if (RPS_UNLIKELY(h == 0))
    h = (h1 & 0xffff) + (h2 & 0xfffff) + 17;
  return h;
}

bool Rps_JsonZone::equal(const Rps_ZoneValue& zv) const
{
  if (zv.stored_type() == Rps_Type::Json) {
    auto othj = reinterpret_cast<const Rps_JsonZone*>(&zv);
    auto lh = lazy_hash();
    auto othlh = othj->lazy_hash();
    if (lh != 0 && othlh != 0 && lh != othlh) return false;
    return _jsonval == othj->_jsonval;
  }
  return false;
}
```

**Purpose**: Enable JSON values to participate in RefPerSys's value system
- **Hash Optimization**: Uses lazy hashing for performance
- **Deep Equality**: Compares JSON structure recursively
- **Type Safety**: Only compares with other JSON zones

### Recursive JSON Hashing
```cpp
static void rps_recursive_hash_json(const Json::Value& jv, std::uint64_t& h1,
                                   std::uint64_t& h2, unsigned depth)
{
  static constexpr unsigned maxrecurdepth = 32;
  if (depth > maxrecurdepth) return;

  switch (jv.type()) {
    case Json::nullValue:    h1++; h2 -= depth; return;
    case Json::intValue:     h1 += ((31*depth) ^ (jv.asInt64() % 200579));
                            h2 ^= (jv.asInt64() >> 48) - (jv.asInt64() % 300593); return;
    case Json::uintValue:    h1 += ((17*depth) ^ (jv.asUInt64() % 200569));
                            h2 ^= (jv.asUInt64() >> 42) + (jv.asUInt64() % 300557); return;
    case Json::realValue:    // Hash double values
    case Json::stringValue:  // Hash string content
    case Json::booleanValue: // Hash boolean values
    case Json::arrayValue:   // Recursively hash arrays
    case Json::objectValue:  // Recursively hash objects with sorted keys
  }
}
```

**Purpose**: Generate stable hash values for JSON structures
- **Depth Limiting**: Prevents stack overflow on deeply nested JSON
- **Type-Specific Hashing**: Different algorithms for different JSON types
- **Deterministic**: Same JSON always produces same hash
- **Collision Resistant**: Uses multiple hash functions (h1, h2)

### Serialization Support
```cpp
Json::Value Rps_JsonZone::dump_json(Rps_Dumper* du) const
{
  RPS_ASSERT(du != nullptr);
  Json::Value jv(Json::objectValue);
  jv["vtype"] = "json";
  jv["json"] = _jsonval;
  return jv;
}

void Rps_JsonZone::val_output(std::ostream& outs, unsigned depth, unsigned maxdepth) const
{
  if (depth > maxdepth) {
    outs << "?";
    return;
  }

  std::ostringstream tempouts;
  tempouts << _jsonval << std::endl;

  if (depth == 0)
    outs << tempouts.str();
  else {
    // Indent nested JSON output
    auto srcstr = tempouts.str();
    const char* pc = nullptr;
    const char* eol = nullptr;
    for (pc = srcstr.c_str(); (eol = strchr(pc, '\n')); pc = eol + 1) {
      std::string lin(pc, eol - pc);
      for (unsigned i = 0; i < depth; i++) outs << ' ';
      outs << lin;
    }
  }
}
```

**Purpose**: Enable JSON values to be persisted and displayed
- **Structured Dumping**: Wraps JSON in RefPerSys's serialization format
- **Pretty Printing**: Indents nested JSON for readability
- **Depth Control**: Prevents excessive output for deep structures

## Rps_DequVal: Deque of Values

### Class Overview
`Rps_DequVal` implements a double-ended queue of RefPerSys values, providing efficient insertion and removal from both ends while maintaining source location information for debugging.

### Construction and Initialization
```cpp
Rps_DequVal::Rps_DequVal(const std::vector<Rps_Value>& vec,
                        const char* sfil, int lin)
  : Rps_DequVal::std_deque_superclass(),
    dqu_srcfil(sfil), dqu_srclin(lin)
{
  for (const Rps_Value& curval : vec) {
    std_deque_superclass::push_back(curval);
  }
}

Rps_DequVal::Rps_DequVal(const Json::Value& jv, Rps_Loader* ld,
                        const char* sfil, int lin)
  : Rps_DequVal::std_deque_superclass(),
    dqu_srcfil(sfil), dqu_srclin(lin)
{
  Json::Value jseq = jv["dequval"];
  for (int ix = 0; ix < jseq.size(); ix++) {
    Rps_Value curval(jseq[ix], ld);
    push_back(curval);
  }
}
```

**Purpose**: Create deque values from vectors or during JSON loading
- **Source Tracking**: Maintains file and line information for debugging
- **Type Safety**: Validates JSON structure during loading
- **Memory Efficiency**: Uses std::deque for optimal performance

### Hashing and Operations
```cpp
Rps_HashInt Rps_DequVal::compute_hash(void) const
{
  Rps_HashInt h1 = 0, h2 = size();
  unsigned ix = 0;
  for (auto it : *this) {
    const Rps_Value curval = it;
    if (ix % 2 == 0)
      h1 = (h1 * 12107) + (11 * curval.valhash()) - (h2 & 0xffff);
    else
      h2 = (h2 * 22247) ^ (223 * curval.valhash() + h1);
    ix++;
  }
  Rps_HashInt h = h1 ^ h2;
  if (h == 0)
    h = ((h1 & 0xffff) | (h2 & 0xffff)) + (size() & 0xff) + 3;
  return h;
}
```

**Purpose**: Generate hash values for deque comparison and storage
- **Order Sensitivity**: Hash depends on element order
- **Value Hashing**: Incorporates hash of each contained value
- **Collision Avoidance**: Ensures non-zero hash values

### Garbage Collection and Persistence
```cpp
void Rps_DequVal::really_gc_mark(Rps_GarbageCollector& gc, unsigned depth) const
{
  RPS_ASSERT(gc.is_valid_garbcoll());
  for (const Rps_Value& curval : *this)
    curval.gc_mark(gc, depth + 1);
}

Json::Value Rps_DequVal::dump_json(Rps_Dumper* du) const
{
  Json::Value job(Json::objectValue);
  Json::Value jseq(Json::arrayValue);
  for (const Rps_Value& curval : *this)
    jseq.append(rps_dump_json_value(du, curval));
  job["dequval"] = jseq;
  return job;
}
```

**Purpose**: Support garbage collection and persistence of deque values
- **Recursive Marking**: Marks all contained values during GC
- **Structured Serialization**: Stores deque as JSON array
- **Depth Tracking**: Prevents infinite recursion in GC

### Output Formatting
```cpp
void Rps_DequVal::output(std::ostream& out, unsigned depth, unsigned maxdepth) const
{
  if (depth > 1 + Rps_Value::max_output_depth || depth > maxdepth)
    out << "°deqval(<...>)";
  else if (size() == 0) {
    if (dqu_srcfil && dqu_srclin > 0)
      out << "°deqval(@" << dqu_srcfil << ":" << dqu_srclin << "⁖⦰)"; // Empty set
    else
      out << "°deqval(⦰)"; // Empty set
  } else {
    // Format with source location and indexed elements
    out << "°deqvalℓ" << size() << "(<";
    int cnt = 0;
    for (const Rps_Value& curval : *this) {
      if (cnt > 0) out << std::endl << " ";
      out << "₍" << cnt << "₎₌"; // Subscript notation
      curval.output(out, depth + 1, maxdepth);
      cnt++;
    }
    out << ">)";
  }
}
```

**Purpose**: Provide human-readable representation of deque values
- **Unicode Symbols**: Uses mathematical notation for clarity
- **Source Information**: Shows creation location when available
- **Indexed Display**: Numbers each element for easy reference
- **Depth Control**: Prevents excessive output nesting

## Rps_PayloadObjMap: Object-to-Value Mapping

### Class Overview
`Rps_PayloadObjMap` implements payloads that map RefPerSys objects to values, providing a flexible associative array mechanism for object attributes and properties.

### Structure and Operations
```cpp
class Rps_PayloadObjMap : public Rps_Payload {
  std::map<Rps_ObjectRef, Rps_Value> obm_map;  // Object to value mapping
  Rps_Value obm_descr;                         // Optional description

public:
  void put_obmap(Rps_ObjectRef obkey, Rps_Value val);
  Rps_Value get_obmap(Rps_ObjectRef obkey, Rps_Value defaultval, bool* pmissing) const;
  bool has_key_obmap(Rps_ObjectRef obkey) const;
};
```

### Object Creation
```cpp
Rps_ObjectZone* Rps_PayloadObjMap::make(Rps_CallFrame* callframe,
                                       Rps_ObjectRef classob, Rps_ObjectRef spaceob)
{
  if (!classob)
    classob = RPS_ROOT_OB(_21O0aRqBH0p030SV0R); // objmap∈class

  if (classob == RPS_ROOT_OB(_21O0aRqBH0p030SV0R)) { // objmap∈class
    _f.mapob = Rps_ObjectRef::make_object(&_, classob, spaceob);
    auto paylobmap = _f.mapob->put_new_plain_payload<Rps_PayloadObjMap>();
    return _f.mapob;
  }
  // Handle environment class as special case
  else if (classob == RPS_ROOT_OB(_5LMLyzRp6kq04AMM8a)) { // environment∈class
    return Rps_PayloadEnvironment::make(&_, classob, spaceob);
  }
  // Handle subclasses of objmap
  else if (classob->is_subclass_of(RPS_ROOT_OB(_21O0aRqBH0p030SV0R))) {
    _f.mapob = Rps_ObjectRef::make_object(&_, classob, spaceob);
    auto paylobmap = _f.mapob->put_new_plain_payload<Rps_PayloadObjMap>();
    return _f.mapob;
  }
  return nullptr;
}
```

**Purpose**: Create objects with object-to-value mapping payloads
- **Class Validation**: Supports objmap class and its subclasses
- **Special Cases**: Handles environment class differently
- **Inheritance Support**: Works with class hierarchy

### Persistence and Serialization
```cpp
void Rps_PayloadObjMap::dump_json_objmap_internal_content(Rps_Dumper* du, Json::Value& jv) const
{
  Json::Value jmap(Json::objectValue);
  for (auto it : obm_map) {
    jmap[it.first.as_string()] = rps_dump_json_value(du, it.second);
  }
  jv["objmap"] = jmap;
  jv["descr"] = rps_dump_json_value(du, obm_descr);
}

void rpsldpy_objmap(Rps_ObjectZone* obz, Rps_Loader* ld,
                   const Json::Value& jv, Rps_Id spacid, unsigned lineno)
{
  auto paylobjmap = obz->put_new_plain_payload<Rps_PayloadObjMap>();
  const Json::Value& jobmap = jv["objmap"];
  if (jobmap.type() == Json::objectValue) {
    auto membvec = jobmap.getMemberNames();
    for (const std::string& keystr : membvec) {
      Rps_ObjectRef keyob(keystr, ld);
      Rps_Value val = Rps_Value(jobmap[keystr], ld);
      paylobjmap->put_obmap(keyob, val);
    }
  }
  paylobjmap->put_descr(Rps_Value(jv["descr"], ld));
}
```

**Purpose**: Enable object maps to be persisted and restored
- **Key Serialization**: Converts object references to strings
- **Value Preservation**: Maintains all value types in mapping
- **Description Support**: Preserves optional description value

### Output and Display
```cpp
void Rps_PayloadObjMap::output_payload(std::ostream& out, unsigned depth, unsigned maxdepth) const
{
  bool ontty = (&out == &std::cout) ? isatty(STDOUT_FILENO)
           : (&out == &std::cerr) ? isatty(STDERR_FILENO) : false;

  const char* BOLD_esc = (ontty ? RPS_TERMINAL_BOLD_ESCAPE : "");
  const char* NORM_esc = (ontty ? RPS_TERMINAL_NORMAL_ESCAPE : "");

  int nbobjmap = (int)obm_map.size();
  if (nbobjmap == 0)
    out << BOLD_esc << "-empty object map-" << NORM_esc;
  else
    out << BOLD_esc << "-object map of " << nbobjmap << ((nbobjmap > 1) ? " entries" : " entry");

  if (obm_descr)
    out << " described by " << NORM_esc << Rps_OutputValue(obm_descr, depth, maxdepth) << std::endl;
  else
    out << " plain" << NORM_esc << std::endl;

  // Sort keys for consistent display
  std::vector<Rps_ObjectRef> attrvect(nbobjmap);
  for (auto it : obm_map) attrvect.push_back(it.first);
  rps_sort_object_vector_for_display(attrvect);

  for (int ix = 0; ix < nbobjmap; ix++) {
    const Rps_ObjectRef curattr = attrvect[ix];
    const Rps_Value curval = obm_map.at(curattr);
    out << BOLD_esc << "*" << NORM_esc << curattr << ": "
        << Rps_OutputValue(curval, depth, maxdepth) << std::endl;
  }
}
```

**Purpose**: Provide formatted display of object mappings
- **Terminal Formatting**: Uses ANSI escape codes when appropriate
- **Sorted Output**: Displays keys in consistent order
- **Rich Display**: Shows descriptions and formatted values
- **Visual Clarity**: Uses symbols and formatting for readability

## Usage Patterns

### Creating Instances
```cpp
// Create instance with components only
auto inst1 = Rps_InstanceZone::make_from_components(
  myclass, {Rps_Value(42), Rps_Value("hello"), Rps_Value(true)});

// Create instance with attributes and components
std::map<Rps_ObjectRef, Rps_Value> attrs = {
  {attr_name, Rps_Value("John")},
  {attr_age, Rps_Value(30)}
};
std::vector<Rps_Value> comps = {Rps_Value("employee")};
auto inst2 = Rps_InstanceZone::make_from_attributes_components(
  employee_class, comps, attrs);
```

### Working with JSON Values
```cpp
// Create JSON value
Json::Value json_data;
json_data["name"] = "example";
json_data["count"] = 42;
auto json_obj = Rps_JsonZone::make(json_data);

// Use in RefPerSys expressions
Rps_Value json_val(json_obj);
// json_val can now be stored, passed, and manipulated like any other value
```

### Using Deque Values
```cpp
// Create deque from vector
std::vector<Rps_Value> vals = {val1, val2, val3};
Rps_DequVal deq(vals, __FILE__, __LINE__);

// Operations
deq.push_back(val4);    // Add to end
deq.pop_front();        // Remove from front
auto hash = deq.compute_hash(); // Get hash for comparison
```

### Object Mapping
```cpp
// Create object map
auto map_obj = Rps_PayloadObjMap::make(&_, nullptr, nullptr);
auto payload = map_obj->get_payload<Rps_PayloadObjMap>();

// Store mappings
payload->put_obmap(key_obj1, Rps_Value(123));
payload->put_obmap(key_obj2, Rps_Value("value"));

// Retrieve values
bool missing;
auto val1 = payload->get_obmap(key_obj1, Rps_Value(), &missing);
if (!missing) {
  // Use val1
}
```

## Design Rationale

### Immutable Instances
**Why separate from regular objects?**
- **Performance**: Immutable instances avoid locking overhead
- **Memory Efficiency**: Compact storage without object headers
- **Functional Programming**: Supports immutable data structures
- **Thread Safety**: No synchronization needed for read operations

### JSON as First-Class Values
**Why integrate JSON directly?**
- **Interoperability**: Easy exchange with external systems
- **Rich Data**: JSON supports complex nested structures
- **Standard Format**: Widely understood and supported
- **Persistence**: JSON values persist naturally

### Deque Collections
**Why deque instead of vector?**
- **Double-Ended**: Efficient operations on both ends
- **Queue Semantics**: Natural for FIFO/LIFO operations
- **Source Tracking**: Debug information for value origins
- **Flexibility**: Supports various access patterns

### Object-to-Value Mapping
**Why separate payload type?**
- **Type Safety**: Dedicated type for associative arrays
- **Performance**: Optimized for object key lookups
- **Extensibility**: Foundation for more complex mappings
- **Integration**: Works with existing payload system

### Recursive JSON Hashing
**Why custom hashing algorithm?**
- **Deterministic**: Same JSON always produces same hash
- **Collision Resistant**: Multiple hash functions reduce collisions
- **Depth Limited**: Prevents stack overflow on deep structures
- **Type Aware**: Different handling for different JSON types

This implementation significantly extends RefPerSys's value system, enabling sophisticated data structures while maintaining the system's principles of immutability, persistence, and reflective capabilities.