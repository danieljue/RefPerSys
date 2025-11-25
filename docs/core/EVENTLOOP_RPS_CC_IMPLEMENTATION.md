# Event Loop and JSONRPC Implementation Analysis (`eventloop_rps.cc`)

## Overview

The `eventloop_rps.cc` file implements the core **event-driven execution framework** and **JSONRPC communication system** for the Reflective Persistent System (RefPerSys), providing asynchronous I/O handling, signal processing, and inter-process communication with external GUI applications. This file serves as the central nervous system for RefPerSys's interaction with the external world.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2022-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// Git versioning information
extern "C" const char rps_eventloop_gitid[];
extern "C" const char rps_eventloop_date[];
extern "C" const char rps_eventloop_shortgitid[];
extern "C" const char rps_eventloop_timestamp[];
```

### Key Dependencies
- `refpersys.hh`: Main header containing core system definitions
- **System libraries**: `poll.h`, `signal.h`, `sys/signalfd.h`, `sys/timerfd.h`
- **POSIX APIs**: `pipe2()`, `signalfd()`, `timerfd_create()`, `poll()`
- Standard C++ threading and synchronization primitives

## Core Data Structures

### Event Loop Data Structure
```cpp
struct event_loop_data_st {
    unsigned eld_magic;                           // Magic number for validation
    int eld_polldelaymillisec;                    // Poll timeout in milliseconds
    unsigned eld_lastix;                          // Last used file descriptor index
    std::recursive_mutex eld_mtx;                 // Thread safety mutex

    double eld_startelapsedtime;                  // Timing information
    double eld_startcputime;

    // File descriptor management (up to RPS_MAXPOLL_FD = 128)
    Rps_EventHandler_sigt* eld_handlarr[RPS_MAXPOLL_FD+1]; // Handler functions
    const char* eld_explarr[RPS_MAXPOLL_FD+1];             // Explanations
    struct pollfd eld_pollarr[RPS_MAXPOLL_FD+1];           // Poll structures
    void* eld_datarr[RPS_MAXPOLL_FD+1];                    // User data

    // Special file descriptors
    int eld_sigfd;                                // signalfd for signal handling
    int eld_timfd;                                // timerfd for timers
    int eld_selfpipereadfd;                       // Self-pipe read end
    int eld_selfpipewritefd;                      // Self-pipe write end

    // Self-pipe communication
    std::deque<unsigned char> eld_selfpipefifo;   // Message queue

    // State management
    std::atomic<bool> eld_eventloopisactive;      // Active flag
    std::atomic<long> eld_nbloops;                // Loop counter

    // Extensibility
    std::vector<std::function<void(struct pollfd*, int& npoll, Rps_CallFrame*)>>
        eld_prepollvect;                          // Pre-poll hooks
};
```

### Self-Pipe Communication Codes
```cpp
enum self_pipe_code_en {
    SelfPipe__NONE = 0,
    SelfPipe_Dump = 'D',        // Trigger persistence dump
    SelfPipe_GarbColl = 'G',    // Trigger garbage collection
    SelfPipe_Process = 'P',     // Handle process events
    SelfPipe_Quit = 'Q',        // Quit event loop
    SelfPipe_Exit = 'X',        // Exit with dump
};
```

### Global State Variables
```cpp
std::atomic<bool> rps_stop_event_loop_flag;       // Global stop flag
int rps_poll_delay_millisec;                      // Poll delay configuration
struct event_loop_data_st rps_eventloopdata;      // Global event loop data
```

## Event Loop Initialization

### Core Initialization Function
```cpp
void rps_initialize_event_loop(void)
```
**Purpose**: Sets up the complete event loop infrastructure
- Validates single-threaded, main-thread execution
- Initializes event loop data structure with magic number
- Sets up poll delay based on debug flags
- Calls specialized initialization functions

**Initialization Sequence**:
1. **Self-pipe setup**: `rps_initialize_pipe_to_self_in_event_loop()`
2. **Signal handling**: `rps_initialize_signalfd_in_event_loop()`
3. **Timer setup**: `rps_initialize_timerfd_in_event_loop()`
4. **JSONRPC initialization**: `rps_initialize_jsonfifo_in_event_loop()` (if FIFO prefix set)

### Self-Pipe Initialization
```cpp
void rps_initialize_pipe_to_self_in_event_loop(void)
```
**Purpose**: Creates the self-pipe for thread-safe inter-thread communication
- Uses `pipe2()` with `O_CLOEXEC` for close-on-exec
- Registers both read and write ends as event handlers
- Enables asynchronous command execution from any thread

**Self-Pipe Architecture**:
- **Read end**: Registered for `POLLIN` events, handled by `rps_self_pipe_read_handler`
- **Write end**: Registered for `POLLOUT` events, handled by `rps_self_pipe_write_handler`
- **FIFO queue**: `eld_selfpipefifo` buffers messages between threads

### Signal Handling Initialization
```cpp
void rps_initialize_signalfd_in_event_loop(void)
```
**Purpose**: Sets up signal handling using Linux signalfd mechanism
- Creates signalfd for specified signals: `SIGCHLD`, `SIGINT`, `SIGTERM`, `SIGXCPU`, `SIGALRM`, `SIGVTALRM`
- Registers signal file descriptor for `POLLIN` events
- Enables reliable, queue-based signal delivery

### Timer Initialization
```cpp
void rps_initialize_timerfd_in_event_loop(void)
```
**Purpose**: Sets up timer handling using Linux timerfd mechanism
- Creates timerfd with `CLOCK_REALTIME` and `TFD_CLOEXEC`
- Registers timer file descriptor for `POLLIN` events
- **Status**: Currently incomplete (marked with `#warning`)

### JSONRPC Initialization
```cpp
void rps_initialize_jsonfifo_in_event_loop(void)
void rps_jsonrpc_initialize(void)
```
**Purpose**: Sets up JSONRPC communication with external GUI processes
- Validates FIFO file descriptors from `rps_get_gui_fifo_fds()`
- **Status**: Both functions are incomplete with extensive TODOs
- Intended for bidirectional JSON communication with GUI applications

## File Descriptor Management

### Handler Registration
```cpp
void rps_event_loop_add_input_fd_handler(int fd, Rps_EventHandler_sigt* f, const char* explanation, void* data)
void rps_event_loop_add_output_fd_handler(int fd, Rps_EventHandler_sigt* f, const char* explanation, void* data)
```
**Purpose**: Registers file descriptors for event monitoring
- Allocates slots in the event loop data arrays
- Sets appropriate poll events (`POLLIN` or `POLLOUT`)
- Stores handler function, explanation, and user data
- Supports FLTK integration when GUI is enabled

### Handler Removal
```cpp
void rps_event_loop_remove_input_fd_handler(int fd)
void rps_event_loop_remove_output_fd_handler(int fd)
```
**Purpose**: Removes file descriptors from event monitoring
- Rebuilds arrays without the specified file descriptor
- Maintains array continuity and proper indexing
- Updates FLTK event handling when applicable

### Handler Inspection
```cpp
bool rps_event_loop_get_entry(int ix, Rps_EventHandler_sigt** pfun, struct pollfd* po, const char** pexpl, void** pdata)
```
**Purpose**: Provides read-only access to event handler information
- Validates index bounds
- Returns handler function, poll structure, explanation, and user data
- Thread-safe with mutex protection

## Self-Pipe Communication System

### Writing to Self-Pipe
```cpp
void rps_self_pipe_write_byte(unsigned char b)
```
**Purpose**: Queues a byte for delivery to the event loop
- Thread-safe enqueue operation
- Used for asynchronous command execution
- Triggers poll wake-up for immediate processing

### Self-Pipe Read Handler
```cpp
void rps_self_pipe_read_handler(Rps_CallFrame* cf, int fd, void* data)
```
**Purpose**: Processes incoming self-pipe messages
- Reads bytes from the pipe
- Dispatches to `handle_self_pipe_byte_rps()` for each byte
- Handles dump, garbage collection, quit, and process commands

### Self-Pipe Write Handler
```cpp
void rps_self_pipe_write_handler(Rps_CallFrame* cf, int fd, void* data)
```
**Purpose**: Flushes queued bytes to the self-pipe
- Writes pending bytes from the FIFO queue
- Handles partial writes and error conditions
- Maintains queue integrity during concurrent access

### Command Dispatch
```cpp
void handle_self_pipe_byte_rps(unsigned char b)
```
**Purpose**: Executes commands received via self-pipe
- **'D' (Dump)**: Triggers persistence dump
- **'G' (GarbColl)**: Triggers garbage collection
- **'P' (Process)**: Handles process events (incomplete)
- **'Q' (Quit)**: Stops event loop
- **'X' (Exit)**: Dumps and exits

### Public Command Interface
```cpp
void rps_postpone_dump(void)
void rps_postpone_garbage_collection(void)
void rps_postpone_quit(void)
void rps_postpone_exit_with_dump(void)
void rps_postpone_child_process(void)
```
**Purpose**: Thread-safe command posting from any thread
- All functions use `rps_self_pipe_write_byte()` for delivery
- Enable asynchronous execution of critical operations

## Signal Handling

### Signal File Descriptor Handler
```cpp
void rps_sigfd_read_handler(Rps_CallFrame* cf, int fd, void* data)
```
**Purpose**: Processes signals delivered via signalfd
- Reads `signalfd_siginfo` structures
- Handles different signal types with appropriate actions

**Signal Processing**:
- **SIGTERM**: Schedules final dump and stops event loop
- **SIGINT**: Stops event loop immediately
- **SIGQUIT**: Stops event loop immediately
- **SIGCHLD**: Logs child process termination (incomplete)
- **SIGUSR1**: Should schedule temporary dump (incomplete)

### Timer Handler
```cpp
void rps_timerfd_read_handler(Rps_CallFrame* cf, int fd, void* data)
```
**Status**: Unimplemented stub with fatal error
- Intended for timer event processing
- Currently terminates with error message

## Event Loop Execution

### Main Event Loop
```cpp
void rps_event_loop(void)
```
**Purpose**: Core event processing loop using `poll(2)`
- Validates single-threaded, main-thread execution
- Sets up local handler arrays and poll structures
- Processes events in a continuous loop until stopped

**Loop Structure**:
1. **Setup**: Initialize poll arrays and handlers
2. **Pre-poll**: Execute registered prepoll functions
3. **Poll**: Wait for events with timeout
4. **Event Processing**: Dispatch events to handlers
5. **Timeout Handling**: Process poll timeouts
6. **Agenda Integration**: Check agenda timeouts

### Poll Array Management
The event loop dynamically builds `pollfd` arrays for each iteration:
- **JSONRPC FIFOs**: Command and response pipes to GUI
- **Timer FD**: Timer events (when available)
- **Self-pipe**: Read and write ends for thread communication
- **FLTK Integration**: GUI event handling (when enabled)

### Pre-Poll Hooks
```cpp
int rps_register_event_loop_prepoller(std::function<void(struct pollfd*, int& npoll, Rps_CallFrame*)> fun)
void rps_unregister_event_loop_prepoller(int rank)
```
**Purpose**: Extensibility mechanism for event loop customization
- Allows registration of functions called before each `poll()`
- Functions can modify poll arrays and add custom file descriptors
- Supports dynamic event source addition

## JSONRPC Communication System

### JSONRPC Buffers
```cpp
static std::mutex rps_jsonrpc_mtx;               // Protects buffers
static std::stringbuf rps_jsonrpc_cmdbuf;         // Commands to GUI
static std::stringbuf rps_jsonrpc_rspbuf;         // Responses from GUI
static std::stringstream rps_jsonrpc_rspstream;   // Response processing
```

### JSONRPC Initialization
```cpp
void rps_jsonrpc_initialize(void)
```
**Status**: Incomplete implementation
- Validates FIFO file descriptors
- **TODO**: Document JSONRPC protocol
- **TODO**: Implement version request handling
- **TODO**: Set up bidirectional communication

### JSONRPC Event Handlers
**Command Output Handler**: Writes JSONRPC commands to GUI process
- **Status**: Fatal error - "missing code"
- **TODO**: Implement command serialization and transmission

**Response Input Handler**: Reads JSONRPC responses from GUI process
- **Status**: Partial implementation with fatal errors
- **TODO**: Implement JSON message parsing and processing
- **TODO**: Handle message framing (double newline/formfeed termination)

## Event Loop Control

### Stop Functions
```cpp
void rps_do_stop_event_loop(void)
```
**Purpose**: Signals event loop termination
- Sets global `rps_stop_event_loop_flag`
- Integrates with FLTK if enabled
- Thread-safe atomic operation

### Status Functions
```cpp
bool rps_is_active_event_loop(void)
bool rps_event_loop_is_running(void)
long rps_event_loop_counter(void)
```
**Purpose**: Query event loop state
- `rps_is_active_event_loop()`: Checks if loop completed successfully
- `rps_event_loop_is_running()`: Checks if loop is currently executing
- `rps_event_loop_counter()`: Returns loop iteration count or -1 if stopped

## Utility Functions

### FIFO Detection
```cpp
bool rps_is_fifo(std::string path)
```
**Purpose**: Determines if a filesystem path is a FIFO (named pipe)
- Uses `stat()` to check file type
- Returns true for `S_IFIFO` file mode

### Event Loop Explanation
```cpp
std::string rps_eventloop_explstring(void)
```
**Status**: Incomplete implementation
- Intended for debugging and status display
- Currently returns minimal timing information
- **TODO**: Add comprehensive event loop state information

## Integration Points

### Agenda Integration
The event loop monitors agenda timeouts:
```cpp
if (Rps_Agenda::agenda_timeout > 0
    && rps_elapsed_real_time() >= Rps_Agenda::agenda_timeout) {
    rps_stop_agenda_mechanism();
    break;
}
```

### FLTK Integration
When FLTK is enabled:
- Event handlers are registered with FLTK event system
- FLTK event loop is stopped when RefPerSys event loop stops
- File descriptor handlers are mirrored in FLTK

### Thread Safety
- All event loop data access is protected by `eld_mtx`
- Self-pipe provides thread-safe command queuing
- Atomic operations for global flags and counters

## Implementation Status and TODOs

### Completed Features
- ✅ Event loop data structure and initialization
- ✅ Self-pipe mechanism for thread communication
- ✅ Signal handling with signalfd for core signals
- ✅ File descriptor registration and removal
- ✅ Basic event loop polling infrastructure
- ✅ Pre-poll hook system for extensibility
- ✅ Thread-safe command posting (dump, GC, quit, exit)
- ✅ Agenda timeout integration

### Incomplete Implementations
- ❌ **Timer handling**: `rps_initialize_timerfd_in_event_loop()` and `rps_timerfd_read_handler()` are stubs
- ❌ **JSONRPC system**: Both initialization and event handlers are incomplete
- ❌ **Process handling**: `SelfPipe_Process` and related functionality not implemented
- ❌ **Signal handling**: SIGCHLD and SIGUSR1 handlers incomplete
- ❌ **Event loop explanation**: `rps_eventloop_explstring()` provides minimal information

### Known Warnings and TODOs
- **Line 104**: JSONRPC buffers should use `std::stringstream`
- **Line 107**: Use `rps_jsonrpc_rspstream` in response handling
- **Line 362**: Missing code for process handling in self-pipe
- **Line 639**: Incomplete `rps_jsonrpc_initialize`
- **Line 668**: Incomplete `rps_eventloop_explstring`
- **Line 715**: Incomplete main event loop
- **Line 718**: Missing timer handling code
- **Line 767**: Missing JSONRPC command output code
- **Line 798**: Missing JSONRPC EOF handling
- **Line 834**: Missing JSON message processing
- **Line 841**: Missing JSONRPC input handling
- **Line 875**: Missing timerfd handling code
- **Line 1069**: Missing cooperation with transient objects
- **Line 1126**: Missing final dump scheduling on SIGTERM
- **Line 1152**: Missing temporary dump on SIGUSR1
- **Line 1170**: Unimplemented timerfd handler
- **Line 1199**: Missing process handling in byte dispatch

### Future Enhancements
- Complete JSONRPC protocol implementation
- Full timerfd integration for scheduled events
- Process lifecycle management integration
- Enhanced signal handling (SIGCHLD completion)
- Comprehensive event loop status reporting
- Integration with `transientobj_rps.cc` for process management

## Usage Patterns

### Basic Event Loop Setup
```cpp
// Initialize event loop (called once from main thread)
rps_initialize_event_loop();

// Start event loop (blocks until stopped)
rps_event_loop();
```

### Thread-Safe Command Posting
```cpp
// From any thread, trigger operations asynchronously
rps_postpone_dump();                    // Schedule persistence dump
rps_postpone_garbage_collection();      // Schedule GC
rps_postpone_quit();                    // Stop event loop
rps_postpone_exit_with_dump();          // Dump and exit
```

### File Descriptor Registration
```cpp
// Add input handler for a file descriptor
rps_event_loop_add_input_fd_handler(fd, my_handler, "my input", user_data);

// Add output handler
rps_event_loop_add_output_fd_handler(fd, my_handler, "my output", user_data);

// Remove handlers when done
rps_event_loop_remove_input_fd_handler(fd);
```

### Pre-Poll Hook Registration
```cpp
// Register a function called before each poll
int hook_id = rps_register_event_loop_prepoller(
    [](struct pollfd* pfds, int& npoll, Rps_CallFrame* cf) {
        // Modify poll array, add custom FDs, etc.
    });

// Unregister when done
rps_unregister_event_loop_prepoller(hook_id);
```

### Event Loop Status
```cpp
// Check if event loop is running
if (rps_event_loop_is_running()) {
    long count = rps_event_loop_counter();
    // count contains loop iteration number
}

// Check if event loop completed successfully
if (rps_is_active_event_loop()) {
    // Event loop finished normally
}
```

## Design Rationale

### Poll-Based Architecture
- **Efficiency**: Single system call monitors multiple file descriptors
- **Portability**: Standard POSIX `poll()` interface
- **Flexibility**: Support for input, output, and special events

### Self-Pipe Pattern
- **Thread Safety**: Enables safe communication from any thread
- **Non-blocking**: Prevents deadlocks in signal handlers
- **Reliability**: Guaranteed wake-up mechanism for event loop

### Signal File Descriptor
- **Reliability**: Avoids race conditions in signal handling
- **Queueing**: Signals are queued and processed in order
- **Integration**: Seamlessly fits into poll-based event loop

### Extensibility Design
- **Pre-poll Hooks**: Allow dynamic addition of event sources
- **Handler Registration**: Clean API for file descriptor management
- **Modular Architecture**: Separate concerns for different event types

This implementation provides a solid foundation for RefPerSys's asynchronous I/O and inter-process communication needs, with clear pathways for completing the remaining functionality, particularly in JSONRPC communication and process management integration.