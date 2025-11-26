# RefPerSys Main Loop Architecture

## Overview

RefPerSys employs a sophisticated **dual-loop concurrent architecture** that combines an event-driven main loop with a multi-threaded agenda system. This design enables efficient handling of both interactive user interfaces and background computational tasks while maintaining thread safety and performance.

## Architecture Components

### 1. Event Loop (Main Thread)

The event loop runs in the main thread and is implemented using the POSIX `poll(2)` system call for efficient I/O multiplexing.

#### Core Implementation (`rps_event_loop()` in `eventloop_rps.cc`)

```cpp
void rps_event_loop(void) {
    while (!rps_stop_event_loop_flag.load()) {
        // Setup poll file descriptors
        int nbfdpoll = setup_poll_descriptors(pollarr);

        // Poll with timeout
        int respoll = poll(pollarr, nbfdpoll, poll_timeout);

        // Process ready events
        if (respoll > 0) {
            process_ready_events(pollarr, nbfdpoll);
        }

        // Check for agenda timeout
        if (agenda_timeout_reached()) break;
    }
}
```

#### Event Sources

**File Descriptor Types:**
- **Signal FDs**: `signalfd(2)` for POSIX signals (SIGTERM, SIGINT, SIGCHLD, SIGALRM, etc.)
- **Timer FDs**: `timerfd_create(2)` for scheduled events
- **JSONRPC FIFOs**: Named pipes for external GUI communication
- **Self-Pipe**: Internal thread communication mechanism

**Maximum Capacity:**
- Up to 128 file descriptors (`RPS_MAXPOLL_FD`)
- Configurable poll timeout (default: 1600ms with debug flags, 111ms without)

#### Event Processing

Each file descriptor has an associated handler function:

```cpp
struct pollfd {
    int fd;         // File descriptor
    short events;   // Requested events (POLLIN, POLLOUT, etc.)
    short revents;  // Returned events
};

Rps_EventHandler_sigt* handler;  // Handler function
void* data;                      // User data
const char* explanation;         // Debug description
```

#### Self-Pipe Communication

The self-pipe enables thread-safe communication between the agenda system and event loop:

```cpp
enum self_pipe_code_en {
    SelfPipe__NONE = 0,
    SelfPipe_Dump = 'D',      // Trigger state dump
    SelfPipe_GarbColl = 'G',  // Trigger garbage collection
    SelfPipe_Process = 'P',   // Process management
    SelfPipe_Quit = 'Q',      // Stop event loop
    SelfPipe_Exit = 'X'       // Exit with dump
};
```

### 2. Agenda System (Worker Threads)

The agenda system provides multi-threaded task execution with priority scheduling.

#### Worker Thread Architecture

**Thread Pool:**
- Configurable thread count (3-24 workers, default 5)
- Each worker runs `Rps_Agenda::run_agenda_worker(ix)`
- Thread naming: `rps-agw#N` (agenda worker #N)

**Priority Queues:**
```cpp
enum agenda_prio_en {
    AgPrio_Low = 0,
    AgPrio_Normal,
    AgPrio_High,
    AgPrio__Last
};

std::deque<Rps_ObjectRef> agenda_fifo_[AgPrio__Last];
```

#### Tasklet Execution Model

**Tasklets** are small, fast-executing units designed to complete in milliseconds:

```cpp
void Rps_Agenda::run_agenda_worker(int ix) {
    while (agenda_is_running_.load()) {
        // Check garbage collection threshold
        if (gc_needed()) {
            coordinate_garbage_collection(ix);
        } else {
            // Fetch highest priority tasklet
            Rps_ObjectRef tasklet = fetch_tasklet_to_run();

            if (tasklet) {
                execute_tasklet(tasklet);
            } else {
                // Wait for new tasklets
                wait_for_agenda_changes();
            }
        }
    }
}
```

#### Garbage Collection Coordination

All worker threads participate in garbage collection:

```cpp
void Rps_Agenda::do_garbage_collect(int ix, Rps_CallFrame* callframe) {
    // Signal GC state
    agenda_work_thread_state_[ix].store(WthrAg_GC);

    // Wait for all threads to reach GC state
    wait_for_all_workers_gc_state();

    // First worker performs actual GC
    if (ix == 1) {
        perform_garbage_collection();
    }

    // Transition to EndGC state
    agenda_work_thread_state_[ix].store(WthrAg_EndGC);
}
```

### 3. Coordination Mechanisms

#### Thread Synchronization

**Mutexes and Condition Variables:**
```cpp
std::recursive_mutex Rps_Agenda::agenda_mtx_;
std::condition_variable_any Rps_Agenda::agenda_changed_condvar_;
```

**Atomic Operations:**
- `agenda_is_running_`: Controls agenda lifecycle
- `agenda_add_counter_`: Tracks tasklet additions
- `agenda_needs_garbcoll_`: Signals GC requirements

#### Timeout Management

**Agenda Timeout:**
- Configurable via `--run-delay` option
- Checked periodically in event loop
- Triggers graceful shutdown when exceeded

**Poll Timeouts:**
- Adaptive: 20ms + configurable delay
- Shorter with debug flags enabled
- Prevents indefinite blocking

### 4. Integration with User Interfaces

#### REPL Mode (Default Interactive)

- Event loop processes stdin/stdout
- Command parsing and execution
- Real-time expression evaluation

#### FLTK GUI Mode

- Integrates with FLTK event system
- `rps_fltk_run()` replaces poll-based event loop
- Windowed environment with menus and dialogs

#### JSONRPC FIFO Mode

- Headless operation with external GUI
- FIFO-based communication:
  - `PREFIX.cmd`: Commands to RefPerSys
  - `PREFIX.out`: Responses from RefPerSys
- JSON-RPC protocol for structured messaging

#### Batch Mode

- No event loop execution
- Direct command processing
- Exits after completion

### 5. Main Execution Flow

From `main_rps.cc`, the complete execution sequence:

```cpp
int main(int argc, char** argv) {
    // Phase 1-9: System initialization
    rps_early_initialization();
    // ... 8 more phases ...

    // Load persistent state
    rps_load_from(rps_my_load_dir);

    // Run loaded application (plugins, commands)
    rps_run_loaded_application(argc, argv);

    // Enter operational mode
    if (batch_mode) {
        // Direct execution, no loops
        return exit_code;
    } else {
        // Start agenda mechanism (worker threads)
        rps_run_agenda_mechanism(rps_nbjobs);

        // Start appropriate event loop
        if (rps_fltk_enabled()) {
            rps_fltk_run();  // FLTK event loop
        } else {
            rps_event_loop();  // poll-based event loop
        }

        // Cleanup and exit
        return exit_code;
    }
}
```

### 6. Performance Characteristics

#### Event Loop
- **Low Latency**: Immediate response to I/O events
- **Scalable**: Up to 128 concurrent file descriptors
- **Efficient**: `poll(2)` system call with adaptive timeouts

#### Agenda System
- **High Throughput**: Parallel task execution
- **Priority-Based**: Ensures critical tasks run first
- **GC-Safe**: Coordinated garbage collection across threads

#### Memory Management
- **GC Integration**: Automatic collection with allocation thresholds
- **Thread Safety**: All operations properly synchronized
- **Minimal Overhead**: Efficient coordination mechanisms

### 7. Error Handling and Recovery

#### Signal Handling
- **SIGTERM/SIGINT**: Graceful shutdown with optional dump
- **SIGCHLD**: Child process management
- **SIGALRM/SIGVTALRM**: Timer-based operations

#### Exception Safety
- **Tasklet Isolation**: Exceptions in one tasklet don't crash others
- **Thread Recovery**: Failed threads log errors and continue
- **State Consistency**: Atomic operations prevent corruption

### 8. Debugging and Monitoring

#### Debug Categories
- `REPL`: Event loop operations
- `EVENT_LOOP`: Poll and I/O operations
- `GUI`: User interface interactions

#### Performance Monitoring
- **Loop Counters**: Track event loop iterations
- **Timing Statistics**: CPU and elapsed time tracking
- **Thread State**: Monitor worker thread status

This dual-loop architecture provides RefPerSys with the flexibility to handle both interactive and batch processing workloads efficiently, with sophisticated coordination ensuring thread safety and optimal performance.