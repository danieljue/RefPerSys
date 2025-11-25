# BisonC++ Polymorphic Semantic Values Template (`bisonc++polymorphic`)

## Overview

The `bisonc++polymorphic` file provides template support for polymorphic semantic values in BisonC++ parsers. Unlike traditional yacc/bison parsers that use unions for semantic values, this system allows different types of semantic values to coexist safely through runtime type identification and automatic memory management.

## Core Architecture

### Type System Foundation
```cpp
extern size_t const *s_nErrors_;    // Error counter for polymorphic operations

// Type tag system
template <typename Tp_>
struct TagOf;                      // Maps C++ types to tags

template <Tag_ tag>
struct TypeOf;                     // Maps tags to C++ types

$insert polymorphicSpecializations  // Generated type mappings
```

**Purpose**: Establishes bidirectional mapping between C++ types and runtime type tags.

### Base Semantic Value Class
```cpp
class Base {
protected:
    Tag_ d_baseTag;        // Runtime type identifier

public:
    Base() = default;
    Base(Base const &other) = delete;  // Prevent slicing

    virtual ~Base();       // Virtual destructor for cleanup

    Tag_ tag() const { return d_baseTag; }
    Base *clone() const;   // Deep copy
    void *data() const;    // Raw data access

private:
    virtual Base *vClone() const = 0;    // Virtual clone implementation
    virtual void *vData() const = 0;     // Virtual data access
};
```

**Key Features**:
- **Type Safety**: Runtime type identification prevents type mismatches
- **Memory Management**: Virtual destructor ensures proper cleanup
- **Clone Support**: Deep copying for parser stack operations

### Template Semantic Value Class
```cpp
template <Tag_ tg_>
class Semantic : public Base {
    typename TypeOf<tg_>::type d_data;    // Actual data storage

public:
    Semantic() { d_baseTag = tg_; }

    // Constructor template for any data type
    template <typename ...Params>
    Semantic(Params &&...params)
        : d_data(std::forward<Params>(params)...) {
        d_baseTag = tg_;
    }

    // Copy constructor
    Semantic(Semantic<tg_> const &other)
        : d_data(other.d_data) {
        d_baseTag = other.d_baseTag;
    }

private:
    Base *vClone() const override {
        return new Semantic<tg_>{*this};
    }

    void *vData() const override {
        return const_cast<typename TypeOf<tg_>::type*>(&d_data);
    }
};
```

**Template Features**:
- **Perfect Forwarding**: Accepts any constructor arguments
- **Type Deduction**: Automatically infers data type from tag
- **Memory Safety**: Proper copy semantics for stack operations

## SType Wrapper Class

### Smart Pointer Interface
```cpp
class SType : private std::unique_ptr<Base> {
    using BasePtr = std::unique_ptr<Base>;

public:
    SType();                                    // Default: END_TAG_
    SType(SType const &other);                  // Deep copy
    SType(SType &&tmp);                         // Move construction

    SType &operator=(SType const &rhs);         // Deep assignment
    SType &operator=(SType &rhs);               // Copy assignment
    SType &operator=(SType &&tmp);              // Move assignment

    // Type-safe accessors
    template <Tag_ tag>
    typename TypeOf<tag>::type &get();

    template <Tag_ tag>
    typename TypeOf<tag>::type const &get() const;

    Tag_ tag() const;                           // Runtime type query
};
```

### Type-Safe Operations
```cpp
// Type-safe assignment with any constructor arguments
template <Tag_ tagParam, typename ...Args>
void SType::assign(Args &&...args) {
    reset(new Semantic<tagParam>(std::forward<Args>(args)...));
}

// Type-safe access with runtime checking
template <Tag_ tg>
typename TypeOf<tg>::type &SType::get() {
    $insert warnTagMismatches  // Optional debug warnings
    return *static_cast<typename TypeOf<tg>::type*>(
        (*this)->data()
    );
}
```

## Usage Examples

### Basic Type Definitions
```cpp
// Generated specializations (example)
template <>
struct TagOf<int> { static Tag_ const tag = Tag_::INT; };

template <>
struct TypeOf<Tag_::INT> { using type = int; };

// More types...
template <>
struct TagOf<std::string> { static Tag_ const tag = Tag_::STRING; };

template <>
struct TypeOf<Tag_::STRING> { using type = std::string; };
```

### Parser Semantic Actions
```cpp
// In grammar rules
expression:
    NUMBER {
        $$.assign<Tag_::INT>($1);  // Store integer value
    }
    | STRING_LITERAL {
        $$.assign<Tag_::STRING>($1);  // Store string value
    }
    | expression '+' expression {
        // Type-safe access
        int left = $1.get<Tag_::INT>();
        int right = $3.get<Tag_::INT>();
        $$.assign<Tag_::INT>(left + right);
    }
```

### Stack Operations
```cpp
void parser::reduce_add() {
    // Automatic deep copying during stack operations
    SType result;
    result.assign<Tag_::INT>(
        vs_(0).get<Tag_::INT>() +   // Top of stack
        vs_(1).get<Tag_::INT>()     // Second from top
    );
    // result now owns the data and will clean it up
}
```

## Memory Management

### Ownership Semantics
```cpp
SType value;
value.assign<Tag_::STRING>("hello");  // Allocates Semantic<Tag_::STRING>

// Copy creates deep copy
SType copy = value;  // New allocation with cloned data

// Move transfers ownership
SType moved = std::move(value);  // No allocation, transfers pointer

// Assignment replaces contents
value = copy;  // Deep copy of copy's data
```

### Automatic Cleanup
```cpp
{
    SType temp;
    temp.assign<Tag_::VECTOR>(std::vector<int>{1,2,3});
    // temp owns the vector
} // temp destructor cleans up vector automatically
```

## Type Safety Features

### Runtime Type Checking
```cpp
SType value;
value.assign<Tag_::INT>(42);

// Safe access
int x = value.get<Tag_::INT>();  // OK

// Unsafe access (runtime error in debug mode)
std::string s = value.get<Tag_::STRING>();  // Type mismatch!
```

### Optional Debug Warnings
```cpp
$insert warnTagMismatches  // Can generate warnings on type mismatches
```

**Debug Output Example**:
```
Warning: Type mismatch in SType::get(): expected STRING, got INT
```

## Performance Characteristics

### Memory Overhead
- **Per Value**: `sizeof(Base)` + `sizeof(data)` + heap allocation overhead
- **Type Tag**: Minimal (enum-sized) runtime type information
- **Virtual Table**: Single vtable per instantiated `Semantic<>` type

### Runtime Performance
- **Type Checking**: Minimal overhead in release builds
- **Memory Operations**: Deep copy on assignment (necessary for safety)
- **Access Speed**: Direct pointer access after type check

### Optimization Opportunities
- **Small Object Optimization**: Could be added for small types
- **Copy Elision**: Move semantics reduce copying
- **Type Caching**: Repeated type checks could be cached

## Integration with Parser Generation

### Generated Code Structure
```cpp
// Generated by BisonC++
namespace Meta_ {

enum Tag_ {
    END_TAG_,     // Sentinel value
    INT,          // int type
    STRING,       // std::string type
    VECTOR,       // std::vector<int> type
    // ... more types from grammar
};

// Type mappings
template <> struct TagOf<int> { static Tag_ const tag = Tag_::INT; };
template <> struct TypeOf<Tag_::INT> { using type = int; };
// ... more mappings

using STYPE_ = SType;  // Parser's semantic value type

}  // namespace Meta_
```

### Grammar Integration
```cpp
// In .y grammar file
%polymorphic
    INT : int
    STRING : std::string
    VECTOR : std::vector<int>

// Rules can use different types
expression : NUMBER { $$.assign<INT>($1); }
           | STRING { $$.assign<STRING>($1); }
```

## Error Handling

### Type Mismatch Errors
```cpp
try {
    std::string s = value.get<Tag_::STRING>();
} catch (std::bad_cast const &e) {
    // Handle type mismatch
    error("Type mismatch in semantic value access");
}
```

### Memory Errors
- **Allocation failures**: Propagated as `std::bad_alloc`
- **Copy failures**: Exception-safe with strong guarantee
- **Destruction errors**: Virtual destructor handles cleanup

## Comparison with Union-Based Systems

### Advantages over Unions
- **Type Safety**: Runtime type checking prevents corruption
- **Flexibility**: Any C++ type can be used, not just POD types
- **Memory Safety**: Automatic cleanup prevents leaks
- **Extensibility**: Easy to add new types without grammar changes

### Performance Trade-offs
- **Memory**: Higher overhead than unions
- **Speed**: Type checking adds small runtime cost
- **Complexity**: More complex implementation

### When to Use Polymorphic Values
- **Complex Types**: Classes, containers, smart pointers
- **Type Safety**: Critical applications requiring runtime safety
- **Debugging**: Development with extensive error checking
- **Flexibility**: Grammars that mix many different value types

This polymorphic semantic value system provides a type-safe, flexible alternative to traditional union-based semantic values, enabling the use of complex C++ types in parser semantic actions while maintaining memory safety and automatic resource management.