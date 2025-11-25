# User Preferences Implementation Analysis (`userpref_rps.cc`)

## Overview

The `userpref_rps.cc` file implements user preferences functionality for the Reflective Persistent System (RefPerSys), providing a configuration system that allows users to customize system behavior through INI-style configuration files. This module uses the INIReader library (inih) to parse configuration files and provides type-safe access to preference values with proper validation and error handling.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later (or LGPLv3+)
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2024-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// INI file parsing library
#include "INIReader.h"  // from benhoyt/inih or OSSystems/inih

// Git versioning information
extern "C" const char rps_userpref_gitid[];
extern "C" const char rps_userpref_date[];
extern "C" const char rps_userpref_shortgitid[];
extern "C" const char rps_userpref_timestamp[];
```

### Key Dependencies
- **refpersys.hh**: Core system definitions and Rps_MemoryFileTokenSource
- **INIReader**: INI file parsing library (libinih-dev package)
- **POSIX functions**: `access()`, `getenv()` for file operations

## Global State Management

### Static Variables
```cpp
static INIReader* rps_userpref_ird;                    // INI reader instance
static std::atomic_bool rps_userpref_is_parsed;        // Parse state flag
static Rps_MemoryFileTokenSource* rps_userpref_mts;    // Token source for file
```

**Purpose**: Maintains global state for user preferences with thread-safe parsing flag.

### Magic String Definition
```cpp
#define RPS_USER_PREFERENCE_MAGIC "*REFPERSYS_USER_PREFERENCES"
```

**Purpose**: Identifies the start of user preferences in configuration files.

## Preference File Loading

### File Setting Function
```cpp
void rps_set_user_preferences(char* path)
{
  // Validate file accessibility and main thread execution
  RPS_ASSERT(!access(path, R_OK));
  RPS_ASSERT(rps_is_main_thread());

  // Prevent multiple preference file settings
  if (rps_userpref_mts)
    RPS_FATALOUT("cannot set user preferences more than once");

  // Create memory file token source
  rps_userpref_mts = new Rps_MemoryFileTokenSource(path);

  // Scan for magic string marker
  while (true) {
    const char* clp = rps_userpref_mts->curcptr();
    if (!clp) break;
    if (!strncmp(clp, RPS_USER_PREFERENCE_MAGIC,
                 strlen(RPS_USER_PREFERENCE_MAGIC))) {
      break;  // Found magic string
    }
  }

  // Parse preferences and register cleanup
  rps_parse_user_preferences(rps_userpref_mts);
  atexit(rps_delete_user_preferences);
}
```

**Validation Process**:
1. **Accessibility**: Check read permission with `access()`
2. **Thread Safety**: Ensure main thread execution
3. **Single Setting**: Prevent multiple preference files
4. **Magic Detection**: Scan file for preference marker

### Magic String Scanning Algorithm
```cpp
// Read lines until magic string found or EOF
while (true) {
  const char* clp = rps_userpref_mts->curcptr();
  if (!clp) break;  // EOF reached

  if (!strncmp(clp, RPS_USER_PREFERENCE_MAGIC,
               strlen(RPS_USER_PREFERENCE_MAGIC))) {
    break;  // Magic string found
  }

  // Continue to next line
  rps_userpref_mts->get_line();
}
```

**Purpose**: Locate the start of user preferences within potentially larger files.

## Preference Parsing

### INI Parsing Function
```cpp
void rps_parse_user_preferences(Rps_MemoryFileTokenSource* mts)
{
  // Create INIReader from file content
  rps_userpref_ird = new INIReader(mts->toksrcmfil_line,
                                   mts->toksrcmfil_end - mts->toksrcmfil_line);

  // Ensure single parsing
  bool parsedonce = !rps_userpref_is_parsed.exchange(true);
  if (!parsedonce)
    RPS_FATALOUT("preferences parsed more than once");

  // Check for parse errors
  if (int pe = rps_userpref_ird->ParseError()) {
    RPS_WARNOUT("failed to parse user preferences: " << pe);
    return;
  }

  RPS_INFORMOUT("parsed user preferences from " << mts->path());
}
```

**Error Handling**:
- **Parse Errors**: Report line numbers and error codes
- **Multiple Parsing**: Prevent duplicate parsing attempts
- **Memory Management**: Proper INIReader lifecycle

### Parse State Checking
```cpp
bool rps_has_parsed_user_preferences(void)
{
  return rps_userpref_is_parsed.load();
}
```

**Purpose**: Allow other components to check if preferences have been loaded.

## Type-Safe Preference Access

### String Preferences
```cpp
std::string rps_userpref_get_string(const std::string& section,
                                   const std::string& name,
                                   const std::string& default_value)
{
  if (!rps_userpref_ird) return default_value;
  return rps_userpref_ird->GetString(section, name, default_value);
}
```

### Numeric Preferences
```cpp
long rps_userpref_get_long(const std::string& section,
                          const std::string& name, long default_value)
{
  if (!rps_userpref_ird) return default_value;
  return rps_userpref_ird->GetInteger(section, name, default_value);
}

double rps_userpref_get_double(const std::string& section,
                              const std::string& name, double default_value)
{
  if (!rps_userpref_ird) return default_value;
  return rps_userpref_ird->GetReal(section, name, default_value);
}
```

### Boolean Preferences
```cpp
bool rps_userpref_get_bool(const std::string& section,
                          const std::string& name, bool default_value)
{
  if (!rps_userpref_ird) return default_value;
  return rps_userpref_ird->GetBoolean(section, name, default_value);
}
```

**Common Pattern**: All access functions check for INIReader availability and return defaults when preferences are not loaded.

## C-String Interface Functions

### String Access with Found Flag
```cpp
const char* rps_userpref_find_dup_cstring(bool* pfound,
                                         const char* csection,
                                         const char* cname)
{
  if (!rps_userpref_ird || !rps_userpref_ird->HasValue(section, name)) {
    *pfound = false;
    return nullptr;
  }

  std::string val = rps_userpref_ird->GetString(section, name, "");
  const char* res = strdup(val.c_str());  // Duplicate for caller
  if (!res) {
    *pfound = false;
    return nullptr;
  }

  *pfound = true;
  return res;
}
```

**Memory Management**: Returns `strdup()` allocated strings that caller must free.

### Raw String Access (Caution)
```cpp
const char* rps_userpref_raw_cstring(const char* csection, const char* cname)
{
  if (!rps_userpref_ird || !rps_userpref_ird->HasValue(section, name))
    return nullptr;

  const std::string& val = rps_userpref_ird->GetString(section, name, "");
  return val.c_str();  // Reference to internal string - DANGEROUS
}
```

**Warning**: Returns reference to internal string that may become invalid. Marked as potentially problematic.

### Numeric Access with Found Flag
```cpp
long rps_userpref_find_clong(bool* pfound,
                            const char* csection, const char* cname)
{
  if (!rps_userpref_ird || !rps_userpref_ird->HasValue(section, name)) {
    *pfound = false;
    return 0L;
  }

  long res = rps_userpref_ird->GetInteger(section, name, 0L);
  *pfound = true;
  return res;
}
```

**Pattern**: Similar functions exist for `double` and `bool` types.

## Section and Value Queries

### Section Existence
```cpp
bool rps_userpref_has_section(const std::string& section)
{
  if (!rps_userpref_ird) return false;
  return rps_userpref_ird->HasSection(section);
}
```

### Value Existence
```cpp
bool rps_userpref_has_value(const std::string& section, const std::string& name)
{
  if (!rps_userpref_ird) return false;
  return rps_userpref_ird->HasValue(section, name);
}
```

**Purpose**: Allow checking for configuration existence without retrieving values.

## Default Preference File Handling

### Automatic Loading
```cpp
void rps_try_parsing_default_user_preferences(void)
{
  // Construct default path: $HOME/.config/refpersys_pref
  char* h = getenv("HOME");
  if (!h) RPS_FATALOUT("missing $HOME environment variable");

  char buf[rps_path_byte_size];
  int s = snprintf(buf, sizeof(buf)-2, "%s/.config/refpersys_pref", h);

  if (s > 0 && s < (int)sizeof(buf)-4) {
    if (!access(buf, R_OK)) {
      rps_set_user_preferences(buf);
    }
  }
}
```

**Default Location**: `$HOME/.config/refpersys_pref`

**Safety Checks**:
- **HOME Variable**: Ensure environment variable exists
- **Path Length**: Prevent buffer overflow
- **File Access**: Only load if file exists and is readable

## Cleanup and Resource Management

### Exit Handler
```cpp
static void rps_delete_user_preferences(void)
{
  if (rps_userpref_mts) {
    delete rps_userpref_mts;
    rps_userpref_mts = nullptr;
  }
  if (rps_userpref_ird) {
    delete rps_userpref_ird;
    rps_userpref_ird = nullptr;
  }
}
```

**Purpose**: Clean up resources at program exit using `atexit()` registration.

## Usage Examples

### Preference File Format
```ini
*REFPERSYS_USER_PREFERENCES

[display]
width = 1024
height = 768
fullscreen = true

[debug]
level = 2
log_file = /tmp/refpersys.log

[performance]
cache_size = 1000000
thread_count = 8
```

### Basic Usage
```cpp
// Load user preferences
rps_set_user_preferences("/path/to/user/prefs.ini");

// Access preferences with defaults
std::string log_file = rps_userpref_get_string("debug", "log_file", "/tmp/default.log");
long cache_size = rps_userpref_get_long("performance", "cache_size", 100000);
bool fullscreen = rps_userpref_get_bool("display", "fullscreen", false);

// Check for preference existence
if (rps_userpref_has_section("advanced")) {
  // Section exists, access advanced preferences
}

// C-style interface
bool found;
const char* log_path = rps_userpref_find_dup_cstring(&found, "debug", "log_file");
if (found) {
  // Use log_path (remember to free())
  free((void*)log_path);
}
```

### Default Preferences
```cpp
// Try to load default preferences automatically
rps_try_parsing_default_user_preferences();

// Check if preferences were loaded
if (rps_has_parsed_user_preferences()) {
  // Preferences available
} else {
  // Use system defaults
}
```

## Performance Characteristics

### Loading Performance
- **File Reading**: O(file size) for initial load
- **INI Parsing**: O(number of entries) using inih library
- **Memory Usage**: O(total configuration size)
- **Thread Safety**: Single load operation, atomic flag protection

### Access Performance
- **Value Retrieval**: O(1) average case with hash table lookup
- **Type Conversion**: Minimal overhead for numeric conversions
- **Memory Allocation**: `strdup()` for C-string interface
- **Existence Checks**: O(1) for section/value presence

### Memory Management
- **Static Allocation**: Global INIReader and token source
- **Exit Cleanup**: Proper resource deallocation via atexit
- **String Duplication**: C interface requires manual memory management
- **Reference Safety**: Raw string access may return dangling pointers

## Design Rationale

### INI File Format Choice
**Why INI format instead of JSON/XML/YAML?**
- **Simplicity**: Easy to read and edit manually
- **Ubiquity**: Widely supported format with existing tools
- **Performance**: Fast parsing with minimal dependencies
- **Compatibility**: Works well with existing configuration systems

### Magic String Requirement
**Why require magic string in preference files?**
- **File Type Detection**: Distinguish preference files from other text files
- **Embedding Support**: Allow preferences within larger files (scripts, etc.)
- **Version Control**: Enable future format changes
- **Safety**: Prevent accidental parsing of unrelated files

### Type-Safe Interface Design
**Why both std::string and C-string interfaces?**
- **C++ Integration**: Natural interface for modern C++ code
- **C Compatibility**: Support for existing C code and libraries
- **Performance**: Avoid string conversions where possible
- **Memory Control**: Explicit memory management for C interface

### Single Preference File Limitation
**Why only one preference file?**
- **Simplicity**: Avoid configuration conflicts and precedence issues
- **Predictability**: Clear source of configuration values
- **Debugging**: Easier to locate and modify settings
- **Performance**: Single parse operation

### Default Location Convention
**Why $HOME/.config/refpersys_pref?**
- **XDG Compliance**: Follows XDG Base Directory specification
- **User Isolation**: Per-user configuration without privilege issues
- **Convention**: Standard location for user-specific configuration
- **Backup Safety**: Location suitable for version control and backups

This implementation provides a robust, type-safe configuration system that allows users to customize RefPerSys behavior while maintaining thread safety and proper resource management.