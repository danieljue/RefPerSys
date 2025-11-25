# Output Implementation Analysis (`output_rps.cc`)

## Overview

The `output_rps.cc` file implements the pretty printing and display infrastructure for the Reflective Persistent System (RefPerSys), providing human-readable representations of values, objects, and their metadata. This file establishes the visual interface for debugging, logging, and interactive exploration of the RefPerSys object graph, using Unicode symbols, terminal formatting, and structured layouts to make complex data structures comprehensible.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2024-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// Git versioning information
extern "C" const char rps_output_gitid[];
extern "C" const char rps_output_date[];
extern "C" const char rps_output_shortgitid[];
extern "C" const char rps_output_timestamp[];
```

### Key Dependencies
- **dlfcn.h**: Dynamic library function address resolution
- **cxxabi.h**: C++ name demangling
- **Terminal control**: ANSI escape sequences for formatting
- **Unicode support**: UTF-8 symbols for visual clarity

## Rps_OutputValue: Value Formatting

### Class Overview
`Rps_OutputValue` provides controlled formatting of RefPerSys values with depth limiting to prevent infinite recursion in circular data structures.

### Structure and Construction
```cpp
class Rps_OutputValue {
  Rps_Value _out_val;      // The value to output
  unsigned _out_depth;     // Current nesting depth
  unsigned _out_maxdepth;  // Maximum allowed depth

public:
  Rps_OutputValue(Rps_Value val, unsigned depth, unsigned maxdepth)
    : _out_val(val), _out_depth(depth), _out_maxdepth(maxdepth) {}

  void do_output(std::ostream& out) const;
};
```

### Value Output Logic
```cpp
void Rps_OutputValue::do_output(std::ostream& out) const
{
  if (_out_depth > _out_maxdepth) {
    out << "?";
    return;
  }

  if (_out_val.is_empty()) {
    out << "*nil*";
    return;
  } else if (_out_val.is_int()) {
    out << _out_val.as_int();
    return;
  }

  const Rps_ZoneValue* outzv = _out_val.as_ptr();
  RPS_ASSERT(outzv);
  outzv->val_output(out, _out_depth, _out_maxdepth);
}
```

**Purpose**: Provides safe, depth-controlled output of values, delegating to the appropriate `val_output` method based on value type.

### Usage Pattern
```cpp
Rps_Value myval = get_some_value();
std::cout << Rps_OutputValue(myval, 0, 5); // Output with max depth 5
```

## Rps_Object_Display: Detailed Object Display

### Class Overview
`Rps_Object_Display` provides comprehensive, structured display of objects including metadata, attributes, components, and payloads. This is the primary debugging and inspection interface for RefPerSys objects.

### Structure and Construction
```cpp
class Rps_Object_Display {
  Rps_ObjectRef _dispobref;    // Object to display
  const char* _dispfile;       // Source file location
  int _displine;              // Source line number
  unsigned _dispdepth;        // Display depth

public:
  Rps_Object_Display(Rps_ObjectRef obr, const char* fil, int lin, unsigned depth = 0)
    : _dispobref(obr), _dispfile(fil), _displine(lin), _dispdepth(depth) {}

  void output_display(std::ostream& out) const;
  void output_routine_addr(std::ostream& out, void* funaddr) const;
};
```

### Comprehensive Object Display
```cpp
void Rps_Object_Display::output_display(std::ostream& out) const
{
  // Terminal detection and formatting setup
  bool ontty = (&out == &std::cout) ? isatty(STDOUT_FILENO)
           : (&out == &std::cerr) ? isatty(STDERR_FILENO) : false;
  if (rps_without_terminal_escape) ontty = false;

  const char* BOLD_esc = (ontty ? RPS_TERMINAL_BOLD_ESCAPE : "");
  const char* NORM_esc = (ontty ? RPS_TERMINAL_NORMAL_ESCAPE : "");

  // Thread-safe object locking
  std::lock_guard<std::recursive_mutex> gudispob(*_dispobref->objmtxptr());

  // Header with object identity and class
  out << std::endl << BOLD_esc << _dispobref << "::{¤¤ object " << NORM_esc
      << std::endl << "  of class " << _dispobref->get_class() << std::endl;

  // Space information
  Rps_ObjectRef obspace = _dispobref->get_space();
  if (!obspace.is_empty())
    out << "¤ in space " << _dispobref->get_space() << std::endl;
  else
    out << BOLD_esc << "¤ temporary" << NORM_esc << " space" << std::endl;

  // Modification time and hash
  double obmtim = _dispobref->get_mtime();
  char mtimbuf[64];
  rps_strftime_centiseconds(mtimbuf, sizeof(mtimbuf),
                           "%Y, %b, %d %H:%M:%S.__ %Z", obmtim);
  out << BOLD_esc << "** mtime: " << mtimbuf
      << "   *hash:" << _dispobref->val_hash() << NORM_esc << std::endl;

  // Function pointers (magic getter and applying functions)
  display_function_pointers(out, BOLD_esc, NORM_esc);

  // Physical attributes
  display_physical_attributes(out, BOLD_esc, NORM_esc);

  // Physical components
  display_physical_components(out, BOLD_esc, NORM_esc);

  // Payload information
  display_payload(out, BOLD_esc, NORM_esc);

  // Footer
  out << " " << BOLD_esc << "|-" << oidpref << "¤¤}" << NORM_esc << std::endl;
}
```

**Purpose**: Provides a complete, structured view of an object's state including identity, metadata, attributes, components, and dynamic behavior.

### Function Address Resolution
```cpp
void Rps_Object_Display::output_routine_addr(std::ostream& out, void* funaddr) const
{
  if (funaddr == nullptr) {
    out << "⏚"; // U+23DA EARTH GROUND
    return;
  }

  Dl_info adinf;
  memset(&adinf, 0, sizeof(adinf));

  if (dladdr(funaddr, &adinf)) {
    if (funaddr == adinf.dli_saddr) {
      // Exact symbol match
      out << "&" << adinf.dli_sname;

      // Demangle C++ names
      if (adinf.dli_sname[0] == '_' && adinf.dli_sname[1]) {
        int status = -1;
        const char* demangled = abi::__cxa_demangle(adinf.dli_sname, nullptr, 0, &status);
        if (demangled && status == 0 && demangled[0]) {
          out << "≡" << demangled; // U+2261 IDENTICAL TO
        }
        free((void*)demangled);
      }
      out << "=" << funaddr;
    } else {
      // Offset from symbol
      size_t delta = (const char*)funaddr - (const char*)adinf.dli_saddr;
      out << "&" << adinf.dli_sname << "+" << delta << "=" << funaddr;
    }

    // Include shared object name
    if (adinf.dli_fname)
      out << " in " << adinf.dli_fname;
  } else {
    out << "?" << funaddr;
  }
}
```

**Purpose**: Resolves function pointers to human-readable names with demangling and library identification.

## Terminal Formatting and ANSI Escape Sequences

### Terminal Detection and Control
```cpp
// Terminal capability detection
bool ontty = (&out == &std::cout) ? isatty(STDOUT_FILENO)
         : (&out == &std::cerr) ? isatty(STDERR_FILENO) : false;

// Respect user preference for plain text
if (rps_without_terminal_escape) ontty = false;

// ANSI escape sequences
#define RPS_TERMINAL_BOLD_ESCAPE "\033[1m"
#define RPS_TERMINAL_NORMAL_ESCAPE "\033[0m"
#define RPS_TERMINAL_ITALICS_ESCAPE "\033[3m"
#define RPS_TERMINAL_UNDERLINE_ESCAPE "\033[4m"
```

**Purpose**: Provides rich terminal formatting when appropriate, with fallback to plain text.

### Visual Hierarchy with Unicode Symbols
```cpp
// Object identity markers
"◌"  // U+25CC DOTTED CIRCLE - object prefix
"∊"  // U+2208 ELEMENT OF - class membership
"∈"  // U+2208 ELEMENT OF - space membership

// Structural markers
"⟦"  // U+27E6 MATHEMATICAL LEFT WHITE SQUARE BRACKET
"⟧"  // U+27E7 MATHEMATICAL RIGHT WHITE SQUARE BRACKET
"⏵"  // U+23F5 BLACK MEDIUM RIGHT-POINTING TRIANGLE - name indicator

// Function markers
"⊚"  // U+229A CIRCLED RING OPERATOR - function indicator
"⏚"  // U+23DA EARTH GROUND - null function
"≡"  // U+2261 IDENTICAL TO - demangled name separator

// Collection markers
"["  // Array indices
"*"  // Attribute bullets
"°"  // Component markers
```

**Purpose**: Uses Unicode symbols to create visual structure and type differentiation in output.

## Object Sorting for Display

### Display-Order Comparison
```cpp
void rps_sort_object_vector_for_display(std::vector<Rps_ObjectRef>& vectobr)
{
  std::sort(vectobr.begin(), vectobr.end(),
            [](Rps_ObjectRef leftob, Rps_ObjectRef rightob) {
    return Rps_ObjectRef::compare_for_display(leftob, rightob) < 0;
  });
}
```

**Purpose**: Sorts objects in a human-friendly order for consistent, readable display.

### Comparison Logic
```cpp
int Rps_ObjectRef::compare_for_display(const Rps_ObjectRef leftob,
                                      const Rps_ObjectRef rightob)
{
  if (leftob.optr() == rightob.optr()) return 0;
  if (leftob.is_empty()) return rightob.is_empty() ? 0 : -1;
  if (rightob.is_empty()) return leftob.is_empty() ? 0 : 1;

  // Compare by name first (alphabetically)
  std::string sleftname, srightname;
  // Extract names from objects...

  if (!sleftname.empty() && !srightname.empty()) {
    if (sleftname == srightname) {
      if (leftid < rightid) return -1; else return 1;
    } else {
      if (sleftname < srightname) return -1; else return 1;
    }
  }

  // Fallback to name presence, then OID
  if (!sleftname.empty()) return -1;
  else if (!srightname.empty()) return +1;
  if (leftid < rightid) return -1; else return +1;
}
```

**Purpose**: Provides stable, human-intuitive ordering prioritizing named objects and alphabetical names.

## Display Components Breakdown

### Function Pointer Display
```cpp
// Magic getter functions
rps_magicgetterfun_t* getfun = _dispobref->magic_getter_function();
if (getfun) {
  out << BOLD_esc << "⊚ magic attribute getter function " << NORM_esc;
  output_routine_addr(out, reinterpret_cast<void*>(getfun));
}

// Applying functions
rps_applyingfun_t* applfun = _dispobref->applying_function();
if (applfun) {
  out << BOLD_esc << "⊚ applying function " << NORM_esc;
  output_routine_addr(out, reinterpret_cast<void*>(applfun));
}
```

**Purpose**: Shows dynamic behavior attached to objects through function pointers.

### Attribute Display
```cpp
Rps_Value setphysattr = _dispobref->set_of_physical_attributes();
if (setphysattr.is_empty()) {
  out << BOLD_esc << "** no physical attributes **" << NORM_esc << std::endl;
} else {
  const Rps_SetOb* physattrset = setphysattr.as_set();
  unsigned nbphysattr = physattrset->cardinal();

  // Sort attributes for consistent display
  std::vector<Rps_ObjectRef> attrvect(nbphysattr);
  for (int ix = 0; ix < (int)nbphysattr; ix++)
    attrvect[ix] = physattrset->at(ix);
  rps_sort_object_vector_for_display(attrvect);

  // Display each attribute
  for (int ix = 0; ix < (int)nbphysattr; ix++) {
    const Rps_ObjectRef curattr = attrvect[ix];
    const Rps_Value curval = _dispobref->get_physical_attr(curattr);
    out << " " << BOLD_esc << "*" << NORM_esc << curattr << ": "
        << Rps_OutputValue(curval, _dispdepth, disp_max_depth) << std::endl;
  }
}
```

**Purpose**: Shows named attributes with their values in sorted, readable format.

### Component Display
```cpp
unsigned nbphyscomp = _dispobref->nb_physical_components();
const std::vector<Rps_Value> vectcomp = _dispobref->vector_physical_components();

for (unsigned ix = 0; ix < nbphyscomp; ix++) {
  out << BOLD_esc << " [" << ix << "]" << NORM_esc << " "
      << Rps_OutputValue(vectcomp[ix], _dispdepth, disp_max_depth) << std::endl;
}
```

**Purpose**: Shows positional components with indexed access.

### Payload Display
```cpp
Rps_Payload* payl = _dispobref->get_payload();
if (!payl) {
  out << BOLD_esc << "* no payload *" << NORM_esc << std::endl;
} else {
  out << BOLD_esc << "* " << payl->payload_type_name() << " payload *"
      << NORM_esc << std::endl;
  payl->output_payload(out, _dispdepth, disp_max_depth);
}
```

**Purpose**: Delegates to payload-specific display methods for type-appropriate formatting.

## Usage Patterns and Macros

### RPS_OBJECT_DISPLAY Macro
```cpp
// Macro for convenient object display (defined in refpersys.hh)
#define RPS_OBJECT_DISPLAY(Obj) \
  Rps_Object_Display(Obj, __FILE__, __LINE__).output_display(std::cout)

// Usage
Rps_ObjectRef myobj = get_some_object();
RPS_OBJECT_DISPLAY(myobj); // Detailed display to stdout
```

### Value Output
```cpp
// Direct value output
Rps_Value val = get_some_value();
std::cout << Rps_OutputValue(val, 0, 10) << std::endl;

// With depth control
std::cout << Rps_OutputValue(complex_val, 0, 3) << std::endl; // Limited depth
```

### Custom Display Streams
```cpp
// Display to different streams
std::ostringstream debug_output;
Rps_Object_Display(myobj, __FILE__, __LINE__).output_display(debug_output);

// Display with custom depth
Rps_Object_Display(myobj, __FILE__, __LINE__, 2).output_display(std::cerr);
```

## Design Rationale

### Rich Visual Formatting
**Why Unicode and ANSI escape codes?**
- **Information Density**: Unicode symbols convey meaning efficiently
- **Accessibility**: Plain text fallback when terminals don't support formatting
- **Professional Appearance**: Structured output aids debugging and development
- **Type Differentiation**: Visual cues help distinguish different object types

### Depth-Limited Output
**Why prevent infinite recursion?**
- **Safety**: Circular references won't cause infinite loops
- **Usability**: Output remains readable even for complex object graphs
- **Performance**: Prevents excessive output for large structures
- **Debugging**: Allows inspection of deep structures at controlled levels

### Thread-Safe Display
**Why lock objects during display?**
- **Consistency**: Object state remains stable during multi-line output
- **Thread Safety**: Prevents race conditions in concurrent environments
- **Atomicity**: Display shows consistent snapshot of object state
- **Debugging**: Ensures reliable output for debugging concurrent issues

### Function Address Resolution
**Why demangle and resolve function pointers?**
- **Debugging**: Shows meaningful function names instead of addresses
- **Development**: Helps trace dynamic behavior and method dispatch
- **Library Integration**: Identifies which shared libraries contain functions
- **Performance Analysis**: Supports profiling and optimization work

### Structured Layout
**Why hierarchical display format?**
- **Readability**: Clear visual separation of different object aspects
- **Completeness**: Shows all relevant object information
- **Consistency**: Standardized format across all object types
- **Extensibility**: Easy to add new display sections for future features

This implementation provides the visual interface for RefPerSys, transforming complex object graphs and values into human-comprehensible representations while maintaining safety, performance, and extensibility.