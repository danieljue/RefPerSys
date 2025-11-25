# Values Implementation Analysis (`values_rps.cc`)

## Overview

The `values_rps.cc` file implements the core value system for the Reflective Persistent System (RefPerSys), providing immutable values, collections, closures, and the fundamental message sending protocol that enables object-oriented programming in RefPerSys. This module defines the essential data structures and operations that form the foundation of the reflective programming environment.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2019-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// Git versioning information
extern "C" const char rps_values_gitid[];
extern "C" const char rps_values_date[];
extern "C" const char rps_values_shortgitid[];
extern "C" const char rps_values_timestamp[];
```

### Key Dependencies
- **refpersys.hh**: Core system definitions and Rps_Value, Rps_ObjectRef types
- **Standard library**: STL containers, algorithms, and memory management

## Rps_Id Class: Base62 Object Identifiers

### Base62 Encoding Scheme
```cpp
const char Rps_Id::b62digits[] = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
const int Rps_Id::base = 62;
const int Rps_Id::nbdigits_hi = 11;
const int Rps_Id::nbdigits_lo = 13;
```

**Format**: `_` + 11 high digits + 13 low digits = 25-character string

### String Conversion Methods
```cpp
void Rps_Id::to_cbuf24(char cbuf[]) const
```
**Purpose**: Convert 128-bit ID to base62 string representation
- **Algorithm**: Divide high and low 64-bit parts by 62, map remainders to digits
- **Output**: 24-character buffer with null termination

```cpp
Rps_Id::Rps_Id(const char*cbuf, const char**pend, bool *pok)
```
**Purpose**: Parse base62 string back to Rps_Id
- **Validation**: Check leading `_`, valid digit ranges
- **Error Handling**: Set `*pok = false` and `*pend = cbuf` on failure
- **Bounds Checking**: Ensure high/low values within valid ranges

**Example Conversions**:
- `_0abcdefghijABCDEFG` ↔ Valid 128-bit identifier
- Invalid cases: missing `_`, invalid characters, out-of-range values

## Rps_QuasiZone: Garbage-Collected Memory Management

### QuasiZone Architecture
```cpp
class Rps_QuasiZone {
    static std::recursive_mutex qz_mtx;
    static std::vector<Rps_QuasiZone*> qz_zonvec;
    static uint32_t qz_cnt;
    static std::atomic<uint64_t> qz_alloc_cumulw;
    uint32_t qz_rank;
};
```

**Purpose**: Manage garbage-collected objects with efficient allocation and deallocation.

### Zone Registration System
```cpp
void Rps_QuasiZone::register_in_zonevec(void)
```
**Algorithm**:
1. **Capacity Check**: Expand vector if >80% full
2. **Slot Search**: Find empty slot using random probing
3. **Fallback**: Append to end if no empty slots found
4. **Registration**: Set rank and increment counter

```cpp
void Rps_QuasiZone::unregister_in_zonevec(void)
```
**Purpose**: Remove zone from tracking vector during destruction.

### Garbage Collection Integration
```cpp
void Rps_QuasiZone::clear_all_gcmarks(Rps_GarbageCollector& gc)
```
**Purpose**: Clear GC marks on all registered zones.

## Immutable Collections

### Rps_SetOb: Immutable Object Sets

#### Construction Methods
```cpp
const Rps_SetOb* Rps_SetOb::make(const std::set<Rps_ObjectRef>& setob)
const Rps_SetOb* Rps_SetOb::make(const std::initializer_list<Rps_ObjectRef>& elemil)
const Rps_SetOb* Rps_SetOb::make(const std::vector<Rps_ObjectRef>& vecob)
const Rps_SetOb* Rps_SetOb::collect(const std::vector<Rps_Value>& vecval)
const Rps_SetOb* Rps_SetOb::collect(const std::initializer_list<Rps_Value>& ilval)
```

**Features**:
- **Deduplication**: Automatic removal of duplicate objects
- **Null Filtering**: Skip null/empty object references
- **Value Collection**: Extract objects from tuples and sets
- **Size Limits**: Enforce maximum collection size

#### Output Formatting
```cpp
void Rps_SetOb::val_output(std::ostream& out, unsigned depth, unsigned maxdepth) const
```
**Format**: `{obj1, obj2, obj3}` with comma separation.

#### Class Computation
```cpp
Rps_ObjectRef Rps_SetOb::compute_class(Rps_CallFrame*) const
```
**Returns**: `RPS_ROOT_OB(_6JYterg6iAu00cV9Ye)` (the `set` class)

#### Element Iteration
```cpp
void Rps_SetOb::repeat_increasing_each_element_until(Rps_CallFrame* cf, void* data,
    const std::function<bool(Rps_CallFrame*, void*, Rps_ObjectRef)>& func) const

void Rps_SetOb::repeat_decreasing_each_element_until(Rps_CallFrame* cf, void* data,
    const std::function<bool(Rps_CallFrame*, void*, Rps_ObjectRef)>& func) const
```

**Purpose**: Iterate over set elements in sorted order with early termination capability.

### Rps_TupleOb: Immutable Object Tuples

#### Construction Methods
```cpp
const Rps_TupleOb* Rps_TupleOb::make(const std::vector<Rps_ObjectRef>& vecob)
const Rps_TupleOb* Rps_TupleOb::make(const std::initializer_list<Rps_ObjectRef>& compil)
const Rps_TupleOb* Rps_TupleOb::collect(const std::vector<Rps_Value>& vecval)
const Rps_TupleOb* Rps_TupleOb::collect(const std::initializer_list<Rps_Value>& valil)
```

**Features**:
- **Order Preservation**: Maintains input order of objects
- **Null Filtering**: Skip null references during collection
- **Nested Collection**: Extract objects from nested values
- **Size Validation**: Check against maximum tuple size

#### Output Formatting
```cpp
void Rps_TupleOb::val_output(std::ostream& out, unsigned depth, unsigned maxdepth) const
```
**Format**: `[obj1, obj2, obj3]` with comma separation.

#### Class Computation
```cpp
Rps_ObjectRef Rps_TupleOb::compute_class(Rps_CallFrame*) const
```
**Returns**: `RPS_ROOT_OB(_6NVM7sMcITg01ug5TC)` (the `tuple` class)

## Closure System

### Rps_ClosureZone: Closure Storage
```cpp
class Rps_ClosureZone : public Rps_QuasiZone {
    Rps_ObjectRef conn;  // Connection object
    Rps_Value* sons;     // Closure arguments
};
```

#### Construction
```cpp
Rps_ClosureZone* Rps_ClosureZone::make(Rps_ObjectRef connob,
                                      const std::initializer_list<Rps_Value>& valil)
Rps_ClosureZone* Rps_ClosureZone::make(Rps_ObjectRef connob,
                                      const std::vector<Rps_Value>& valvec)
```

**Purpose**: Create closures with connection object and argument values.

#### Output Formatting
```cpp
void Rps_ClosureZone::val_output(std::ostream& out, unsigned depth, unsigned maxdepth) const
```
**Format**: `%connection(arg1, arg2, arg3)`

#### Class Computation
```cpp
Rps_ObjectRef Rps_ClosureZone::compute_class(Rps_CallFrame*) const
```
**Returns**: `RPS_ROOT_OB(_4jISxMJ4PYU0050nUl)` (the `closure` class)

### Rps_ClosureValue: Closure Interface

#### Application Methods
```cpp
Rps_TwoValues Rps_ClosureValue::apply_vect(Rps_CallFrame* callerframe,
                                          const std::vector<Rps_Value>& argvec) const
Rps_TwoValues Rps_ClosureValue::apply_ilist(Rps_CallFrame* callerframe,
                                           const std::initializer_list<Rps_Value>& argil) const
```

**Arity Handling**:
- **0-3 args**: Direct application methods (`apply0`, `apply1`, `apply2`, `apply3`)
- **4+ args**: Use `rps_applyingfun_t` function pointer with rest arguments

## Message Sending Protocol

### Core Message Sending Architecture
The message sending protocol enables object-oriented method dispatch through selector-based method resolution.

#### Method Resolution Process
```cpp
Rps_ClosureValue Rps_Value::closure_for_method_selector(Rps_CallFrame* callerframe,
                                                       Rps_ObjectRef obselectorarg) const
```

**Algorithm**:
1. **Class Determination**: Compute receiver's class
2. **Inheritance Traversal**: Walk class hierarchy from specific to general
3. **Method Lookup**: Search for selector in class method dictionary
4. **Superclass Continuation**: Move to superclass if method not found
5. **Termination**: Stop at `value` class (top of hierarchy)

#### Send Methods (0-9 arguments)
```cpp
Rps_TwoValues Rps_Value::send0(Rps_CallFrame* callerframe, const Rps_ObjectRef obsel) const
Rps_TwoValues Rps_Value::send1(Rps_CallFrame* callerframe, const Rps_ObjectRef obsel,
                              Rps_Value arg0) const
// ... up to send9
```

**Common Pattern**:
1. **Frame Setup**: Create local call frame with message sending symbol
2. **Method Resolution**: Find closure for selector
3. **Application**: Apply closure with self + arguments
4. **Return Values**: Two-value result (primary + secondary)

#### Vector and Initializer List Variants
```cpp
Rps_TwoValues Rps_Value::send_vect(Rps_CallFrame* callerframe, const Rps_ObjectRef obsel,
                                  const std::vector<Rps_Value>& argvec) const
Rps_TwoValues Rps_Value::send_ilist(Rps_CallFrame* callerframe, const Rps_ObjectRef obsel,
                                   const std::initializer_list<Rps_Value>& argil) const
```

**Features**:
- **Variable Arity**: Handle arbitrary number of arguments
- **GC Safety**: Register arguments with garbage collector
- **Self Prepending**: Automatically add receiver as first argument

## Attribute Access System

### Attribute Retrieval
```cpp
Rps_Value Rps_Value::get_attr(Rps_CallFrame* stkf, const Rps_ObjectRef obattr) const
```

**Resolution Order**:
1. **Magic Getter**: Check for custom attribute getter function
2. **Object Attributes**: Search object's attribute map
3. **Fallback**: Return null value

### Class Computation
```cpp
Rps_ObjectRef Rps_Value::compute_class(Rps_CallFrame* stkf) const
```

**Type-Specific Classes**:
- **Empty**: `nullptr`
- **Integer**: `RPS_ROOT_OB(_2A2mrPpR3Qf03p6o5b)` (int class)
- **String**: `RPS_ROOT_OB(_62LTwxwKpQ802SsmjE)` (string class)
- **Double**: `RPS_ROOT_OB(_98sc8kSOXV003i86w5)` (double class)
- **Tuple**: `RPS_ROOT_OB(_6NVM7sMcITg01ug5TC)` (tuple class)
- **Set**: `RPS_ROOT_OB(_6JYterg6iAu00cV9Ye)` (set class)
- **Closure**: `RPS_ROOT_OB(_4jISxMJ4PYU0050nUl)` (closure class)
- **Pointer**: Delegate to pointer's `compute_class` method

## Output and Debugging Utilities

### Value Printing Functions
```cpp
void rps_print_value(const Rps_Value val)
void rps_print_ptr_value(const void* v)
void rps_limited_print_value(const Rps_Value val, unsigned depth, unsigned maxdepth)
void rps_limited_print_ptr_value(const void* v, unsigned depth, unsigned maxdepth)
```

**Purpose**: GDB-compatible printing functions for debugging.

### Vector Output Operator
```cpp
std::ostream& operator<<(std::ostream& out, const std::vector<Rps_Value>& vect)
```
**Format**: `(|val1,val2,val3,val4\n val5,val6,...|)` with line breaks every 4 elements.

## Usage Examples

### Object Identifier Operations
```cpp
// Create and manipulate object IDs
Rps_Id oid = Rps_Id::random();  // Generate random ID
char buf[25];
oid.to_cbuf24(buf);             // Convert to string: "_0abcdefghijABCDEFG"

// Parse from string
const char* endptr;
bool valid;
Rps_Id parsed_id("_0abcdefghijABCDEFG", &endptr, &valid);
if (valid) {
    // Use parsed_id
}
```

### Immutable Collections
```cpp
// Create sets
auto myset = Rps_SetOb::make({obj1, obj2, obj3});
auto emptyset = Rps_SetOb::_setob_emptyset_;

// Create tuples
auto mytuple = Rps_TupleOb::make({obj1, obj2, obj3});

// Collect objects from values
std::vector<Rps_Value> values = {obj1, Rps_TupleOb::make({obj2, obj3}), myset};
auto collected = Rps_SetOb::collect(values);  // Extracts all objects
```

### Closure Creation and Application
```cpp
// Create closure
auto myclosure = Rps_ClosureZone::make(connection_obj, {arg1, arg2, arg3});

// Apply closure
Rps_TwoValues result = myclosure.apply2(caller_frame, receiver, arg1);
```

### Message Sending
```cpp
// Send messages with different arities
Rps_Value receiver = some_object;
Rps_ObjectRef selector = some_selector;

auto result0 = receiver.send0(frame, selector);
auto result1 = receiver.send1(frame, selector, arg1);
auto result2 = receiver.send2(frame, selector, arg1, arg2);

// Send with variable arguments
std::vector<Rps_Value> args = {arg1, arg2, arg3};
auto result_vec = receiver.send_vect(frame, selector, args);

// Send with initializer list
auto result_ilist = receiver.send_ilist(frame, selector, {arg1, arg2});
```

### Attribute Access
```cpp
// Get attribute value
Rps_Value attr_value = object.get_attr(frame, attribute_selector);

// Compute class of value
Rps_ObjectRef value_class = some_value.compute_class(frame);
```

### Method Resolution
```cpp
// Get closure for method
Rps_ClosureValue method_closure =
    receiver.closure_for_method_selector(frame, selector);

// Apply method closure
Rps_TwoValues result = method_closure.apply1(frame, receiver);
```

This implementation provides the fundamental value system that enables RefPerSys's reflective, object-oriented programming model with immutable collections, closures, and dynamic message sending.