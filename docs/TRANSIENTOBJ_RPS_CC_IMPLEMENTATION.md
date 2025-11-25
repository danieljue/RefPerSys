# Transient Objects Implementation Analysis (`transientobj_rps.cc`)

## Overview

The `transientobj_rps.cc` file implements transient object payloads for the Reflective Persistent System (RefPerSys), providing runtime system interaction capabilities that are not persisted. This module includes payloads for Unix process management, C++ stream handling, and popen-ed file operations, enabling RefPerSys to interact with external processes and I/O streams while maintaining thread safety and proper resource management.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2023-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// Requires GNU libstdc++
#if !defined(__GLIBCXX__) && !defined(__GLIBCPP__)
#error need GNU libstdc++
#endif

// Git versioning information
extern "C" const char rps_transientobj_gitid[];
extern "C" const char rps_transientobj_date[];
extern "C" const char rps_transientobj_shortgitid[];
extern "C" const char rps_transientobj_timestamp[];
```

### Key Dependencies
- **refpersys.hh**: Core system definitions and Rps_Payload base class
- **GNU libstdc++**: Required for C++ standard library features
- **POSIX functions**: `prlimit`, `fork`, `pipe`, `popen` for system interactions
- **Threading**: `std::recursive_mutex`, `std::mutex` for thread safety

## Rps_PayloadUnixProcess Class

### Process Payload Structure
```cpp
class Rps_PayloadUnixProcess : public Rps_Payload {
    std::atomic<pid_t> _unixproc_pid;                    // Process ID
    std::string _unixproc_exe;                          // Executable path
    std::vector<std::string> _unixproc_argv;            // Command arguments
    Rps_ClosureValue _unixproc_closure;                 // Termination handler
    Rps_ClosureValue _unixproc_inputclos;               // Input handler
    Rps_ClosureValue _unixproc_outputclos;              // Output handler
    int _unixproc_pipeinputfd;                          // Input pipe FD
    int _unixproc_pipeoutputfd;                         // Output pipe FD
    // Resource limits...
    std::atomic<unsigned> _unixproc_cpu_time_limit;
    std::atomic<unsigned> _unixproc_elapsed_time_limit;
    std::atomic<time_t> _unixproc_start_time;
    std::atomic<unsigned> _unixproc_as_mb_limit;
    std::atomic<unsigned> _unixproc_fsize_mb_limit;
    std::atomic<unsigned> _unixproc_core_mb_limit;
    std::atomic<bool> _unixproc_forbid_core;
    std::atomic<unsigned> _unixproc_nofile_limit;
};
```

**Purpose**: Manages Unix process lifecycle with resource limits and I/O handling.

### Constructor and Initialization
```cpp
Rps_PayloadUnixProcess::Rps_PayloadUnixProcess(Rps_ObjectZone* owner)
  : Rps_Payload(Rps_Type::PaylUnixProcess, owner),
    _unixproc_pid(0),
    _unixproc_exe(),
    _unixproc_argv(),
    // ... initialize all fields to defaults
{ }
```

**Note**: Loading constructor throws fatal error - processes cannot be deserialized.

### Process Creation
```cpp
Rps_ObjectRef Rps_PayloadUnixProcess::make_dormant_unix_process_object(
    Rps_CallFrame* callerframe, const std::string& exec)
{
  // Resolve executable path using PATH or absolute path
  // Create unix_process class object with payload
  // Initialize executable and arguments
}
```

**Path Resolution Algorithm**:
1. **Absolute Path**: Use `realpath()` to canonicalize
2. **PATH Search**: Iterate through PATH directories to find executable
3. **Validation**: Check execute permission with `access()`

### Resource Limit Management

#### Core Dump Control
```cpp
void forbid_core_dump(void)
{
  std::lock_guard<std::recursive_mutex> gu(*owner()->objmtxptr());
  _unixproc_forbid_core.store(true);
  if (pid > 0) {
    struct rlimit newlim = {.rlim_cur=0, .rlim_max=RLIM_INFINITY};
    prlimit(pid, RLIMIT_CORE, &newlim, &oldlim);
  }
}
```

**Purpose**: Disable core dump generation for security or resource reasons.

#### Memory Limits
```cpp
unsigned address_space_megabytes_limit(unsigned newlimit)
{
  // Set RLIMIT_AS using prlimit()
  // Convert megabytes to bytes (<<20)
  // Return previous limit
}
```

**Resource Types Managed**:
- **Address Space**: `RLIMIT_AS` - total virtual memory
- **File Size**: `RLIMIT_FSIZE` - maximum file size
- **Core Size**: `RLIMIT_CORE` - core dump size
- **Open Files**: `RLIMIT_NOFILE` - maximum file descriptors

### Process Lifecycle Management

#### Process Arguments
```cpp
void add_process_argument(const std::string& arg)
{
  std::lock_guard<std::recursive_mutex> gu(*owner()->objmtxptr());
  _unixproc_argv.push_back(arg);
}
```

**Purpose**: Build command-line arguments for process execution.

#### Closure Management
```cpp
void put_process_closure(Rps_ClosureValue closv)
void put_input_closure(Rps_ClosureValue closv)
void put_output_closure(Rps_ClosureValue closv)
```

**Closure Types**:
- **Process Closure**: Called when process terminates
- **Input Closure**: Handles stdin requirements
- **Output Closure**: Processes stdout data

#### Process Execution
```cpp
void start_process(Rps_CallFrame* callframe)
{
  // Add to runnable queue
  // Postpone child process handling
  // Integration with eventloop_rps.cc
}
```

**Execution Flow**:
1. **Queue Addition**: Add to `queue_of_runnable_processes`
2. **Event Integration**: Call `rps_postpone_child_process()`
3. **Fork Handling**: Managed by event loop code

### Global Process Management
```cpp
static std::set<Rps_PayloadUnixProcess*> set_of_runnable_processes;
static std::mutex mtx_of_runnable_processes;
static std::deque<Rps_PayloadUnixProcess*> queue_of_runnable_processes;
```

**Purpose**: Track active processes for garbage collection and iteration.

#### Active Process Iteration
```cpp
void do_on_active_process_queue(
    std::function<void(Rps_ObjectRef, Rps_CallFrame*, void*)> fun,
    Rps_CallFrame* callframe, void* client_data)
{
  std::lock_guard<std::mutex> gu(mtx_of_runnable_processes);
  for (auto* paylup : queue_of_runnable_processes) {
    Rps_ObjectRef obown = paylup->owner();
    fun(obown, callframe, client_data);
  }
}
```

## Rps_PayloadCppStream Class

### Stream Payload Structure
```cpp
class Rps_PayloadCppStream : public Rps_Payload {
    Rps_KindStream _kind_stream;        // Stream type
    union {
        void* _ptr_stream;
        std::ostream* _out_stream;
        std::istream* _in_stream;
        std::iostream* _inout_stream;
    };
    int _ix_stream;                     // Registration index
    const int _ix_magic;                // Magic number for validation
};
```

**Purpose**: Manages C++ stream objects with registration and POSIX file descriptor access.

### Stream Registration System
```cpp
static std::recursive_mutex _cppstream_mtx;
static std::vector<Rps_PayloadCppStream*> _cppstream_vector;
```

**Registration Process**:
1. **Index Allocation**: Use `std::ios_base::xalloc()` for unique index
2. **Vector Management**: Store payload pointers in global vector
3. **Thread Safety**: Protected by recursive mutex

#### Stream Registration
```cpp
int register_cpp_stream(void)
{
  std::lock_guard<std::recursive_mutex> _gu_(_cppstream_mtx);
  int uniqix = std::ios_base::xalloc();
  _cppstream_vector.reserve(rps_prime_above(5*uniqix+3));
  _cppstream_vector[uniqix] = this;
  _ix_stream = uniqix;
  return uniqix;
}
```

#### POSIX File Descriptor Access
```cpp
int posix_fd(void)
{
  // Extract native handle from fstream objects
#if __cpplib_fstream_native_handle_type
  std::ifstream* fs = dynamic_cast<std::ifstream*>(_in_stream);
  if (fs) return fs->native_handle();
#endif
  return -1;
}
```

**Platform Dependency**: Requires C++23 `native_handle()` support or fallback implementation.

### Stream Constructors
```cpp
Rps_PayloadCppStream::Rps_PayloadCppStream(Rps_ObjectZone* owner, std::ostream& output)
  : Rps_Payload(Rps_Type::PaylCppStream, owner),
    _kind_stream(rps_output_stream),
    _out_stream(&output),
    _ix_stream(-1),
    _ix_magic(_ix_magicnum_)
{ }
```

**Stream Types Supported**:
- **Output Stream**: `std::ostream` for writing
- **Input Stream**: `std::istream` for reading
- **Bidirectional**: `std::iostream` for both operations

## Rps_PayloadPopenedFile Class

### Popen Payload Structure
```cpp
class Rps_PayloadPopenedFile : public Rps_Payload {
    const std::string _popened_cmd;      // Command string
    const bool _popened_to_read;         // Read/write mode
    std::atomic<FILE*> _popened_file;    // FILE pointer
};
```

**Purpose**: Manages popen-ed processes with command execution and I/O handling.

### Constructor
```cpp
Rps_PayloadPopenedFile::Rps_PayloadPopenedFile(
    Rps_ObjectZone* owner, const std::string command, bool reading)
  : Rps_Payload(Rps_Type::PaylPopenedFile, owner),
    _popened_cmd(command),
    _popened_to_read(reading),
    _popened_file(nullptr)
{ }
```

**Note**: Loading constructor throws fatal error - popen files cannot be deserialized.

## Transient Payload Characteristics

### Persistence Behavior
```cpp
void dump_scan(Rps_Dumper* du) const { /* do nothing */ }
void dump_json_content(Rps_Dumper* du, Json::Value&) const { /* do nothing */ }
bool is_erasable(void) const { return false; }
```

**Transient Properties**:
- **No Persistence**: Not saved during system dumps
- **Not Erasable**: Cannot be garbage collected for erasure
- **Runtime Only**: Exist only during system execution

### Garbage Collection Integration
```cpp
void gc_mark(Rps_GarbageCollector& gc) const
{
  // Mark closures and referenced objects
  if (_unixproc_closure) {
    _unixproc_closure.gc_mark(gc, 1);
  }
}
```

**GC Behavior**:
- **Closure Marking**: Process closures are marked for GC
- **Object References**: Owner objects are marked during active process GC
- **No Self-Marking**: Transient payloads don't mark themselves

## Thread Safety

### Mutex Usage Patterns
```cpp
// Object-level mutex for payload operations
std::lock_guard<std::recursive_mutex> gu(*owner()->objmtxptr());

// Global mutex for process management
std::lock_guard<std::mutex> gu(mtx_of_runnable_processes);

// Stream registration mutex
std::lock_guard<std::recursive_mutex> _gu_(_cppstream_mtx);
```

**Thread Safety Levels**:
- **Object Level**: Protects individual payload state
- **Global Level**: Protects shared process/stream collections
- **Atomic Operations**: PID and limit fields use atomic operations

## Usage Examples

### Unix Process Management
```cpp
// Create a dormant process object
Rps_ObjectRef proc = Rps_PayloadUnixProcess::make_dormant_unix_process_object(
    callframe, "/bin/ls");

// Add arguments
auto payload = proc->get_dynamic_payload<Rps_PayloadUnixProcess>();
payload->add_process_argument("-l");
payload->add_process_argument("/tmp");

// Set resource limits
payload->address_space_megabytes_limit(100);  // 100MB limit
payload->core_megabytes_limit(0);             // No core dumps

// Set closures
payload->put_process_closure(termination_handler);
payload->put_output_closure(output_handler);

// Start process
payload->start_process(callframe);
```

### Stream Management
```cpp
// Create output stream payload
std::ofstream outfile("/tmp/output.txt");
Rps_ObjectRef stream_obj = Rps_ObjectRef::make_object(callframe, stream_class);
auto payload = stream_obj->put_new_plain_payload<Rps_PayloadCppStream>(
    stream_obj.optr(), outfile);

// Register stream
int stream_ix = payload->register_cpp_stream();

// Get POSIX file descriptor
int fd = payload->posix_fd();
```

### Popen File Management
```cpp
// Create popen payload for reading
Rps_ObjectRef popen_obj = Rps_ObjectRef::make_object(callframe, popen_class);
auto payload = popen_obj->put_new_plain_payload<Rps_PayloadPopenedFile>(
    popen_obj.optr(), "ls -l /tmp", true);  // Read mode

// Access FILE pointer
FILE* fp = payload->get_popened_file();
```

## Performance Characteristics

### Process Management
- **Creation**: O(PATH length) for executable resolution
- **Limits**: O(1) with prlimit() system calls
- **Queue Operations**: O(1) for addition, O(n) for iteration
- **Memory**: O(argument count) for argument storage

### Stream Management
- **Registration**: O(1) amortized with vector growth
- **FD Access**: O(1) with dynamic_cast overhead
- **Memory**: O(1) per stream object
- **Thread Safety**: Mutex-protected operations

### Transient Behavior
- **GC Overhead**: Minimal - only closure marking
- **Persistence**: Zero overhead - not serialized
- **Memory**: Efficient storage without persistence metadata

## Design Rationale

### Transient Payload Design
**Why separate transient payloads from persistent ones?**
- **Security**: Sensitive runtime objects shouldn't be persisted
- **Performance**: Skip serialization overhead for temporary objects
- **Correctness**: Runtime state may not be serializable
- **Resource Management**: Different lifecycle management

### Process Management Architecture
**Why complex process management instead of simple fork/exec?**
- **Resource Control**: Fine-grained limits on child processes
- **I/O Handling**: Asynchronous input/output with closures
- **Lifecycle Tracking**: Monitor process state and termination
- **Integration**: Coordinate with event loop and agenda system

### Stream Registration System
**Why global stream registration?**
- **Cleanup**: Ensure streams are properly unregistered on destruction
- **Resource Tracking**: Global view of active streams
- **FD Access**: Enable POSIX file descriptor retrieval
- **Safety**: Prevent use-after-free of stream objects

### Popen vs Process Payloads
**Why both popen and process payloads?**
- **Simplicity**: Popen for simple command execution
- **Control**: Process payloads for complex lifecycle management
- **Integration**: Different integration points with system
- **Use Cases**: Popen for scripting, processes for long-running tasks

This implementation provides comprehensive transient object support essential for RefPerSys's interaction with external processes and I/O streams while maintaining proper resource management, thread safety, and garbage collection integration.