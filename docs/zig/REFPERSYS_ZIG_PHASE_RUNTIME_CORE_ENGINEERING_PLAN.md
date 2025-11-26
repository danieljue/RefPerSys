# RefPerSys Zig Phase 2: Runtime Core Engineering Plan

## Overview

Phase 2 implements the runtime execution engine, focusing on the event loop system and multi-threaded agenda execution. This phase establishes the core execution model that coordinates I/O operations, signal handling, and concurrent task execution - the heartbeat of the RefPerSys system.

## Data Structures

### 1. Event Loop Infrastructure

#### EventLoop
**Purpose**: Central coordinator for I/O multiplexing and event handling
**Key Components**:
- `poll_fds`: Array of `std.os.pollfd` structures (max 128)
- `handlers`: Array of event handler functions
- `self_pipe`: Internal communication pipe
- `signal_fd`: File descriptor for signal handling
- `timer_fd`: File descriptor for timer events

**Zig-Specific Optimizations**:
- `std.ArrayList` for dynamic handler management
- `std.os.linux` for direct system call access
- `std.atomic` for thread-safe state updates

**Performance Considerations**:
- Minimize syscall overhead with batched operations
- Cache-aligned data structures
- Lock-free handler registration where possible

#### EventHandler
**Purpose**: Callback interface for event processing
**Key Components**:
- `callback`: Function pointer with signature `fn(*EventLoop, fd: i32, events: u16) anyerror!void`
- `user_data`: Opaque user context pointer
- `explanation`: Debug description string

**Zig-Specific Optimizations**:
- `anyerror!void` for composable error handling
- `std.meta` for type-safe user data
- `comptime` validation of callback signatures

#### SignalState
**Purpose**: Signal handling coordination
**Key Components**:
- `mask`: Signal set for blocking/unblocking
- `fd`: signalfd file descriptor
- `pending_signals`: Queue of unprocessed signals

**Zig-Specific Optimizations**:
- `std.os.linux.sigset_t` for native signal sets
- `std.atomic.Queue` for thread-safe signal queuing
- Compile-time signal number validation

### 2. Agenda System

#### Agenda
**Purpose**: Multi-threaded task execution coordinator
**Key Components**:
- `queues`: Array of 3 priority queues (High/Normal/Low)
- `worker_threads`: Array of worker thread handles
- `mutex`: Recursive mutex for queue protection
- `condvar`: Condition variable for thread coordination
- `stats`: Execution statistics

**Zig-Specific Optimizations**:
- `std.Thread` for native threading
- `std.atomic` counters for statistics
- `std.ArrayList` for dynamic queue management

**Performance Considerations**:
- Lock contention minimization
- Work-stealing algorithms
- Cache-efficient data structures

#### Tasklet
**Purpose**: Executable unit of work
**Key Components**:
- `closure`: Function to execute with arguments
- `priority`: Execution priority level
- `metadata`: Creation time, execution count, etc.

**Zig-Specific Optimizations**:
- `anytype` parameters for flexible closures
- `std.time` for timing metadata
- `comptime` priority validation

#### WorkerState
**Purpose**: Per-worker thread state management
**Key Components**:
- `thread_id`: Unique worker identifier
- `current_tasklet`: Currently executing task
- `gc_callframe`: GC integration state
- `stats`: Thread-specific statistics

**Zig-Specific Optimizations**:
- `threadlocal` variables for per-thread state
- `std.atomic` for thread-safe statistics
- `std.meta` for type-safe tasklet handling

### 3. Coordination Primitives

#### SelfPipe
**Purpose**: Thread-safe inter-thread communication
**Key Components**:
- `read_fd`: Pipe read end
- `write_fd`: Pipe write end
- `buffer`: Command queue
- `mutex`: Synchronization for multi-writer scenarios

**Zig-Specific Optimizations**:
- `std.os.pipe` for pipe creation
- `std.atomic.Queue` for lock-free single-producer
- `std.os.linux` for direct I/O operations

#### TimeoutManager
**Purpose**: Timer event coordination
**Key Components**:
- `timer_fd`: timerfd file descriptor
- `pending_timeouts`: Priority queue of scheduled events
- `resolution`: Timer precision settings

**Zig-Specific Optimizations**:
- `std.time` for high-precision timing
- `std.PriorityQueue` for efficient timeout management
- `std.os.linux.timerfd_settime` for direct timer control

## Algorithms

### 1. Event Loop Execution

#### Poll-Based Event Multiplexing
**Algorithm Overview**:
1. Setup pollfd array with registered file descriptors
2. Calculate timeout based on agenda requirements
3. Call `poll()` system call
4. Process ready file descriptors
5. Handle timeouts and signals

**Zig Implementation Strategy**:
```zig
// High-level algorithm structure
fn eventLoopIteration(loop: *EventLoop) !void {
    // Setup poll array
    var poll_fds: [128]std.os.pollfd = undefined;
    const num_fds = try setupPollArray(loop, &poll_fds);

    // Calculate timeout
    const timeout_ms = calculateTimeout(loop);

    // Poll for events
    const events = try std.os.poll(&poll_fds, num_fds, timeout_ms);

    // Process events
    for (poll_fds[0..num_fds]) |*fd| {
        if (fd.revents != 0) {
            try dispatchEvent(loop, fd.fd, fd.revents);
        }
    }

    // Handle timeouts
    if (events == 0) {
        try handleTimeout(loop);
    }
}
```

**Performance Considerations**:
- Minimize syscall frequency with longer timeouts
- Batch event processing to reduce overhead
- Optimize pollfd array layout for cache efficiency

#### Signal Handling
**Algorithm Overview**:
1. Block signals using sigprocmask
2. Read signals from signalfd
3. Queue signals for processing
4. Dispatch to appropriate handlers

**Zig Implementation Strategy**:
```zig
// High-level algorithm structure
fn processSignals(state: *SignalState) !void {
    var siginfo: std.os.linux.signalfd_siginfo = undefined;

    // Read available signals
    while (true) {
        const bytes_read = std.os.read(state.fd, std.mem.asBytes(&siginfo)) catch break;
        if (bytes_read != @sizeOf(std.os.linux.signalfd_siginfo)) break;

        // Queue signal for processing
        try state.pending_signals.put(siginfo);

        // Dispatch based on signal type
        switch (siginfo.signo) {
            std.os.SIG.INT => try handleInterrupt(state),
            std.os.SIG.TERM => try handleTermination(state),
            std.os.SIG.CHLD => try handleChildProcess(state),
            else => try handleOtherSignal(state, siginfo),
        }
    }
}
```

### 2. Agenda Task Execution

#### Priority-Based Task Scheduling
**Algorithm Overview**:
1. Check priority queues in order (High → Normal → Low)
2. Dequeue highest priority tasklet
3. Execute tasklet with error handling
4. Update execution statistics

**Zig Implementation Strategy**:
```zig
// High-level algorithm structure
fn fetchAndExecuteTasklet(agenda: *Agenda, worker_id: usize) !void {
    const tasklet = fetchHighestPriorityTasklet(agenda) orelse {
        // No tasks available, wait for new tasks
        try waitForNewTasks(agenda);
        return;
    };

    // Execute tasklet
    agenda.worker_states[worker_id].current_tasklet = tasklet;
    defer agenda.worker_states[worker_id].current_tasklet = null;

    try executeTasklet(tasklet);

    // Update statistics
    std.atomic.fetchAdd(&agenda.stats.tasks_completed, 1, .monotonic);
}

fn fetchHighestPriorityTasklet(agenda: *Agenda) ?*Tasklet {
    agenda.mutex.lock();
    defer agenda.mutex.unlock();

    // Check queues in priority order
    inline for (.{ .high, .normal, .low }) |priority| {
        if (agenda.queues[@enumToInt(priority)].popOrNull()) |tasklet| {
            return tasklet;
        }
    }

    return null;
}
```

**Performance Considerations**:
- Lock contention minimization with fine-grained locking
- Work-stealing for load balancing
- Cache-efficient tasklet structures

#### Worker Thread Lifecycle
**Algorithm Overview**:
1. Initialize thread-local state
2. Enter main execution loop
3. Process tasks or wait for work
4. Coordinate with garbage collector
5. Handle shutdown gracefully

**Zig Implementation Strategy**:
```zig
// High-level algorithm structure
fn workerThreadMain(agenda: *Agenda, worker_id: usize) !void {
    // Initialize thread state
    agenda.worker_states[worker_id] = WorkerState.init(worker_id);

    // Main execution loop
    while (agenda.running.load(.acquire)) {
        // Check for GC coordination
        if (agenda.needs_gc.load(.acquire)) {
            try coordinateGarbageCollection(agenda, worker_id);
        } else {
            // Execute tasks
            fetchAndExecuteTasklet(agenda, worker_id) catch |err| {
                // Log error and continue
                std.log.err("Tasklet execution failed: {}", .{err});
            };
        }
    }

    // Cleanup
    agenda.worker_states[worker_id].deinit();
}
```

### 3. Inter-Thread Communication

#### Self-Pipe Messaging
**Algorithm Overview**:
1. Write command byte to pipe
2. Read commands in event loop
3. Dispatch to appropriate handlers
4. Handle overflow scenarios

**Zig Implementation Strategy**:
```zig
// High-level algorithm structure
const Command = enum(u8) {
    dump = 'D',
    garbage_collect = 'G',
    quit = 'Q',
    exit = 'X',
};

fn sendCommand(pipe: *SelfPipe, cmd: Command) !void {
    const byte = @enumToInt(cmd);
    _ = try std.os.write(pipe.write_fd, &[_]u8{byte});
}

fn processCommands(pipe: *SelfPipe) !void {
    var buffer: [64]u8 = undefined;

    const bytes_read = try std.os.read(pipe.read_fd, &buffer);
    for (buffer[0..bytes_read]) |byte| {
        const cmd = std.meta.intToEnum(Command, byte) catch {
            std.log.warn("Unknown command byte: {}", .{byte});
            continue;
        };

        try dispatchCommand(cmd);
    }
}
```

## References

### Inspiration from Existing Codebase

#### Event Loop System
- **eventloop_rps.cc**: Complete event loop implementation
- **REFPERSYS_MAIN_LOOP_ARCHITECTURE.md**: Architecture analysis
- **rps_event_loop()**: Poll-based multiplexing algorithm
- **rps_self_pipe_write_byte()**: Self-pipe communication

#### Agenda System
- **agenda_rps.cc**: Worker thread implementation
- **REFPERSYS_CONCURRENCY_MODEL_ANALYSIS.md**: Concurrency analysis
- **Rps_Agenda::run_agenda_worker()**: Task execution algorithm
- **Rps_Agenda::fetch_tasklet_to_run()**: Priority scheduling

#### Data Structures
- **event_loop_data_st**: Event loop state structure
- **Rps_Agenda**: Agenda coordinator design
- **Tasklet payload**: Work unit structure

### Zig-Specific Adaptations

#### System Calls
- Replace `poll(2)` with `std.os.poll`
- Use `std.os.linux.signalfd` for signal handling
- Leverage `std.os.pipe` for self-pipe creation

#### Threading
- `std.Thread` instead of pthreads
- `std.sync.Mutex` and `std.sync.Condition` for synchronization
- `threadlocal` variables for per-thread state

#### Error Handling
- `anyerror!void` for composable error handling
- `try`/`catch` for system call failures
- Structured error types for different failure modes

## Integration Points

### With Phase 1: Foundation
- **GC Integration**: Event loop triggers GC, agenda coordinates with GC
- **Memory Allocation**: All runtime structures use Phase 1 allocators
- **Value Operations**: Event handlers and tasklets manipulate RpsValue instances

### With Phase 3: Advanced Features
- **Persistence**: Event loop handles I/O for loading/saving
- **Plugin System**: Dynamic loading coordinated through event loop
- **JIT Compilation**: Code generation triggered via agenda tasks

### With Phase 4: User Interfaces
- **REPL Integration**: Command processing through event loop
- **GUI Events**: Window system events fed through poll interface
- **External Communication**: JSONRPC over FIFOs

### With Phase 5: Ecosystem
- **Build System**: Runtime components integrated into build.zig
- **Testing**: Event loop and agenda testing frameworks
- **Profiling**: Performance monitoring of runtime components

### Testing Strategy
- **Event Loop Tests**: I/O multiplexing, signal handling, timer accuracy
- **Agenda Tests**: Task scheduling, thread safety, load balancing
- **Integration Tests**: Event loop + agenda coordination
- **Performance Tests**: Throughput, latency, resource usage
- **Stress Tests**: High concurrency, memory pressure, error conditions

This engineering plan establishes the runtime execution foundation, providing the coordination layer that makes RefPerSys responsive and efficient in the Zig implementation.