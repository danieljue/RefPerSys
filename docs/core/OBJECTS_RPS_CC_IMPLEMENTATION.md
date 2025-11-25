# Objects Implementation Analysis (`objects_rps.cc`)

## Overview

The `objects_rps.cc` file implements the core object system of the Reflective Persistent System (RefPerSys), providing the foundational infrastructure for objects, references, payloads, and the meta-object protocol. This file establishes the homoiconic principle where code and data are unified through a sophisticated object model that supports reflection, persistence, and dynamic behavior modification.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2019-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// Git versioning information
extern "C" const char rps_objects_gitid[];
extern "C" const char rps_objects_date[];
extern "C" const char rps_objects_shortgitid[];
extern "C" const char rps_objects_timestamp[];
```

### Global Object Registry
```cpp
// Global object storage and lookup
std::unordered_map<Rps_Id, Rps_ObjectZone*, Rps_Id::Hasher> Rps_ObjectZone::ob_idmap_(50777);
std::map<Rps_Id, Rps_ObjectZone*> Rps_ObjectZone::ob_idbucketmap_[Rps_Id::maxbuckets];
std::recursive_mutex Rps_ObjectZone::ob_idmtx_;
```

**Purpose**: Thread-safe global registry for all objects in the system, supporting fast lookup by object ID and bucket-based organization for scalability.

## Rps_ObjectZone: Core Object Implementation

### Class Overview
`Rps_ObjectZone` represents the fundamental object entity in RefPerSys, implementing the memory layout and behavior of all objects in the system.

### Object Structure
```cpp
class Rps_ObjectZone : public Rps_ZoneValue {
  Rps_Id ob_oid;                    // Unique object identifier
  std::recursive_mutex ob_mtx;      // Thread synchronization
  std::atomic<Rps_ObjectRef> ob_class;    // Object's class
  std::atomic<Rps_ObjectRef> ob_space;    // Containing space
  std::atomic<double> ob_mtime;     // Modification timestamp

  std::map<Rps_ObjectRef, Rps_Value> ob_attrs;  // Named attributes
  std::vector<Rps_Value> ob_comps;              // Positional components
  std::atomic<Rps_Payload*> ob_payload;         // Dynamic payload

  // Function pointers for dynamic behavior
  std::atomic<rps_magicgetterfun_t*> ob_magicgetterfun;
  std::atomic<rps_applyingfun_t*> ob_applyingfun;
};
```

### Object Creation and Registration
```cpp
Rps_ObjectZone* Rps_ObjectZone::make(void)
{
  Rps_Id oid = fresh_random_oid(nullptr);
  Rps_ObjectZone* obz = Rps_QuasiZone::rps_allocate<Rps_ObjectZone, Rps_Id, registermode_en>
    (oid, OBZ_REGISTER);

  *(const_cast<Rps_Id*>(&obz->ob_oid)) = oid;
  double rtime = rps_wallclock_real_time();
  obz->ob_mtime.store(rtime);
  obz->ob_class.store(RPS_ROOT_OB(_5yhJGgxLwLp00X0xEQ)); // object∈class

  return obz;
}
```

**Purpose**: Create new objects with unique IDs, proper initialization, and registration in the global object map.

### Object Lookup and Management
```cpp
Rps_ObjectZone* Rps_ObjectZone::find(Rps_Id oid)
{
  if (!oid.valid()) return nullptr;
  std::lock_guard<std::recursive_mutex> gu(ob_idmtx_);
  auto obr = ob_idmap_.find(oid);
  if (obr != ob_idmap_.end()) return obr->second;
  return nullptr;
}

void Rps_ObjectZone::register_objzone(Rps_ObjectZone* obz)
{
  RPS_ASSERT(obz != nullptr);
  std::lock_guard<std::recursive_mutex> gu(ob_idmtx_);
  auto oid = obz->oid();
  if (ob_idmap_.find(oid) != ob_idmap_.end())
    RPS_FATALOUT("Rps_ObjectZone::register_objzone duplicate oid " << oid);
  ob_idmap_.insert({oid, obz});
  ob_idbucketmap_[oid.bucket_num()].insert({oid, obz});
}
```

**Purpose**: Provide fast, thread-safe object lookup and ensure uniqueness of object IDs.

### Garbage Collection Integration
```cpp
void Rps_ObjectZone::gc_mark(Rps_GarbageCollector& gc, unsigned) const
{
  if (is_gcmarked(gc)) return;
  std::lock_guard<std::recursive_mutex> gu(ob_mtx);
  gc.mark_obj(this);
  const_cast<Rps_ObjectZone*>(this)->set_gcmark(gc);
}

void Rps_ObjectZone::mark_gc_inside(Rps_GarbageCollector& gc)
{
  std::lock_guard<std::recursive_mutex> gu(ob_mtx);
  Rps_ObjectZone* obcla = ob_class.load();
  gc.mark_obj(obcla);

  for (auto atit : ob_attrs) {
    gc.mark_obj(atit.first);
    if (atit.second.is_ptr()) gc.mark_value(atit.second);
  }

  for (auto compv : ob_comps) {
    if (compv.is_ptr()) gc.mark_value(compv);
  }

  Rps_Payload* payl = ob_payload.load();
  if (payl && payl->owner() == this) payl->gc_mark(gc);
}
```

**Purpose**: Ensure all object references and contained values are properly marked during garbage collection.

## Rps_ObjectRef: Smart Object References

### Class Overview
`Rps_ObjectRef` provides smart pointer semantics for object references, implementing automatic memory management and thread-safe access patterns.

### Reference Construction and Conversion
```cpp
Rps_ObjectRef::Rps_ObjectRef(Rps_CallFrame* callerframe, const char* oidstr, Rps_ObjIdStrTag)
{
  if (!oidstr) throw RPS_RUNTIME_ERROR_OUT("Rps_ObjectRef: null oidstr");
  const char* end = nullptr;
  bool ok = false;
  Rps_Id oid(oidstr, &end, &ok);
  if (!end || *end) throw RPS_RUNTIME_ERROR_OUT("Rps_ObjectRef: invalid oidstr=" << oidstr);
  *this = find_object_or_fail_by_oid(callerframe, oid);
}
```

**Purpose**: Create object references from compile-time string literals with validation.

### Display and Comparison
```cpp
int Rps_ObjectRef::compare_for_display(const Rps_ObjectRef leftob, const Rps_ObjectRef rightob)
{
  if (leftob.optr() == rightob.optr()) return 0;
  if (leftob.is_empty()) return rightob.is_empty() ? 0 : -1;
  if (rightob.is_empty()) return leftob.is_empty() ? 0 : 1;

  // Compare by name first, then by OID
  std::string sleftname, srightname;
  // Extract names from objects...
  if (!sleftname.empty() && !srightname.empty()) {
    if (sleftname == srightname) {
      if (leftid < rightid) return -1; else return 1;
    } else {
      if (sleftname < srightname) return -1; else return 1;
    }
  }
  // Fallback to OID comparison
  if (leftid < rightid) return -1; else return 1;
}
```

**Purpose**: Provide human-friendly object comparison for display and sorting purposes.

### String Representation
```cpp
const std::string Rps_ObjectRef::as_string(void) const
{
  if (_optr == nullptr) return std::string{"__"};
  if (_optr == RPS_EMPTYSLOT) return std::string{"_⁂_"};

  const Rps_Id curoid = _optr->oid();
  std::lock_guard<std::recursive_mutex> gu(*_optr->objmtxptr());

  // Check for symbol payload
  if (const Rps_PayloadSymbol* symbpayl = _optr->get_dynamic_payload<Rps_PayloadSymbol>()) {
    const std::string& syna = symbpayl->symbol_name();
    if (!syna.empty()) return std::string{"¤"} + syna;  // U+00A4 CURRENCY SIGN
  }

  // Check for class payload
  else if (const Rps_PayloadClassInfo* classpayl = _optr->get_dynamic_payload<Rps_PayloadClassInfo>()) {
    const std::string clana = classpayl->class_name_str();
    if (!clana.empty()) return std::string{"∋"} + clana;  // U+220B CONTAINS AS MEMBER
  }

  // Check for named objects
  else if (const Rps_Value namval = _optr->get_physical_attr(RPS_ROOT_OB(_1EBVGSfW2m200z18rx))) {
    if (namval.is_string()) {
      std::string str{"⁑"};  // U+2051 TWO ASTERISKS ALIGNED VERTICALLY
      str.append(namval.as_cppstring());
      str.append(":");
      str.append(curoid.to_string());
      return str;
    }
  }

  return curoid.to_string();
}
```

**Purpose**: Generate human-readable string representations using Unicode symbols to distinguish different object types.

### Output Formatting
```cpp
void Rps_ObjectRef::output(std::ostream& outs, unsigned depth, unsigned maxdepth) const
{
  if (is_empty()) {
    outs << "__";
    return;
  }
  if (depth > maxdepth) {
    outs << "??";
    return;
  }

  std::lock_guard<std::recursive_mutex> gu(*_optr->objmtxptr());
  Rps_Value valname = obptr()->get_physical_attr(RPS_ROOT_OB(_1EBVGSfW2m200z18rx));

  outs << "◌" << obptr()->oid().to_string();  // U+25CC DOTTED CIRCLE

  if (valname.is_string()) {
    outs << "/" << valname.as_cstring();
  }

  // Show class information at appropriate depth
  if (depth <= 2) {
    Rps_ObjectRef obclass = obptr()->get_class();
    if (obclass) {
      std::lock_guard<std::recursive_mutex> gucl(*obclass->objmtxptr());
      auto obclpayl = obclass->get_dynamic_payload<Rps_PayloadClassInfo>();
      if (obclpayl) {
        outs << "∊" << obclpayl->class_name_str();  // U+2208 ELEMENT OF
      }
    }
  }
}
```

**Purpose**: Provide structured, depth-controlled output with class information and Unicode visual cues.

## Payload Classes: Dynamic Object Behavior

### Rps_PayloadClassInfo: Class Metadata
```cpp
class Rps_PayloadClassInfo : public Rps_Payload {
  Rps_ObjectRef pclass_super;           // Superclass reference
  Rps_ObjectRef pclass_symbname;        // Symbol name for the class
  std::map<Rps_ObjectRef, Rps_Value> pclass_methdict;  // Method dictionary
  std::atomic<Rps_SetOb*> pclass_attrset;  // Attribute set
};
```

**Purpose**: Store class metadata including inheritance, methods, and attributes.

#### Method Management
```cpp
Rps_SetValue Rps_PayloadClassInfo::compute_set_of_own_method_selectors(Rps_CallFrame* callerframe) const
{
  std::set<Rps_ObjectRef> mutset;
  std::mutex mutlock;

  auto obrown = owner();
  std::lock_guard<std::recursive_mutex> guown(*(obrown->objmtxptr()));

  for (auto& it : pclass_methdict) {
    _f.obcursel = it.first;
    _f.curclos = it.second;
    if (!_f.obcursel || !_f.curclos) continue;
    mutset.insert(_f.obcursel);
  }

  _f.setsel = Rps_SetValue(mutset);
  return _f.setsel;
}
```

**Purpose**: Compute the set of method selectors defined directly on this class.

### Rps_PayloadSetOb: Object Sets
```cpp
class Rps_PayloadSetOb : public Rps_Payload {
  std::set<Rps_ObjectRef> psetob;  // Set of object references
};
```

**Purpose**: Implement mutable sets of objects with thread-safe operations.

#### Set Operations
```cpp
Rps_ObjectRef Rps_PayloadSetOb::make_mutable_set_object(Rps_CallFrame* callerframe,
  Rps_ObjectRef classobarg, Rps_ObjectRef spaceobarg)
{
  // Validate class and space
  if (!classobarg) classobarg = RPS_ROOT_OB(_0J1C39JoZiv03qA2HA); // mutable_set∈class
  if (spaceobarg && !spaceobarg->is_subclass_of(RPS_ROOT_OB(_2i66FFjmS7n03HNNBx)))
    throw std::runtime_error("invalid space object");

  // Create object and add to global set
  _f.resob = Rps_ObjectRef::make_object(&_, classobarg, spaceobarg);
  _f.resob->put_new_plain_payload<Rps_PayloadSetOb>();

  // Register in global mutable set collection
  std::unique_lock<std::recursive_mutex> gumutsetcla(*(_f.obthemutsetclasses->objmtxptr()));
  auto paylsetcla = _f.obthemutsetclasses->get_dynamic_payload<Rps_PayloadSetOb>();
  paylsetcla->add(_f.resob);

  return _f.resob;
}
```

**Purpose**: Create mutable set objects with proper class validation and global registration.

### Rps_PayloadVectOb: Object Vectors
```cpp
class Rps_PayloadVectOb : public Rps_Payload {
  std::vector<Rps_ObjectRef> pvectob;  // Vector of object references
};
```

**Purpose**: Implement mutable vectors of objects with efficient indexed access.

### Rps_PayloadVectVal: Value Vectors
```cpp
class Rps_PayloadVectVal : public Rps_Payload {
  std::vector<Rps_Value> pvectval;  // Vector of values
};
```

**Purpose**: Store sequences of arbitrary values with garbage collection support.

#### Closure Creation
```cpp
const Rps_ClosureZone* Rps_PayloadVectVal::make_closure_zone_from_vector(Rps_ObjectRef connob)
{
  if (!connob) return nullptr;
  std::lock_guard<std::recursive_mutex> gu(*(connob->objmtxptr()));
  return Rps_ClosureZone::make(connob, pvectval);
}
```

**Purpose**: Convert value vectors into executable closures.

### Rps_PayloadSpace: Space Objects
```cpp
class Rps_PayloadSpace : public Rps_Payload {
  // Minimal payload for space objects
};
```

**Purpose**: Mark objects as spaces in the object hierarchy.

### Rps_PayloadSymbol: Symbol Objects
```cpp
class Rps_PayloadSymbol : public Rps_Payload {
  std::string symb_name;                    // Symbol name
  std::atomic<Rps_Value> symb_data;         // Symbol value
  std::atomic<bool> symb_is_weak;           // Weak reference flag

  // Global symbol registry
  static std::recursive_mutex symb_tablemtx;
  static std::map<std::string, Rps_PayloadSymbol*> symb_table;
  static std::unordered_map<std::string, Rps_ObjectRef*> symb_hardcoded_hashtable;
};
```

**Purpose**: Implement symbols with global name registry and value storage.

#### Symbol Management
```cpp
bool Rps_PayloadSymbol::register_name(std::string name, Rps_ObjectRef obj, bool weak)
{
  if (!obj || !valid_name(name)) return false;

  std::lock_guard<std::recursive_mutex> gu(*(obj->objmtxptr()));
  if (obj->get_payload() && !obj->has_erasable_payload()) return false;

  std::lock_guard<std::recursive_mutex> gusy(symb_tablemtx);
  if (symb_table.find(name) != symb_table.end()) return false;

  Rps_PayloadSymbol* paylsymb = obj->put_new_plain_payload<Rps_PayloadSymbol>();
  paylsymb->symb_name = name;
  symb_table.insert({paylsymb->symb_name, paylsymb});
  paylsymb->symb_is_weak.store(weak);

  // Update hardcoded references
  auto symbit = symb_hardcoded_hashtable.find(name);
  if (symbit != symb_hardcoded_hashtable.end()) {
    *(symbit->second) = obj;
  }

  return true;
}
```

**Purpose**: Register symbols in the global symbol table with weak/strong reference semantics.

## Attribute and Component Management

### Attribute Operations
```cpp
void Rps_ObjectZone::put_attr(const Rps_ObjectRef obattr, const Rps_Value valattr)
{
  RPS_ASSERT(stored_type() == Rps_Type::Object);
  if (obattr.is_empty() || obattr->stored_type() != Rps_Type::Object) return;

  // Check for magic attributes
  rps_magicgetterfun_t* getfun = obattr->ob_magicgetterfun.load();
  if (getfun) throw RPS_RUNTIME_ERROR_OUT("cannot put magic attribute " << obattr);

  std::lock_guard gu(ob_mtx);
  if (valattr.is_empty())
    ob_attrs.erase(obattr);
  else
    ob_attrs.insert_or_assign(obattr, valattr);

  ob_mtime.store(rps_wallclock_real_time());
}
```

**Purpose**: Set object attributes with validation and timestamp updates.

### Component Operations
```cpp
void Rps_ObjectZone::append_comp1(Rps_Value comp0)
{
  RPS_ASSERT(stored_type() == Rps_Type::Object);
  if (RPS_UNLIKELY(comp0.is_empty())) comp0.clear();

  std::lock_guard gu(ob_mtx);
  ob_comps.push_back(comp0);
}
```

**Purpose**: Add components to objects with empty value normalization.

### Batch Operations
```cpp
void Rps_ObjectZone::put_attr2(const Rps_ObjectRef obattr0, const Rps_Value valattr0,
                              const Rps_ObjectRef obattr1, const Rps_Value valattr1)
{
  // Validate both attributes
  // Set both attributes atomically
  // Update modification time once
}
```

**Purpose**: Perform multiple attribute operations efficiently in a single lock acquisition.

## Method Installation and Dispatch

### Method Installation
```cpp
void Rps_ObjectRef::install_own_method(Rps_CallFrame* callerframe,
                                      Rps_ObjectRef obselarg, Rps_Value closvarg)
{
  _f.obclass = *this;
  _f.obsel = obselarg;
  _f.closv = closvarg;

  // Validation
  if (_f.obclass.is_empty()) throw RPS_RUNTIME_ERROR_OUT("empty class");
  if (_f.obsel.is_empty()) throw RPS_RUNTIME_ERROR_OUT("empty selector");
  if (_f.closv.is_empty() || !_f.closv.is_closure())
    throw RPS_RUNTIME_ERROR_OUT("invalid closure");

  // Install method
  auto paylcl = _f.obclass->get_classinfo_payload();
  paylcl->put_own_method(_f.obsel, _f.closv);
}
```

**Purpose**: Install methods on classes with proper validation and error handling.

### Batch Method Installation
```cpp
void Rps_ObjectRef::install_own_2_methods(Rps_CallFrame* callerframe,
    Rps_ObjectRef obsel0arg, Rps_Value closv0arg,
    Rps_ObjectRef obsel1arg, Rps_Value closv1arg)
{
  // Validate both selectors and closures
  // Install both methods atomically
}
```

**Purpose**: Install multiple methods efficiently with shared validation.

## Object Creation Patterns

### Named Class Creation
```cpp
Rps_ObjectRef Rps_ObjectRef::make_named_class(Rps_CallFrame* callerframe,
                                             Rps_ObjectRef superclassarg, std::string name)
{
  // Validate superclass
  if (!superclassarg) throw std::runtime_error("no superclass");
  if (superclassarg->get_class() != RPS_ROOT_OB(_41OFI3r0S1t03qdB2E))
    throw std::runtime_error("invalid superclass");

  // Validate name
  if (!Rps_PayloadSymbol::valid_name(name))
    throw std::runtime_error("invalid class name");

  // Create or find symbol
  _f.obsymbol = Rps_PayloadSymbol::find_named_object(name);
  if (!_f.obsymbol) _f.obsymbol = make_new_strong_symbol(&_, name);

  // Create class object
  _f.obclass = Rps_ObjectZone::make();
  _f.obclass->ob_class.store(RPS_ROOT_OB(_41OFI3r0S1t03qdB2E)); // class∈class

  // Set up class info payload
  auto paylclainf = _f.obclass->put_new_plain_payload<Rps_PayloadClassInfo>();
  paylclainf->put_superclass(superclassarg);
  paylclainf->put_symbname(_f.obsymbol);

  // Register symbol value and class
  paylsymbol->symbol_put_value(_f.obclass);
  rps_add_root_object(_f.obclass);

  // Add to global class set
  auto paylsetcla = _f.obthemutsetclasses->get_dynamic_payload<Rps_PayloadSetOb>();
  paylsetcla->add(_f.obclass);

  return _f.obclass;
}
```

**Purpose**: Create named classes with proper inheritance, symbol registration, and global tracking.

### Symbol Creation
```cpp
Rps_ObjectRef Rps_ObjectRef::make_new_symbol(Rps_CallFrame* callerframe,
                                            std::string name, bool isweak)
{
  std::lock_guard<std::mutex> gusymb(rps_symbol_mtx);

  if (!Rps_PayloadSymbol::valid_name(name))
    throw std::runtime_error("invalid symbol name");

  if (Rps_PayloadSymbol::find_named_object(name))
    throw std::runtime_error("symbol already exists");

  _f.obsymbol = Rps_ObjectZone::make();
  _f.obsymbol->ob_class.store(RPS_ROOT_OB(_36I1BY2NetN03WjrOv)); // symbol∈class

  Rps_PayloadSymbol::register_name(name, _f.obsymbol, isweak);

  return _f.obsymbol;
}
```

**Purpose**: Create new symbols with global registration and weak/strong reference options.

### Object Creation
```cpp
Rps_ObjectRef Rps_ObjectRef::make_object(Rps_CallFrame* callerframe,
                                        Rps_ObjectRef classobarg, Rps_ObjectRef spaceobarg)
{
  _f.classob = classobarg;
  _f.spaceob = spaceobarg;

  // Validate space
  if (_f.spaceob && !is_instance_of(_f.spaceob, RPS_ROOT_OB(_2i66FFjmS7n03HNNBx)))
    throw RPS_RUNTIME_ERROR_OUT("invalid spaceob");

  // Validate class
  if (!_f.classob || !is_subclass_of(_f.classob, RPS_ROOT_OB(_5yhJGgxLwLp00X0xEQ)))
    throw RPS_RUNTIME_ERROR_OUT("invalid class");

  _f.resultob = Rps_ObjectZone::make();
  _f.resultob->ob_class.store(_f.classob);
  _f.resultob->put_space(_f.spaceob);

  return _f.resultob;
}
```

**Purpose**: Create objects with proper class and space validation.

## Persistence and Serialization

### Object Serialization
```cpp
void Rps_ObjectZone::dump_json_content(Rps_Dumper* du, Json::Value& json) const
{
  json["class"] = rps_dump_json_objectref(du, ob_class.load());
  json["mtime"] = Json::Value(get_mtime());

  // Magic getter function
  rps_magicgetterfun_t* mgfun = ob_magicgetterfun.load();
  if (mgfun) {
    // Extract function name from symbol table
    Dl_info di = {};
    if (dladdr((void*)mgfun, &di)) {
      if (di.dli_sname && strncmp(di.dli_sname, RPS_GETTERFUN_PREFIX, ...) == 0) {
        Rps_Id oidfun(di.dli_sname + sizeof(RPS_GETTERFUN_PREFIX), &pend, &ok);
        if (ok && oidfun) json["magicgetter"] = oidfun.to_string();
      }
    }
  }

  // Applying function
  rps_applyingfun_t* apfun = ob_applyingfun.load();
  if (apfun) {
    // Similar extraction for applying functions
  }

  // Attributes
  if (!ob_attrs.empty()) {
    Json::Value jattrs(Json::arrayValue);
    for (auto atit : ob_attrs) {
      if (rps_is_dumpable_objref(du, atit.first) &&
          rps_is_dumpable_value(du, atit.second)) {
        Json::Value jcurat(Json::objectValue);
        jcurat["at"] = rps_dump_json_objectref(du, atit.first);
        jcurat["va"] = rps_dump_json_value(du, atit.second);
        jattrs.append(jcurat);
      }
    }
    json["attrs"] = jattrs;
  }

  // Components
  if (!ob_comps.empty()) {
    Json::Value jcomps(Json::arrayValue);
    for (auto compv : ob_comps) {
      jcomps.append(rps_dump_json_value(du, compv));
    }
    json["comps"] = jcomps;
  }

  // Payload
  Rps_Payload* payl = ob_payload.load();
  if (payl && payl->owner() == this) {
    json["payload"] = Json::Value(payl->payload_type_name());
    payl->dump_json_content(du, json);
  }
}
```

**Purpose**: Serialize complete object state including metadata, attributes, components, and dynamic payloads.

### Object Lookup by String
```cpp
Rps_ObjectRef Rps_ObjectRef::find_object_by_string(Rps_CallFrame* callerframe,
                                                  const std::string& str,
                                                  Find_Behavior_en behav)
{
  if (str.empty()) {
    if (behav == Rps_Null_When_Missing) return Rps_ObjectRef(nullptr);
    throw std::runtime_error("empty string");
  }

  if (isalpha(str[0])) {
    // Try symbol lookup first
    _f.obsymb = Rps_PayloadSymbol::find_named_object(str);
    if (_f.obsymb) {
      auto symbpayl = _f.obsymb->get_dynamic_payload<Rps_PayloadSymbol>();
      if (symbpayl->symbol_value().is_object())
        _f.obfound = symbpayl->symbol_value().as_object();
      if (!_f.obfound) _f.obfound = _f.obsymb;
    }
  } else if (str[0] == '_') {
    // Direct OID lookup
    Rps_Id id(str);
    if (!id) {
      if (behav == Rps_Null_When_Missing) return Rps_ObjectRef(nullptr);
      throw std::runtime_error("bad id " + str);
    }
    _f.obfound = Rps_ObjectRef(Rps_ObjectZone::find(id));
  } else {
    if (behav == Rps_Null_When_Missing) return Rps_ObjectRef(nullptr);
    throw std::runtime_error("bad string " + str);
  }

  return _f.obfound;
}
```

**Purpose**: Provide flexible object lookup by name or OID with configurable error handling.

## Usage Patterns

### Creating Classes and Objects
```cpp
// Create a named class
Rps_ObjectRef myclass = Rps_ObjectRef::make_named_class(&_,
  RPS_ROOT_OB(_5yhJGgxLwLp00X0xEQ), // object∈class
  "MyClass");

// Create an object of that class
Rps_ObjectRef myobj = Rps_ObjectRef::make_object(&_, myclass, nullptr);

// Set attributes
myobj->put_attr(RPS_ROOT_OB(_1EBVGSfW2m200z18rx), // name∈named_attribute
                Rps_Value("my object"));

// Add components
myobj->append_comp1(Rps_Value(42));
myobj->append_comp1(Rps_Value("hello"));
```

### Method Installation
```cpp
// Create method selector
Rps_ObjectRef selector = Rps_ObjectRef::make_new_strong_symbol(&_, "print");

// Create closure (would be created from code)
Rps_Value closure = make_closure_from_code(...);

// Install method on class
myclass->install_own_method(&_, selector, closure);
```

### Symbol Management
```cpp
// Create a symbol
Rps_ObjectRef mysymb = Rps_ObjectRef::make_new_symbol(&_, "my_symbol", false);

// Set symbol value
auto symbpayl = mysymb->get_dynamic_payload<Rps_PayloadSymbol>();
symbpayl->symbol_put_value(myobj);

// Look up by name
Rps_ObjectRef found = Rps_PayloadSymbol::find_named_object("my_symbol");
```

### Working with Collections
```cpp
// Create a mutable set
Rps_ObjectRef myset = Rps_PayloadSetOb::make_mutable_set_object(&_,
  nullptr, nullptr);

// Add objects to set
auto setpayl = myset->get_dynamic_payload<Rps_PayloadSetOb>();
setpayl->add(myobj);

// Create a vector of values
Rps_ObjectRef myvect = Rps_ObjectRef::make_object(&_,
  RPS_ROOT_OB(_2A8l8zNp1tI02PkLLw), nullptr); // vector∈class
auto vectpayl = myvect->put_new_plain_payload<Rps_PayloadVectVal>();
vectpayl->pvectval = {Rps_Value(1), Rps_Value(2), Rps_Value(3)};
```

## Design Rationale

### Unified Object Model
**Why single object representation?**
- **Simplicity**: One core object type reduces complexity
- **Flexibility**: Payload system allows dynamic behavior extension
- **Performance**: Efficient memory layout and access patterns
- **Reflection**: Unified interface for introspection and modification

### Payload Architecture
**Why dynamic payloads?**
- **Memory Efficiency**: Only allocate what you need
- **Type Safety**: Compile-time type checking for payload access
- **Extensibility**: New payload types can be added without core changes
- **Performance**: Fast type-specific operations

### Thread-Safe Operations
**Why recursive mutexes?**
- **Reentrancy**: Allow nested operations on same object
- **Deadlock Prevention**: Consistent locking order
- **Performance**: Fine-grained locking for concurrent access
- **Safety**: Prevent race conditions in complex operations

### Symbol Registry
**Why global symbol table?**
- **Fast Lookup**: O(1) average case symbol resolution
- **Uniqueness**: Prevent symbol name conflicts
- **Persistence**: Symbols survive object graph changes
- **Integration**: Connects naming system with object system

### Attribute vs Component Distinction
**Why both attributes and components?**
- **Named Access**: Attributes provide semantic meaning
- **Positional Access**: Components support ordered collections
- **Flexibility**: Choose appropriate access pattern per use case
- **Performance**: Optimize for common access patterns

This implementation establishes the foundational object system that makes RefPerSys's homoiconic, reflective nature possible, providing the infrastructure for code and data unification while maintaining performance, safety, and extensibility.