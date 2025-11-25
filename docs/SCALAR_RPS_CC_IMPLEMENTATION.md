# Scalar Implementation Analysis (`scalar_rps.cc`)

## Overview

The `scalar_rps.cc` file implements scalar value handling for the Reflective Persistent System (RefPerSys), focusing on string and numeric types with comprehensive UTF-8 support. This module provides the foundational infrastructure for text processing, numeric representation, and file system operations within the system.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2019-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// Git versioning information
extern "C" const char rps_scalar_gitid[];
extern "C" const char rps_scalar_date[];
extern "C" const char rps_scalar_shortgitid[];
extern "C" const char rps_scalar_timestamp[];
```

### Key Dependencies
- **refpersys.hh**: Core system definitions and Rps_Value classes
- **GNU libunistring**: UTF-8 processing (`u8_mbtouc`, `u8_next`, `u8_check`)
- **POSIX functions**: `wordexp`, `glob`, `realpath` for file operations

## UTF-8 String Hashing

### Core Hashing Function
```cpp
int rps_compute_cstr_two_64bits_hash(int64_t ht[2], const char*cstr, int len)
{
  // Computes two 64-bit hash values for UTF-8 strings
  // Uses libunistring for proper Unicode character handling
}
```

**Algorithm Overview**:
1. **Initialization**: Set up hash accumulators with length and magic constants
2. **UTF-8 Processing**: Iterate through string using `u8_mbtouc` for character extraction
3. **Rolling Hash**: Apply different prime multipliers for each character position
4. **Error Handling**: Return 0 on invalid UTF-8 sequences

### Hash Properties
- **Dual Hash**: Produces two independent 64-bit hash values for collision resistance
- **Unicode Aware**: Properly handles multi-byte UTF-8 sequences
- **Deterministic**: Same input always produces same output
- **Avalanche Effect**: Small input changes produce large hash differences

### Hash Constants
```cpp
// Carefully chosen prime numbers for hash computation
h0 = (h0 * 60869) ^ (uc1 * 5059 + (h1 & 0xff));
h1 = (h1 * 53087) ^ (uc2 * 43063 + utf8cnt + (h0 & 0xff));
// Additional constants: 73063, 53089, 73019, 23057
```

**Purpose**: Provides high-quality hashing for string interning and dictionary operations.

## Rps_String Class Implementation

### String Creation and Validation
```cpp
const Rps_String* Rps_String::make(const char*cstr, int len)
{
  // Normalize input parameters
  cstr = normalize_cstr(cstr);
  len = normalize_len(cstr, len);

  // Validate UTF-8 encoding
  if (u8_check(reinterpret_cast<const uint8_t*>(cstr), len))
    throw std::domain_error("invalid UTF-8 string");

  // Allocate string object with proper memory layout
  Rps_String* str = rps_allocate_with_wordgap<Rps_String>(
    len/sizeof(void*)+1, cstr, len);
  return str;
}
```

**Validation Process**:
1. **Normalization**: Handle null pointers and length calculation
2. **UTF-8 Check**: Use `u8_check` to validate encoding
3. **Memory Allocation**: Use word-aligned allocation for GC compatibility
4. **Exception Safety**: Throw on invalid UTF-8 sequences

### Serialization Support
```cpp
Json::Value Rps_String::dump_json(Rps_Dumper* du) const
{
  if (cstr()[0] == '_') {
    // Special handling for strings starting with underscore
    Json::Value vmap(Json::objectValue);
    vmap["str"] = cstr();
    return vmap;
  } else {
    return Json::Value(cstr());
  }
}
```

**Purpose**: Strings starting with '_' are treated specially for internal object naming.

### Output Formatting
```cpp
void Rps_String::val_output(std::ostream& out, unsigned depth, unsigned maxdepth) const
{
  Json::Value jstr(cstr());
  out << jstr;  // JSON-formatted output
}
```

### Class Association
```cpp
Rps_ObjectRef Rps_String::compute_class(Rps_CallFrame*) const
{
  return RPS_ROOT_OB(_62LTwxwKpQ802SsmjE); // string class
}
```

## Rps_Double Class Implementation

### Output Formatting
```cpp
void Rps_Double::val_output(std::ostream& out, unsigned depth, unsigned maxdepth) const
{
  if (depth > maxdepth) {
    out << "??";
    return;
  }
  out << dval();  // Direct double output
}
```

### Class Association
```cpp
Rps_ObjectRef Rps_Double::compute_class(Rps_CallFrame*) const
{
  return RPS_ROOT_OB(_98sc8kSOXV003i86w5); // double class
}
```

## UTF-8 Output Functions

### HTML Escaping
```cpp
void rps_output_utf8_html(std::ostream& out, const char* str, int bytlen, bool nl2br)
{
  // Convert UTF-8 string to HTML-safe output
  // Handles special characters and optional line break conversion
}
```

**Character Handling**:
- **HTML Entities**: `&` → `&`, `<` → `<`, `>` → `>`, etc.
- **Line Breaks**: Optional `<br/>` insertion for `\n` when `nl2br=true`
- **Unicode Characters**: Numeric entities for non-ASCII characters
- **Performance**: Optimized handling of common ASCII characters

### C/JSON Escaping
```cpp
void rps_output_utf8_cjson(std::ostream& out, const char* str, int bytlen)
{
  // Convert UTF-8 string to C string literal or JSON string format
  // Handles escape sequences for special characters
}
```

**Escape Sequences**:
- **Quotes**: `"` → `\"`, `'` → `\'`
- **Backslash**: `\` → `\\`
- **Control Characters**: `\n`, `\t`, `\r`, etc.
- **Unicode**: `\xHH`, `\uHHHH`, `\UHHHHHHHH` for non-ASCII

### Performance Optimizations
```cpp
// Fast path for common ASCII characters
case '0' ... '9':
case 'a' ... 'z':
case 'A' ... 'Z':
case '.':
case ',':
case ':':
case ';':
case '?':
case '_':
case '+':
case '-':
case '*':
case '/':
case '(':
case ')':
case '[':
case ']':
case '{':
case '}':
case '!':
case '=':
  out << (char)uc;  // Direct output without escaping
  break;
```

## File Path Globbing

### Plain File Path Resolution
```cpp
std::string rps_glob_plain_file_path(const char* shellpatt, const char* dirpath)
{
  // Resolve shell patterns to readable plain files
  // Searches through colon-separated directory paths
}
```

**Pattern Types Supported**:
1. **Tilde Expansion**: `~/file` or `~user/file` using `wordexp`
2. **Absolute Paths**: `/path/to/file` using `glob`
3. **Relative Paths**: Search through `dirpath` directories

### Search Algorithm
```cpp
// For relative paths in dirpath
std::string dirstr(dirpath);
for (const char* pc = dirstr.c_str(); pc && *pc; pc = colon ? colon+1 : nullptr) {
  colon = strchr(pc, ':');
  std::string curdir = colon ? std::string(pc, colon-pc) : std::string(pc);

  // Try glob in current directory
  std::string curpatt = curdir + "/" + shellpatt;
  glob_t g = {};
  if (glob(curpatt.c_str(), GLOB_ERR|GLOB_MARK, nullptr, &g) == 0) {
    if (g.gl_pathc == 1) {  // Exactly one match
      char* rp = realpath(g.gl_pathv[0], nullptr);
      // Validate it's a readable regular file
      if (is_readable_regular_file(rp)) {
        return std::string(rp);
      }
    }
  }
  globfree(&g);
}
```

### Validation Criteria
```cpp
// File must be:
if (stat(rp, &rs) || (rs.st_mode & S_IFMT) != S_IFREG || access(rp, R_OK)) {
  // Not a readable regular file
  free(rp);
  return std::string();
}
```

**Requirements**: File must exist, be a regular file, and be readable by the process.

## Usage Patterns and Examples

### String Hashing
```cpp
// Compute hash for UTF-8 string
int64_t hash[2];
int utf8_chars = rps_compute_cstr_two_64bits_hash(hash, "Hello 世界", -1);
// hash[0] and hash[1] contain the two hash values
// utf8_chars contains the number of UTF-8 characters processed
```

### String Creation
```cpp
// Create validated UTF-8 string
const Rps_String* str = Rps_String::make("Hello 世界", -1);
// Throws std::domain_error on invalid UTF-8

// String belongs to string class
Rps_ObjectRef str_class = str->compute_class(nullptr);
// Returns the string class object
```

### HTML Output
```cpp
std::stringstream html_out;
rps_output_utf8_html(html_out, "<script>alert('Hello & welcome!')</script>", -1, false);
// Output: <script>alert('Hello & welcome!')</script>
```

### JSON Output
```cpp
std::stringstream json_out;
rps_output_utf8_cjson(json_out, "Line 1\nLine 2\tTabbed", -1);
// Output: "Line 1\nLine 2\tTabbed"
```

### File Path Resolution
```cpp
// Find header file in include paths
std::string header_path = rps_glob_plain_file_path(
  "sys/stat.h",
  "/usr/include:/usr/local/include:/opt/include"
);
// Returns "/usr/include/sys/stat.h" if found and readable
```

## Performance Characteristics

### Hash Function Performance
- **Time Complexity**: O(n) where n is string length in bytes
- **UTF-8 Handling**: Processes multi-byte sequences correctly
- **Collision Resistance**: Dual 64-bit hashes reduce collision probability
- **Memory Usage**: No additional memory allocation

### String Operations
- **Creation Cost**: UTF-8 validation + memory allocation
- **Storage Efficiency**: Word-aligned allocation for GC compatibility
- **Output Performance**: Optimized fast paths for common characters

### File Globbing Performance
- **Search Strategy**: Linear search through directory list
- **Glob Efficiency**: Uses system `glob()` for pattern matching
- **Validation Overhead**: `stat()` and `access()` calls for each candidate
- **Caching**: No caching - resolves paths on each call

## Design Rationale

### UTF-8 First Approach
**Why strict UTF-8 validation?**
- **Unicode Compliance**: Ensures all text is properly encoded
- **Security**: Prevents malformed UTF-8 from causing issues
- **Interoperability**: Compatible with modern systems and libraries
- **Future-Proofing**: Supports international characters correctly

### Dual Hash Design
**Why two hash values instead of one?**
- **Collision Reduction**: Two independent hashes dramatically reduce false positives
- **Hash Table Support**: Enables double-hashing for collision resolution
- **Fingerprinting**: Provides better identification for string interning
- **Legacy Compatibility**: Maintains compatibility with existing hash usage

### HTML vs JSON Escaping
**Why separate functions for different output formats?**
- **Performance**: Format-specific optimizations
- **Correctness**: Different escaping rules for HTML vs JSON/C strings
- **Flexibility**: Allows format-specific behavior (e.g., nl2br in HTML)
- **Maintainability**: Clear separation of concerns

### File Path Globbing Design
**Why custom globbing instead of system PATH?**
- **Controlled Search**: Explicit directory lists instead of environment variables
- **Security**: Prevents PATH-based attacks
- **Predictability**: Deterministic search order
- **Flexibility**: Supports custom include paths for different contexts

This implementation provides robust scalar value handling with comprehensive UTF-8 support, forming the foundation for text processing and numeric operations within RefPerSys.