# RefPerSys Self-Referential Behavior: A Concrete Example

## The Meta-Object Protocol in Action

RefPerSys demonstrates profound self-referential behavior through its **meta-object protocol**, where objects can inspect and determine their own class relationships. This creates a homoiconic system where code and data are unified, and the system can reason about its own structure.

## Core Self-Referential Mechanism: `compute_class()`

### The Fundamental Self-Reference

Every value in RefPerSys can determine its own class through the `compute_class()` method:

```cpp
// From values_rps.cc lines 671-691
Rps_ObjectRef
Rps_Value::compute_class(Rps_CallFrame*stkf) const
{
  if (is_empty())
    return nullptr;
  else if (is_int())
    return RPS_ROOT_OB(_2A2mrPpR3Qf03p6o5b); // the `int` class
  else if (is_string())
    return RPS_ROOT_OB(_62LTwxwKpQ802SsmjE); // the `string` class
  else if (is_double())
    return RPS_ROOT_OB(_98sc8kSOXV003i86w5); // the `double` class
  else if (is_tuple())
    return RPS_ROOT_OB(_6NVM7sMcITg01ug5TC); // the `tuple` class
  else if (is_set())
    return RPS_ROOT_OB(_6JYterg6iAu00cV9Ye); // the `set` class
  else if (is_closure())
    return RPS_ROOT_OB(_4jISxMJ4PYU0050nUl); // the `closure` class
  else if (is_ptr())
    return as_ptr()->compute_class(stkf);  // DELEGATION: Objects compute their own class
  return nullptr;
} // end Rps_Value::compute_class
```

**Self-Referential Insight**: The `compute_class()` method for objects delegates to the object itself (`as_ptr()->compute_class(stkf)`), creating a circular relationship where objects determine their own type.

### Class Objects Are Also Objects

Classes in RefPerSys are themselves objects with the class `class`:

```cpp
// From objects_rps.cc lines 1459-1463
Rps_ObjectRef
Rps_ObjectZone::compute_class(Rps_CallFrame*) const
{
  return get_class();  // Returns this object's class attribute
} // end Rps_ObjectZone::compute_class
```

**Self-Referential Loop**: An object asks itself "what is my class?" and the answer comes from its own `get_class()` method, which returns a reference to another object (the class object), which itself has a class (the `class` class).

## Message Sending: Objects Calling Methods on Themselves

### The `send0()` Method: Self-Messaging

When an object sends a message to itself, it first determines its own class:

```cpp
// From values_rps.cc lines 798-825
Rps_TwoValues
Rps_Value::send0(Rps_CallFrame*callerframe, const Rps_ObjectRef obselarg) const
{
  RPS_LOCALFRAME(RPS_ROOT_OB(_5yQcFbU0seU018B48Z), // `message_sending` symbol
                 callerframe,
                 Rps_Value selfv; // the receiver (self)
                 Rps_ClosureValue closv; // the method to call
                 Rps_ObjectRef obsel; // the selector
                );
  _f.selfv = Rps_Value(*this);
  _f.obsel = obselarg;
  RPS_DEBUG_LOG(MSGSEND, "send0 selfv=" << _f.selfv
                << " of class:" <<  _f.selfv.compute_class(&_)); // SELF-REFERENCE HERE
  _f.closv = _f.selfv.closure_for_method_selector(&_,_f.obsel);
  // ... method execution
}
```

**Self-Referential Pattern**: The object (`selfv`) calls `compute_class()` on itself to determine what class it belongs to, then looks up the appropriate method in that class's method dictionary.

### Method Resolution: Classes Inspecting Themselves

The `closure_for_method_selector()` method walks the class hierarchy:

```cpp
// From values_rps.cc lines 696-787
Rps_ClosureValue
Rps_Value::closure_for_method_selector(Rps_CallFrame*callerframe, Rps_ObjectRef obselectorarg) const
{
  RPS_LOCALFRAME(RPS_ROOT_OB(_6JbWqOsjX5T03M1eGM), // `closure_for_method_selector` symbol
                 callerframe,
                 Rps_Value val; // the current value
                 Rps_ObjectRef obselect; // the method selector
                 Rps_ObjectRef obcurclass; // current class in hierarchy walk
                 Rps_ClosureValue closval; // result closure
                );
  _f.val = Rps_Value(*this);
  _f.obselect = obselectorarg;
  _f.obcurclass = _f.val.compute_class(&_); // SELF-REFERENCE: ask self for class

  while (loopcount++ < (int)maximal_inheritance_depth) {
    // Check if current class has the method
    if (_f.obcurclass == RPS_ROOT_OB(_41OFI3r0S1t03qdB2E)) { // the `class` class
      auto valclasspayl = reinterpret_cast<Rps_PayloadClassInfo*>(_f.obcurclass->get_payload());
      _f.closval = valclasspayl->get_own_method(_f.obselect);
      if (_f.closval && _f.closval.is_closure())
        return _f.closval;
      else {
        _f.obcurclass = valclasspayl->superclass(); // Walk up hierarchy
        continue;
      }
    }
  }
}
```

**Deep Self-Reference**: The object asks itself for its class, then the class object (which is also an object) is inspected for methods, and if the class is the `class` class, it looks up methods in its own payload.

## Classes as First-Class Objects

### Class Objects Have Their Own Class

The class of all classes is the `class` class:

```cpp
// From refpersys.hh - Class hierarchy
RPS_ROOT_OB(_41OFI3r0S1t03qdB2E) // class∈class - the class of all classes
RPS_ROOT_OB(_6XLY6QfcDre02922jz) // value∈class - root of value hierarchy
```

**Self-Referential Paradox**: The `class` class is an instance of itself. When you ask "what is the class of the `class` class?", the answer is the `class` class itself.

### Class Objects Can Be Manipulated Like Data

```cpp
// Classes can have attributes, just like regular objects
Rps_ObjectRef class_object = RPS_ROOT_OB(_41OFI3r0S1t03qdB2E); // the `class` class
class_object->put_attr(some_attribute, some_value); // Classes can be modified

// Classes can receive messages
Rps_TwoValues result = class_object.send1(&_, some_selector, some_arg);
```

## Code Generation: Objects Generating Code About Themselves

### Self-Referential Code Generation

When generating code, objects create representations of themselves:

```cpp
// From cppgen_rps.cc - Objects generate C++ code describing themselves
void emit_cplusplus_declarations(Rps_CallFrame*callerframe, Rps_ObjectRef argmodule) {
  for (int cix = 0; cix < module->nb_components(&_); cix++) {
    Rps_Value component = module->component_at(&_, cix);
    Rps_ObjectRef compclass = component.compute_class(&_); // SELF-REFERENCE
    
    // Generate C++ declaration based on component's class
    if (compclass == RPS_ROOT_OB(_41OFI3r0S1t03qdB2E)) { // class class
      // Generate class declaration
    } else if (compclass == RPS_ROOT_OB(_4jISxMJ4PYU0050nUl)) { // closure class
      // Generate function declaration
    }
    // ... etc
  }
}
```

**Self-Referential Code Generation**: Objects examine their own class to determine how to represent themselves in generated code.

## The Bootstrap Paradox

### How Does the System Know About Itself?

**The Bootstrap Problem**: How does a class know it belongs to the `class` class without creating infinite recursion?

**Solution**: Root objects are pre-defined in the system:

```cpp
// From generated/rps-roots.hh - Pre-defined root objects
RPS_ROOT_OB(_41OFI3r0S1t03qdB2E) // class∈class
RPS_ROOT_OB(_6XLY6QfcDre02922jz) // value∈class

// These are defined at compile time, breaking the self-reference loop
#define RPS_ROOT_OB(Oid) extern Rps_ObjectRef RPS_ROOT_OB(Oid)
#include "generated/rps-roots.hh"
```

**The Paradox Resolved**: The system starts with a set of pre-defined root objects that establish the basic class relationships. The `class` class knows it belongs to itself because this relationship is baked into the system's foundation.

## Practical Self-Referential Operations

### REPL Inspection: Objects Describing Themselves

```repl
# Ask an object about its own class
> show some_object.compute_class()

# Ask a class about its own class (returns itself)
> show class.compute_class()  # Returns 'class'

# Objects can send messages to themselves
> some_object.send0(display_selector)
```

### Dynamic Class Creation

```cpp
// Create a new class (which becomes an object of class 'class')
Rps_ObjectRef new_class = Rps_ObjectRef::make_named_class(&_,
  superclass, "MyNewClass");

// The new class object can now receive messages
Rps_TwoValues result = new_class.send1(&_, add_method_selector, method_closure);
```

## Implications for Homoiconicity

### Code and Data Unity

1. **Objects Represent Code**: Classes and methods are objects that can be manipulated
2. **Code Represents Objects**: Generated code describes object structures
3. **Self-Inspection**: Objects can examine their own class relationships
4. **Self-Modification**: Objects can change their own behavior through method addition

### Reflective Capabilities

- **Introspection**: Objects know their own class hierarchy
- **Dynamic Dispatch**: Method lookup walks class hierarchies at runtime
- **Meta-Programming**: Code can generate code that describes itself
- **Self-Extension**: Objects can add methods to their own classes

## Conclusion

RefPerSys's self-referential architecture creates a truly homoiconic system where:

1. **Objects know their own classes** through `compute_class()`
2. **Classes are objects** that belong to the `class` class
3. **Message sending** involves objects asking themselves for method implementations
4. **Code generation** involves objects creating representations of their own structure
5. **The system can reason about itself** through its meta-object protocol

This self-referential design enables powerful reflective and meta-programming capabilities, where the system can inspect, understand, and modify its own structure and behavior at runtime.