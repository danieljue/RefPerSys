# Agenda Implementation Analysis (`agenda_rps.cc`)

## Overview

The `agenda_rps.cc` file implements the core **agenda mechanism** for the Reflective Persistent System (RefPerSys), providing a multi-threaded task scheduling system with priority-based queues, worker thread management, and coordinated garbage collection. This file is central to RefPerSys's concurrency model, enabling efficient parallel execution of tasks while maintaining thread safety and memory management.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti, Nimesh Neema
- **Copyright**: © 2020-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include "refpersys.hh"

// Git versioning information
extern "C" const char rps_agenda_gitid[];
extern "C" const char rps_agenda_shortgitid[];
extern "C" const char rps_agenda_date[];
extern "C" const char rps_agenda_timestamp[];
```

### Key Dependencies
- `refpersys.hh`: Main header file containing core system definitions
- Threading libraries: `std::thread`, `std::mutex`, `std::condition_variable`
- Atomic operations: `std::atomic` for thread-safe counters
- JSON handling: `Json::Value` for serialization

## Global Variables and Constants

### Thread-Local Variables
```cpp
thread_local int rps_curthread_ix;                    // Current thread index
thread_local Rps_CallFrame* rps_curthread_callframe;  // Current call frame
```

### Agenda State Variables
```cpp
unsigned long rps_run_delay;                           // Run delay in seconds
double Rps_Agenda::agenda_timeout;                     // Agenda timeout value

// Synchronization primitives
std::recursive_mutex Rps_Agenda::agenda_mtx_;          // Main agenda mutex
std::condition_variable_any Rps_Agenda::agenda_changed_condvar_; // Condition variable

// Task queues (one per priority level)
std::deque<Rps_ObjectRef> Rps_Agenda::agenda_fifo_[Rps_Agenda::AgPrio__Last];

// Thread management
constexpr unsigned rps_JMAX = RPS_NBJOBS_MAX + 2;
std::atomic<unsigned long> Rps_Agenda::agenda_add_counter_;
std::atomic<bool> Rps_Agenda::agenda_is_running_;
std::atomic<std::thread*> Rps_Agenda::agenda_thread_array_[rps_JMAX];

// Garbage collection coordination
std::atomic<Rps_Agenda::workthread_state_en> Rps_Agenda::agenda_work_thread_state_[rps_JMAX];
std::atomic<bool> Rps_Agenda::agenda_needs_garbcoll_;
std::atomic<uint64_t> Rps_Agenda::agenda_cumulw_gc_;
std::atomic<Rps_CallFrame*> Rps_Agenda::agenda_work_gc_callframe_[rps_JMAX];
std::atomic<Rps_CallFrame**> Rps_Agenda::agenda_work_gc_current_callframe_ptr[rps_JMAX];
```

### Priority System
```cpp
const char* Rps_Agenda::agenda_priority_names[Rps_Agenda::AgPrio__Last];

// Priority levels (enumerated):
// - AgPrio_Low: "low_priority"
// - AgPrio_Normal: "normal_priority"
// - AgPrio_High: "high_priority"
```

## Core Classes

### Rps_Agenda Class

The `Rps_Agenda` class provides static methods for managing the agenda system. It uses static members to maintain global state accessible from all threads.

#### Key Methods

##### Initialization
```cpp
void Rps_Agenda::initialize(void)
```
**Purpose**: Sets up the agenda system at startup
- Initializes priority names
- Sets agenda timeout based on `rps_run_delay`
- Logs initialization details

##### Task Management
```cpp
void Rps_Agenda::add_tasklet(agenda_prio_en prio, Rps_ObjectRef obtasklet)
```
**Purpose**: Adds a tasklet to the specified priority queue
- Validates priority range and tasklet validity
- Thread-safely adds to the appropriate FIFO queue
- Increments the add counter and notifies waiting threads

```cpp
Rps_ObjectRef Rps_Agenda::fetch_tasklet_to_run(void)
```
**Purpose**: Retrieves the next tasklet to execute
- Searches priority queues from high to low
- Returns the front tasklet from the highest priority non-empty queue
- Removes the tasklet from its queue

##### Garbage Collection Integration
```cpp
void Rps_Agenda::gc_mark(Rps_GarbageCollector& gc)
```
**Purpose**: Marks all tasklets in agenda queues during GC
- Acquires agenda mutex for thread safety
- Iterates through all priority queues
- Marks each valid object reference

##### Serialization Support
```cpp
void Rps_Agenda::dump_scan_agenda(Rps_Dumper* du)
void Rps_Agenda::dump_json_agenda(Rps_Dumper* du, Json::Value& jv)
```
**Purpose**: Supports persistence and debugging
- `dump_scan_agenda`: Scans objects for dumping
- `dump_json_agenda`: Exports agenda state as JSON

##### Worker Thread Execution
```cpp
void Rps_Agenda::run_agenda_worker(int ix)
```
**Purpose**: Main execution loop for worker threads
- Sets up thread-local variables and call frames
- Monitors memory usage for GC triggers
- Fetches and executes tasklets in a loop
- Handles thread synchronization and state management

**Algorithm Flow**:
1. Initialize thread context (name, index, call frame)
2. Wait for thread registration in `agenda_thread_array_`
3. Main execution loop while agenda is running:
   - Check GC threshold and trigger collection if needed
   - Fetch next tasklet to execute
   - Execute tasklet closure if available
   - Wait for new tasks if queue is empty

##### Garbage Collection Coordination
```cpp
void Rps_Agenda::do_garbage_collect(int ix, Rps_CallFrame* callframe)
```
**Purpose**: Coordinates stop-the-world garbage collection across all worker threads
- Transitions thread to GC state
- Waits for all threads to enter GC state
- Primary thread (ix=1) performs actual GC with call stack marking
- Transitions all threads to EndGC state

**Coordination Protocol**:
1. Set thread state to `WthrAg_GC`
2. Store current call frame for stack scanning
3. Wait for all worker threads to reach GC state
4. Primary thread executes garbage collection
5. Transition all threads to `WthrAg_EndGC`
6. Resume normal execution

### Payload Classes

#### Rps_PayloadAgenda
Represents the agenda object itself, owned by the root agenda object.

```cpp
class Rps_PayloadAgenda : public Rps_Payload {
public:
    ~Rps_PayloadAgenda();
    void gc_mark(Rps_GarbageCollector& gc) const;
    void dump_scan(Rps_Dumper* du) const;
    void dump_json_content(Rps_Dumper* du, Json::Value& jv) const;
    bool is_erasable() const;
};
```

**Key Methods**:
- Delegates to `Rps_Agenda` static methods for GC marking and dumping
- Cannot be erased (returns `false` from `is_erasable`)

#### Rps_PayloadTasklet
Represents executable task units with closures and metadata.

```cpp
class Rps_PayloadTasklet : public Rps_Payload {
    Rps_ClosureValue tasklet_todoclos;  // The closure to execute
    double tasklet_obsoltime;           // Obsolete time (for expiration)
    bool tasklet_permanent;             // Whether tasklet persists
};
```

**Key Methods**:
```cpp
void gc_mark(Rps_GarbageCollector& gc) const
```
- Marks closure if obsolete time hasn't passed

```cpp
void dump_scan(Rps_Dumper* du) const
void dump_json_content(Rps_Dumper* du, Json::Value& jv) const
```
- Handles serialization of permanent tasklets with valid closures

```cpp
bool is_erasable() const
```
- Returns `true` (tasklets can be mutated to other types)

## Global Functions

### Agenda Lifecycle Management
```cpp
void rps_run_agenda_mechanism(int nbjobs)
```
**Purpose**: Starts the agenda system with specified number of worker threads
- Validates thread count bounds
- Creates and starts worker threads
- Enters main event loop (currently incomplete - TODO items reference event loop integration)
- Waits for agenda shutdown and cleans up threads

```cpp
void rps_stop_agenda_mechanism(void)
```
**Purpose**: Signals agenda shutdown
- Sets `agenda_is_running_` to false
- Notifies all waiting threads

### Persistence Support
```cpp
void rpsldpy_agenda(Rps_ObjectZone* obz, Rps_Loader* ld, const Json::Value& jv,
                    Rps_Id spacid, unsigned lineno)
```
**Purpose**: Loads agenda state from persistent storage
- Validates that the object is the agenda root
- Creates `Rps_PayloadAgenda` payload
- Restores tasklets from JSON array for each priority level

```cpp
void rpsldpy_tasklet(Rps_ObjectZone* obz, Rps_Loader* ld, const Json::Value& jv,
                     Rps_Id spacid, unsigned lineno)
```
**Purpose**: Loads tasklet objects from persistence
- Creates `Rps_PayloadTasklet` payload
- Restores closure and obsolete time from JSON

## Thread States and Coordination

### Worker Thread States
```cpp
enum workthread_state_en {
    WthrAg_Idle,     // Waiting for tasks
    WthrAg_Run,      // Executing a tasklet
    WthrAg_GC,       // Participating in garbage collection
    WthrAg_EndGC,    // Transitioning out of GC
    WthrAg__None     // Thread terminated
};
```

### Synchronization Patterns

1. **Task Addition**: Mutex-protected queue insertion with condition variable notification
2. **Task Retrieval**: Mutex-protected priority-based dequeue
3. **GC Coordination**: Atomic state transitions with condition variable waits
4. **Thread Management**: Atomic thread pointers with careful startup sequencing

## Integration Points

### With Garbage Collection (`garbcoll_rps.cc`)
- Agenda provides call frame pointers for stack scanning
- Coordinates stop-the-world pauses across all worker threads
- Tracks memory usage thresholds for GC triggering

### With Event Loop (`eventloop_rps.cc`)
- TODO items indicate planned integration for process and I/O monitoring
- Event loop will handle SIGCHLD and file descriptor polling
- Self-pipe communication for thread-safe commands

### With Persistence (`load_rps.cc`, `dump_rps.cc`)
- JSON serialization of agenda state and tasklets
- Loading restores task queues and closures
- Dump scanning for object graph traversal

## Performance Characteristics

### Threading
- **Worker Count**: Configurable (3-24 threads typical)
- **Task Distribution**: O(1) enqueue, O(1) amortized dequeue
- **Synchronization**: Minimal locking with condition variables
- **Memory**: Thread-local call frames avoid allocation contention

### Garbage Collection
- **Coordination**: O(n) wait for n worker threads
- **Pause Time**: Stop-the-world with precise stack marking
- **Threshold**: Configurable memory usage triggers

### Task Execution
- **Priority Handling**: High priority tasks processed first
- **Closure Application**: Direct function call with call frame context
- **Error Handling**: Exception catching with state reset

## Design Patterns and Rationale

### Static Class Design
- `Rps_Agenda` uses static members for global accessibility
- Avoids object instantiation overhead
- Ensures single agenda instance per process

### Priority Queue Architecture
- Three-level priority system for responsiveness
- FIFO within priorities for fairness
- Simple implementation with `std::deque`

### Cooperative Threading Model
- Worker threads voluntarily participate in GC
- Condition variables for efficient waiting
- Atomic operations for state coordination

### Thread-Local Storage Usage
- Call frames provide GC roots for each thread
- Thread indices enable per-thread debugging
- Avoids synchronization for thread-specific data

## Known Limitations and TODOs

### Incomplete Features
1. **Event Loop Integration**: TODO §1 and §2 reference missing cooperation with Unix processes and popen commands
2. **Tasklet Destructor**: `Rps_PayloadTasklet` destructor is stubbed as "unimplemented"
3. **Agenda Loading**: Marked as "incomplete rpsldpy_agenda"

### Threading Constraints
- Fixed maximum thread count (`RPS_NBJOBS_MAX`)
- Stop-the-world GC may cause latency spikes
- No work stealing between threads

## Usage Examples

### Adding a Tasklet
```cpp
// Create a tasklet with a closure
Rps_ObjectRef tasklet = Rps_ObjectRef::make_tasklet(some_closure);
// Add to high priority queue
Rps_Agenda::add_tasklet(Rps_Agenda::AgPrio_High, tasklet);
```

### Starting the Agenda
```cpp
// Start with 8 worker threads
rps_run_agenda_mechanism(8);
// Agenda runs until stopped
rps_stop_agenda_mechanism();
```

### Tasklet Closure Execution
```cpp
// Within a tasklet closure
void my_tasklet_function(Rps_CallFrame* cf, Rps_ObjectRef tasklet) {
    // Perform work using call frame context
    // Access tasklet payload if needed
    auto payload = tasklet->get_dynamic_payload<Rps_PayloadTasklet>();
}
```

## Future Evolution

### Potential Enhancements
1. **Work Stealing**: Allow idle threads to steal tasks from busy threads
2. **Incremental GC**: Reduce pause times with concurrent collection
3. **Task Affinity**: Bind tasks to specific threads
4. **Coroutine Support**: Lightweight task switching
5. **NUMA Optimization**: Thread placement for multi-socket systems

### Integration Improvements
1. **Complete Event Loop**: Implement process monitoring and I/O handling
2. **Self-Pipe Commands**: Extend inter-thread communication
3. **Task Persistence**: Enhanced serialization of running tasks

This implementation provides a robust foundation for RefPerSys's concurrent execution model, balancing performance, safety, and complexity while enabling efficient parallel task processing.