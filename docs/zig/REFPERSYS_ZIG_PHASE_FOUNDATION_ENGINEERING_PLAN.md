# RefPerSys Zig Phase 1: Foundation Engineering Plan

## Overview

Phase 1 establishes the fundamental memory management and data structure foundations for the RefPerSys Zig reimplementation. This phase focuses on the custom garbage collector and value system - the most critical and complex components that must be rock-solid before any other functionality can be built.

## Data Structures

### 1. Memory Management Infrastructure

#### QuasiZone
**Purpose**: Memory zone abstraction for GC-managed allocations
**Key Components**:
- `base_addr`: Pointer to mmap-allocated memory block
- `size`: Total zone size in bytes
- `allocated`: Current allocation offset
- `metadata`: Zone type and GC generation info

**Zig-Specific Optimizations**:
- Use `std.mem.Allocator` interface for composability
- Leverage Zig's `comptime` for zone type specialization
- `align(@alignOf(usize))` for optimal memory access

**Performance Considerations**:
- Minimize cache misses with proper alignment
- Support for different allocation patterns (bump, arena, pool)
- Memory safety with bounds checking in debug builds

#### MemoryArena
**Purpose**: Region-based memory allocator for short-lived objects
**Key Components**:
- `buffer`: Contiguous memory block
- `offset`: Current allocation position
- `parent_zone`: Owning QuasiZone for GC coordination

**Zig-Specific Optimizations**:
- `std.ArrayList` for dynamic growth
- Compile-time size calculation with `comptime`
- `std.mem.zeroes` for efficient initialization

#### WriteBarrier
**Purpose**: Track object modifications for generational GC
**Key Components**:
- `card_table`: Bit array marking modified memory regions
- `dirty_cards`: Queue of cards needing scanning
- `card_size`: Bytes per card (typically 512 bytes)

**Zig-Specific Optimizations**:
- `std.bit_set` for efficient bit operations
- SIMD operations for bulk card marking
- `std.atomic` for thread-safe updates

### 2. Garbage Collector Core

#### GCHeader
**Purpose**: Object header for GC metadata
**Key Components**:
- `mark_bits`: Mark-sweep state (3 bits for tri-color marking)
- `size`: Object size in bytes
- `type_tag`: Object type identifier
- `generation`: GC generation (young/old)

**Zig-Specific Optimizations**:
- Packed struct with `@sizeOf` verification
- Union types for type-specific metadata
- `packed struct` for minimal memory overhead

#### GCState
**Purpose**: Global GC state management
**Key Components**:
- `roots`: Array of GC roots (stack, globals, registers)
- `worklist`: Objects pending processing
- `phase`: Current GC phase (mark/sweep/compact)
- `stats`: Collection statistics

**Zig-Specific Optimizations**:
- `std.ArrayList` for dynamic root management
- `std.hash_map` for object identity tracking
- `std.time` for pause time measurement

### 3. Value System

#### RpsValue
**Purpose**: Universal tagged union for all RefPerSys values
**Key Components**:
- Tagged union with type discriminators
- Support for: integers, floats, strings, objects, sets, tuples, closures
- Type-safe operations with compile-time checking

**Zig-Specific Optimizations**:
- `union(enum)` for type-safe tagged unions
- `comptime` type checking and generation
- `std.meta` for reflection capabilities

#### ObjectHeader
**Purpose**: Common header for all objects
**Key Components**:
- `oid`: 128-bit unique object identifier
- `class`: Pointer to class object
- `ref_count`: Reference count for RC optimization
- `gc_header`: Embedded GC metadata

**Zig-Specific Optimizations**:
- `extern struct` for C ABI compatibility
- `std.atomic` for thread-safe reference counting
- `packed struct` to minimize header size

## Algorithms

### 1. Memory Allocation

#### Bump Pointer Allocation
**Algorithm Overview**:
1. Maintain allocation pointer in zone
2. Increment pointer by requested size
3. Check for zone exhaustion
4. Trigger GC if necessary

**Zig Implementation Strategy**:
```zig
// High-level algorithm structure
fn bumpAllocate(zone: *QuasiZone, size: usize, alignment: usize) ?*anyopaque {
    const aligned_size = std.mem.alignForward(size, alignment);
    const current_offset = @atomicLoad(usize, &zone.allocated, .acquire);
    const new_offset = current_offset + aligned_size;

    if (new_offset > zone.size) return null;

    if (@cmpxchgStrong(usize, &zone.allocated, current_offset, new_offset, .acq_rel, .acquire) == current_offset) {
        return @intToPtr(*anyopaque, @intCast(usize, zone.base_addr) + current_offset);
    }

    return null; // Retry or trigger GC
}
```

**Performance Considerations**:
- Lock-free for single-threaded allocation
- Minimal overhead (pointer increment)
- Excellent cache locality

#### Arena Allocation
**Algorithm Overview**:
1. Allocate from current arena block
2. Create new block when exhausted
3. Batch deallocation at arena reset

**Zig Implementation Strategy**:
- `std.heap.ArenaAllocator` as foundation
- Custom extensions for GC integration
- Compile-time arena size optimization

### 2. Garbage Collection

#### Tri-Color Mark-and-Sweep
**Algorithm Overview**:
1. **Mark Phase**: Traverse live objects from roots
2. **Sweep Phase**: Free unmarked objects
3. **Compact Phase**: Optional memory defragmentation

**Zig Implementation Strategy**:
```zig
// High-level algorithm structure
const GCPhase = enum { idle, mark, sweep, compact };

fn collect(collector: *GCCollector) void {
    collector.phase = .mark;
    markFromRoots(collector);
    collector.phase = .sweep;
    sweepUnmarked(collector);
    collector.phase = .idle;
}

fn markFromRoots(collector: *GCCollector) void {
    var worklist = std.ArrayList(*ObjectHeader).initCapacity(collector.allocator, 1024);

    // Add roots to worklist
    for (collector.roots) |root| {
        if (markObject(root)) {
            worklist.appendAssumeCapacity(root);
        }
    }

    // Process worklist
    while (worklist.popOrNull()) |obj| {
        markObjectReferences(obj, &worklist);
    }
}
```

**Performance Considerations**:
- Minimize pause times with incremental marking
- Parallel marking with work-stealing
- Write barriers to avoid full heap scans

#### Generational Collection
**Algorithm Overview**:
1. Frequent collection of young generation
2. Infrequent collection of old generation
3. Promotion of long-lived objects

**Zig Implementation Strategy**:
- Separate arenas for generations
- Age-based object promotion
- Cross-generational reference tracking

### 3. Value Operations

#### Tagged Union Dispatch
**Algorithm Overview**:
1. Extract type tag from value
2. Dispatch to type-specific operation
3. Handle type mismatches gracefully

**Zig Implementation Strategy**:
```zig
// High-level algorithm structure
pub const ValueOperation = union(enum) {
    add: struct { lhs: RpsValue, rhs: RpsValue },
    compare: struct { lhs: RpsValue, rhs: RpsValue },
    // ... other operations
};

fn dispatchOperation(op: ValueOperation) !RpsValue {
    switch (op) {
        .add => |args| return addValues(args.lhs, args.rhs),
        .compare => |args| return compareValues(args.lhs, args.rhs),
        // ...
    }
}

fn addValues(lhs: RpsValue, rhs: RpsValue) !RpsValue {
    switch (lhs) {
        .int => |lval| switch (rhs) {
            .int => |rval| return RpsValue{ .int = lval + rval },
            .float => |rval| return RpsValue{ .float = @intToFloat(f64, lval) + rval },
            else => return error.TypeMismatch,
        },
        // ... handle other combinations
    }
}
```

## References

### Inspiration from Existing Codebase

#### Memory Management
- **REFPERSYS_MEMORY_MANAGEMENT_GARBAGE_COLLECTION_ANALYSIS.md**: Detailed GC algorithm analysis
- **garbcoll_rps.cc**: C++ GC implementation patterns
- **QuasiZone concept**: Memory zone abstraction foundation

#### Value System
- **REFPERSYS_OBJECT_MODEL_ANALYSIS.md**: Object structure and identity
- **refpersys.hh lines 1732-1912**: Rps_Value tagged union implementation
- **scalar_rps.cc**: Scalar value operations

#### Data Structures
- **Object zones**: `Rps_ObjectZone` in objects_rps.cc
- **Value representation**: Tagged union patterns in refpersys.hh
- **Memory layout**: Zone-based allocation in garbcoll_rps.cc

### Zig-Specific Adaptations

#### Memory Safety
- Replace C++ raw pointers with Zig's optional pointers
- Use `std.heap` for allocation instead of manual mmap
- Leverage Zig's `comptime` for type safety

#### Error Handling
- Replace C++ exceptions with Zig's error unions
- Use `try`/`catch` for allocation failures
- `std.mem.Allocator.Error` for memory exhaustion

#### Concurrency
- `std.Thread` instead of pthreads
- `std.atomic` for thread-safe operations
- `std.sync` primitives for coordination

## Integration Points

### With Phase 2: Runtime Core
- **GC Integration**: Event loop must trigger GC when allocation fails
- **Value Operations**: Event handlers will manipulate RpsValue instances
- **Memory Zones**: Event loop data structures allocated in GC-managed zones

### With Phase 3: Advanced Features
- **Object Identity**: OID system depends on stable object headers
- **Persistence**: Value serialization requires complete value representation
- **Plugin System**: Plugin interface must work with GC-managed objects

### With Phase 4: User Interfaces
- **REPL Evaluation**: Expression evaluation depends on value operations
- **GUI Objects**: Visual objects are first-class RpsValue instances
- **Error Display**: Error handling integrated with value system

### Testing Strategy
- **Unit Tests**: Each data structure and algorithm thoroughly tested
- **Memory Safety**: AddressSanitizer and Valgrind equivalents
- **Performance Benchmarks**: Compare against C++ implementation
- **Stress Testing**: Memory pressure and concurrent access patterns

This engineering plan provides the foundation for implementing RefPerSys's most critical components in Zig, with careful consideration of performance, safety, and integration requirements.