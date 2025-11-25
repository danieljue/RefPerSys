# Utilities Implementation Analysis (`utilities_rps.cc`)

## Overview

The `utilities_rps.cc` file provides a comprehensive collection of utility functions for the Reflective Persistent System (RefPerSys), implementing system-level operations, debugging support, version information, and infrastructure management. This module serves as the foundation for many core system capabilities, including thread management, inter-process communication, file system operations, and debugging facilities.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later (or LGPLv3+)
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2019-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// GUI toolkit (optional)
#if RPS_WITH_FLTK
#include <FL/Fl.H>
#include <FL/platform.H>
// ... FLTK headers
#endif

// Libraries
#include "glib.h"
#include "libgccjit.h"

// Git versioning information
extern "C" const char rps_utilities_gitid[];
extern "C" const char rps_utilities_date[];
extern "C" const char rps_utilities_shortgitid[];
extern "C" const char rps_utilities_timestamp[];
```

### Key Dependencies
- **refpersys.hh**: Core system definitions and macros
- **FLTK**: Optional GUI toolkit for graphical interfaces
- **glib**: GNOME GLib library for utility functions
- **libgccjit**: GNU Compiler Collection JIT compilation
- **POSIX functions**: System calls for process and file management

## Version Information Functions

### Version Accessors
```cpp
int rps_get_major_version(void) { return RPS_MAJOR_VERSION_NUM; }
int rps_get_minor_version(void) { return RPS_MINOR_VERSION_NUM; }
```

**Purpose**: Provide programmatic access to version information defined in header files.

### Thread Management
```cpp
bool rps_is_main_thread(void)
{
  return pthread_self() == rps_main_thread_handle;
}
```

**Purpose**: Identify the main thread for thread-safety checks and main-thread-only operations.

### User Preferences Help Flag
```cpp
bool rps_want_user_preferences_help(void)
{
  return rps_flag_pref_help;
}
```

**Purpose**: Check if user preferences help was requested via command-line options.

## FIFO Communication Setup

### FIFO Prefix Management
```cpp
void rps_put_fifo_prefix(const char* pref)
{
  if (!rps_fifo_prefix.empty()) return;
  if (pref && pref[0]) rps_fifo_prefix = std::string(pref);
}

std::string rps_get_fifo_prefix(void) { return rps_fifo_prefix; }
```

**Purpose**: Set and retrieve the FIFO prefix for inter-process communication with GUI processes.

### GUI FIFO File Descriptors
```cpp
struct rps_fifo_fdpair_st rps_get_gui_fifo_fds(void)
{
  if (rps_gui_pid) return rps_fifo_pair;
  else return {-1, -1};
}

pid_t rps_get_gui_pid(void) { return rps_gui_pid; }
```

**Purpose**: Provide access to FIFO file descriptors for GUI communication.

### FIFO Creation
```cpp
void rps_do_create_fifos_from_prefix(void)
{
  // Create command and output FIFO files
  // Open file descriptors for communication
  // Register cleanup handler
}
```

**Algorithm**:
1. **Path Construction**: Build FIFO paths using prefix
2. **FIFO Creation**: Use `mkfifo()` to create named pipes
3. **File Opening**: Open FIFOs with appropriate permissions
4. **Cleanup Registration**: Register `atexit()` handler for FIFO removal

### FIFO Cleanup
```cpp
static void rps_remove_fifos(void)
{
  // Remove command and output FIFO files
  if (rps_is_fifo(cmdfifo)) remove(cmdfifo.c_str());
  if (rps_is_fifo(outfifo)) remove(outfifo.c_str());
}
```

**Purpose**: Clean up FIFO files at program exit.

## Directory and Host Information

### Home Directory Resolution
```cpp
const char* rps_homedir(void)
{
  static std::mutex homedirmtx;
  std::lock_guard<std::mutex> gu(homedirmtx);
  if (RPS_UNLIKELY(rps_bufpath_homedir[0] == (char)0)) {
    const char* rpshome = getenv("REFPERSYS_HOME");
    const char* home = getenv("HOME");
    const char* path = rpshome ? rpshome : home;
    if (!path) RPS_FATAL("no RefPerSys home");
    char* rp = realpath(path, nullptr);
    if (!rp) RPS_FATAL("realpath failed on RefPerSys home");
    // Validate path length and copy
    strncpy(rps_bufpath_homedir, rp, sizeof(rps_bufpath_homedir)-1);
  }
  return rps_bufpath_homedir;
}
```

**Priority Order**:
1. `$REFPERSYS_HOME` environment variable
2. `$HOME` environment variable
3. Fatal error if neither exists

### Load Directory Access
```cpp
const std::string& rps_get_loaddir(void) { return rps_my_load_dir; }
```

**Purpose**: Provide access to the configured load directory path.

### Hostname Caching
```cpp
const char* rps_hostname(void)
{
  static char hnambuf[80];
  if (RPS_UNLIKELY(!hnambuf[0])) gethostname(hnambuf, sizeof(hnambuf)-1);
  return hnambuf;
}
```

**Purpose**: Cache hostname to avoid repeated system calls.

## Extra Arguments Handling

### Argument Storage and Retrieval
```cpp
static std::map<std::string, std::string> rps_dict_extra_arg;

const char* rps_get_extra_arg(const char* name)
{
  if (!name) return nullptr;
  // Validate name format (alphanumeric + underscore)
  std::string goodstr{name};
  auto it = rps_dict_extra_arg.find(goodstr);
  return it != rps_dict_extra_arg.end() ? it->second.c_str() : nullptr;
}
```

**Validation Rules**:
- Name must start with alphabetic character
- Only alphanumeric characters and underscores allowed
- Empty or invalid names return nullptr

## Copyright Notice Generation

### GPLv3 Copyright Notice
```cpp
void rps_emit_gplv3_copyright_notice_AT(std::ostream& outs, ...)
{
  // Emit complete GPLv3 copyright notice with:
  // - SPDX license identifier
  // - Generation metadata (git ID, timestamp)
  // - Copyright years and owner
  // - Full license text
}
```

**Parameters**:
- **outs**: Output stream
- **fil/lin/fromfunc**: Source location information
- **path**: Generated file path
- **linprefix/linsuffix**: Line formatting
- **owner**: Copyright owner (defaults to team)
- **reason**: Generation reason

### LGPLv3 Copyright Notice
```cpp
void rps_emit_lgplv3_copyright_notice_AT(std::ostream& outs, ...)
{
  // Similar to GPLv3 but for Lesser GPL license
}
```

**Purpose**: Generate properly formatted copyright notices for generated files.

## Type Information and Version Display

### Type Information Printing
```cpp
void rps_print_types_info(void)
{
  // Print size and alignment information for core types
  // Uses X-macros to generate type information table
  #define EXPLAIN_TYPE(Ty) printf(TYPEFMT_rps " %5d %5d\n", #Ty, sizeof(Ty), alignof(Ty));
  // ... many type definitions
}
```

**Displayed Types**:
- Primitive types (int, double, char, bool, void*)
- Standard library types (string, vector, mutex, atomic)
- RefPerSys types (Rps_ObjectRef, Rps_Value, etc.)
- FLTK types (when available)

### Version Information Display
```cpp
void rps_show_version(void)
{
  // Display comprehensive version information:
  // - Program name and version
  // - Build information (git, timestamp, host)
  // - Library versions (glib, gccjit, jsoncpp)
  // - System information
  // - Source file version details
}
```

**Information Categories**:
- **Version Numbers**: Major/minor versions
- **Build Details**: Git ID, timestamp, hostname
- **Libraries**: GLib, GCCJIT, JSONCPP versions
- **System Info**: Hostname, PID, executable path
- **Source Files**: Version info for each compiled source file

### Source File Version Display
```cpp
void rps_show_version_handwritten_source_files(void)
{
  // Iterate through all source files
  // Use dlsym to find gitid and date symbols
  // Display formatted version information
}
```

**Algorithm**:
1. **File Iteration**: Loop through `rps_files` array
2. **Symbol Lookup**: Use `dlsym()` to find version symbols
3. **Validation**: Check symbol validity and format
4. **Display**: Format and output version information

## Time Routines

### Elapsed Time Functions
```cpp
double rps_elapsed_real_time(void)
{
  return rps_monotonic_real_time() - rps_start_monotonic_time;
}

double rps_get_start_wallclock_real_time()
{
  return rps_start_wallclock_real_time;
}
```

**Purpose**: Track elapsed real time and wall clock time since program start.

### Centisecond strftime
```cpp
char* rps_strftime_centiseconds(char* bfr, size_t len, const char* fmt, double tm)
{
  // Format time with centisecond precision
  // Replace ".__" in format string with fractional seconds
}
```

**Format Extension**: `"__.__"` in format string gets replaced with centiseconds (e.g., "23" for 0.23 seconds).

## Environment Management

### Environment Extension
```cpp
void rps_extend_env(void)
{
  // Set RefPerSys-specific environment variables:
  // REFPERSYS_PID, REFPERSYS_SHORTGITID, REFPERSYS_GITID,
  // REFPERSYS_TOPDIR, REFPERSYS_FIFO_PREFIX, REFPERSYS_RUN_NAME
}
```

**Environment Variables Set**:
- **REFPERSYS_PID**: Process ID
- **REFPERSYS_SHORTGITID**: Short git commit ID
- **REFPERSYS_GITID**: Full git commit ID
- **REFPERSYS_TOPDIR**: Top directory path
- **REFPERSYS_FIFO_PREFIX**: FIFO communication prefix
- **REFPERSYS_RUN_NAME**: Run name identifier

## File Modification Time Checking

### Build Freshness Check
```cpp
void rps_check_mtime_files(void)
{
  // Compare executable timestamp with source file timestamps
  // Warn if source files are newer than executable
  // Run make check if available
}
```

**Algorithm**:
1. **Executable Check**: Get executable modification time
2. **Source File Scan**: Check each source file in `rps_files`
3. **Comparison**: Warn if source files are newer
4. **Make Check**: Run `make -q objects` to verify build status

### Postponed File Removal
```cpp
void rps_postponed_remove_file(const std::string& path)
{
  // Add file to removal queue
  // Schedule removal via at command
}

void rps_schedule_files_postponed_removal(void)
{
  // Execute queued file removals using at command
}
```

**Purpose**: Safely remove temporary files after program exit using the `at` command.

## Program Arguments Handling

### Argument Parsing
```cpp
void rps_parse_program_arguments(int& argc, char** argv)
{
  // Initialize early system state
  // Parse command-line arguments using argp
  // Handle special options (--full-git, --short-git)
}
```

**Special Options**:
- **--full-git**: Print full git ID and exit
- **--short-git**: Print short git ID and exit
- **--debug**: Enable debug flags

### Argument Output
```cpp
void rps_output_program_arguments(std::ostream& out, int argc, const char* const* argv)
{
  // Format and output program arguments
  // Quote non-printable arguments appropriately
}
```

**Formatting Rules**:
- Plain arguments: output as-is
- Complex arguments: wrap in quotes or single quotes
- Special characters: escape as needed

## Root Object Management

### Root Object Set
```cpp
static std::set<Rps_ObjectRef> rps_object_root_set;
static std::mutex rps_object_root_mtx;
static std::unordered_map<Rps_Id, Rps_ObjectRef*, Rps_Id::Hasher> rps_object_global_root_hashtable;
```

**Purpose**: Maintain the set of garbage collection roots.

### Root Object Operations
```cpp
void rps_add_root_object(const Rps_ObjectRef ob)
void rps_remove_root_object(const Rps_ObjectRef ob)
bool rps_is_root_object(const Rps_ObjectRef ob)
std::set<Rps_ObjectRef> rps_set_root_objects(void)
unsigned rps_nb_root_objects(void)
```

**Purpose**: Manage the global set of objects that are always reachable during garbage collection.

### Root Initialization
```cpp
void rps_initialize_roots_after_loading(Rps_Loader* ld)
{
  // Initialize root object hashtable from generated headers
  #include "generated/rps-roots.hh"
}
```

**Purpose**: Set up root object references after system loading.

## Constant Object Management

### Constant Addition
```cpp
void rps_add_constant_object(Rps_CallFrame* callframe, const Rps_ObjectRef argob)
{
  // Add object to the system's constant set
  // Update RefPerSys_system object's constant attribute
}
```

**Protection**: Prevents addition of core sacred root objects as constants.

### Constant Removal
```cpp
void rps_remove_constant_object(Rps_CallFrame* callframe, const Rps_ObjectRef argobconst)
{
  // TODO: Implement constant removal
  RPS_FATALOUT("unimplemented");
}
```

**Status**: Currently unimplemented.

## Symbol Initialization

### Symbol Table Setup
```cpp
void rps_initialize_symbols_after_loading(Rps_Loader* ld)
{
  // Initialize symbol hashtable from generated names
  #include "generated/rps-names.hh"
}
```

**Purpose**: Set up global symbol references after loading.

## Debugging Support

### Debug Flag Management
```cpp
bool rps_is_set_debug(const std::string& curlev)
Rps_Debug rps_debug_of_string(const std::string& deblev)
bool rps_set_debug_flag(const std::string& curlev)
void rps_set_debug(const std::string& deblev)
void rps_add_debug_cstr(const char* d)
const char* rps_cstr_of_debug(Rps_Debug dbglev)
void rps_output_debug_flags(std::ostream& out, unsigned flags)
```

**Features**:
- **Flag Setting**: Enable/disable debug categories
- **String Conversion**: Convert between strings and debug enums
- **Comma-Separated**: Support multiple flags in one string
- **Help Display**: Show available debug options

## Event Loop Utilities

### After-Event-Loop Functions
```cpp
static std::recursive_mutex rps_aftevntloop_mtx;
static std::vector<std::function<void(void)>> rps_aftevntloop_vec;

void rps_register_after_event_loop(std::function<void(void)> f)
{
  std::lock_guard<std::recursive_mutex> gu(rps_aftevntloop_mtx);
  rps_aftevntloop_vec.push_back(f);
}

void rps_run_after_event_loop(void)
{
  std::lock_guard<std::recursive_mutex> gu(rps_aftevntloop_mtx);
  for (auto f : rps_aftevntloop_vec) f();
  rps_aftevntloop_vec.clear();
}
```

**Purpose**: Register functions to run after the event loop completes.

## String Formatting Utilities

### Printf-Style String Formatting
```cpp
std::string rps_stringprintf(const char* fmt, ...)
{
  // Variable argument string formatting
  // Handle buffer sizing automatically
}
```

**Features**:
- **Buffer Management**: Automatic buffer sizing
- **Memory Safety**: Proper allocation and cleanup
- **Format Support**: Standard printf format specifiers

### Vector String Output
```cpp
void rps_output_vector_string(std::ostream& out, const std::vector<std::string>& vecstr, int indent)
{
  // Formatted output of string vectors
  // TODO: Implement indentation support
}
```

**Purpose**: Pretty-print string vectors with indexing and quoting.

## Usage Examples

### Version Information
```cpp
// Get version numbers
int major = rps_get_major_version();  // e.g., 0
int minor = rps_get_minor_version();  // e.g., 6

// Display full version information
rps_show_version();
```

### FIFO Communication Setup
```cpp
// Set FIFO prefix for GUI communication
rps_put_fifo_prefix("/tmp/rps_gui_");

// Create FIFO files
rps_do_create_fifos_from_prefix();

// Get file descriptors for communication
auto fds = rps_get_gui_fifo_fds();
int cmd_fd = fds.fifo_ui_wcmd;    // Write commands to GUI
int out_fd = fds.fifo_ui_rout;    // Read output from GUI
```

### Home Directory Access
```cpp
// Get RefPerSys home directory
const char* home = rps_homedir();  // Uses $REFPERSYS_HOME or $HOME

// Get load directory
const std::string& load_dir = rps_get_loaddir();
```

### Copyright Notice Generation
```cpp
// Generate GPLv3 copyright notice for generated file
std::ofstream outfile("generated_file.cc");
rps_emit_gplv3_copyright_notice_AT(outfile,
                                   __FILE__, __LINE__, __PRETTY_FUNCTION__,
                                   "generated_file.cc", "// ", "\n",
                                   "My Organization", "Code generation");
```

### Debug Flag Management
```cpp
// Enable multiple debug flags
rps_set_debug("REPL,DUMP,LOAD");

// Check if debug flag is set
if (rps_is_set_debug("REPL")) {
  // REPL debugging is enabled
}

// Get debug flags as string
std::ostringstream out;
rps_output_debug_flags(out, 0);  // Current flags
```

### Environment Extension
```cpp
// Extend environment with RefPerSys variables
rps_extend_env();
// Now available: REFPERSYS_PID, REFPERSYS_GITID, etc.
```

### Postponed File Removal
```cpp
// Schedule file for removal 5 minutes after exit
rps_postponed_remove_file("/tmp/temp_file.txt");
// File will be removed by: rm -f /tmp/temp_file.txt
```

This implementation provides essential utility functions that support the core operations of RefPerSys, from basic system information to complex inter-process communication and debugging facilities.