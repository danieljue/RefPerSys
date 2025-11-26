# RefPerSys Core Capabilities Design Document

## Overview

RefPerSys is a homoiconic, reflective, persistent symbolic artificial intelligence system designed as an inference engine. This document outlines its core capabilities in a language-agnostic manner, suitable for implementation in systems programming languages like C or Zig.

## 1. Homoiconic Object System

### Core Concept
RefPerSys treats code and data uniformly as objects, enabling self-modifying and self-inspecting programs.

### Key Components

#### Object Identity System
- **128-bit unique object identifiers** (OIDs) for persistent identity
- Base-62 string encoding for human-readable representation
- Cryptographically strong random OID generation
- Hash-based lookup tables for efficient object retrieval

#### Object Structure
- **Mutable objects** with per-object recursive mutexes for thread safety
- **Attributes**: Key-value associations (object → value)
- **Components**: Ordered sequence of arbitrary values
- **Payload**: Optional additional data (string buffers, collections, metadata)

#### Class System
- **Single inheritance hierarchy** with root classes
- **Dynamic class creation** at runtime
- **Method dispatch** through class hierarchy traversal
- **Meta-object protocol** for runtime introspection and modification

## 2. Memory Management and Garbage Collection

### Precise Multi-threaded Garbage Collector

#### Memory Zones
- **Small blocks**: 8MB zones for frequent allocations
- **Large blocks**: 64MB zones for bigger objects
- **mmap-based allocation** for direct memory mapping
- **Zone tracking** for garbage collection

#### Collection Algorithm
- **Mark-and-sweep** with tri-color marking
- **Generational collection** (young vs old objects)
- **Write barriers** to track object modifications
- **Concurrent mutator support** with thread coordination

#### Call Frame Integration
- **Explicit call frames** for local variable management
- **GC root registration** through frame linking
- **Write barrier calls** after object modifications
- **Periodic collection triggers** every few milliseconds

## 3. Value System

### Universal Value Representation
- **Single-word tagged union** for all value types
- **Type tags** for runtime type discrimination
- **Automatic memory management** for heap-allocated values

### Supported Value Types
- **Integers**: Tagged integers for small values
- **Floats**: Boxed double-precision floating point
- **Strings**: Immutable UTF-8 strings with lazy hashing
- **Objects**: References to mutable objects
- **Sets**: Immutable sets of object references
- **Tuples**: Immutable sequences of values
- **Closures**: First-class functions with captured environment
- **Instances**: Objects instantiated from classes

## 4. Code Generation and Execution

### Multi-strategy Code Generation

#### GCC JIT Integration
- **High-quality optimized code** generation
- **Runtime compilation** of generated C code
- **Integration with system compiler** toolchain

#### GNU Lightning Integration
- **Fast code generation** for simple cases
- **Machine code emission** at runtime
- **Register allocation** and instruction selection

#### C++ Template Generation
- **Template-based code generation**
- **Plugin compilation** workflow
- **Dynamic shared object loading**

### Execution Model
- **Closure application** with argument passing
- **Method dispatch** through selector lookup
- **Stack-based evaluation** with call frame management

## 5. Persistent Storage Engine

### JSON-Based Serialization

#### Space-Based Organization
- **Named spaces** for object grouping
- **Manifest files** tracking space contents
- **Atomic operations** for consistency
- **Incremental updates** and snapshots

#### Two-Pass Loading
- **First pass**: Object skeleton creation
- **Second pass**: Relationship restoration
- **Forward references** resolution
- **Version compatibility** handling

### Dump Process
- **Object graph traversal** with cycle detection
- **JSON serialization** of all values
- **Space file updates** with atomic commits
- **Manifest regeneration** for consistency

## 6. Plugin System

### Dynamic Extension Architecture

#### Plugin Loading
- **Shared object loading** via dlopen interface
- **Symbol resolution** for initialization functions
- **Plugin registration** with system registry
- **Argument passing** to plugin initialization

#### Plugin Interface
- **Standardized entry points** for initialization
- **Class registration** capabilities
- **Method installation** in existing classes
- **System extension** through new operations

## 7. Concurrency Model

### Agenda-Based Task Scheduling

#### Tasklet System
- **Small executable units** (milliseconds execution time)
- **Priority queue** for task ordering
- **Worker thread pool** (configurable size 3-24 threads)
- **Task continuation** and chaining

#### Event Loop Integration
- **File descriptor monitoring** with poll/select
- **Signal handling** integration
- **Timer management** for scheduled tasks
- **I/O event processing**

### Thread Safety
- **Per-object mutexes** for data protection
- **Atomic operations** for performance-critical paths
- **Lock ordering** to prevent deadlocks
- **Concurrent GC** coordination

## 8. User Interface Layer

### Multiple Interface Modes

#### REPL Interface
- **Command-line interaction** with expression evaluation
- **Object manipulation** commands
- **Debug and inspection** capabilities
- **Script execution** support

#### Graphical Interface (FLTK)
- **Windowed environment** with GUI components
- **Object browser** and inspector
- **Integrated REPL console**
- **Debug visualization**

#### JSONRPC Interface
- **Headless operation** with external communication
- **Named pipe (FIFO)** based messaging
- **Structured JSON** message format
- **Remote GUI** support

#### Batch Processing
- **Non-interactive execution**
- **Automated task processing**
- **Script and command execution**
- **Exit status reporting**

## 9. Reflection and Meta-programming

### Meta-Object Protocol

#### Runtime Introspection
- **Object structure examination**
- **Class hierarchy traversal**
- **Method lookup and dispatch**
- **Attribute and component access**

#### Dynamic Modification
- **Class creation and alteration**
- **Method installation and removal**
- **Attribute manipulation**
- **Object relationship changes**

### Self-Referential Capabilities
- **Code as data** representation
- **Program modification** at runtime
- **Automatic code generation**
- **System evolution** capabilities

## 10. System Architecture

### Bootstrap and Initialization

#### Startup Sequence
1. **Environment validation** and setup
2. **Memory system initialization**
3. **Persistent state loading**
4. **Plugin system activation**
5. **User interface startup**
6. **Main execution loop**

#### Configuration Management
- **Environment variables** for system paths
- **Preference files** for user settings
- **Command-line arguments** for runtime configuration
- **Dynamic reconfiguration** capabilities

### Error Handling and Debugging

#### Comprehensive Debugging
- **Multiple debug categories** (GC, REPL, load, dump, etc.)
- **Stack trace generation** with libbacktrace
- **Memory debugging** and leak detection
- **Performance profiling** and statistics

#### Error Recovery
- **Graceful degradation** on subsystem failures
- **Plugin isolation** to prevent system crashes
- **State consistency** checking and repair
- **Logging and monitoring** capabilities

## Implementation Considerations

### Language Requirements
- **Manual memory management** with custom GC integration
- **Threading support** with mutexes and atomic operations
- **Dynamic loading** capabilities (dlopen equivalent)
- **JIT compilation** support (optional but beneficial)

### Performance Characteristics
- **Memory efficient** with zone-based allocation
- **Concurrent** with controlled thread pools
- **Persistent** with incremental state management
- **Extensible** through plugin architecture

### Portability Considerations
- **Linux-specific** features (mmap, dlopen, threads)
- **64-bit architecture** requirement
- **GCC toolchain** dependency for JIT features
- **UTF-8** text encoding throughout

This design document provides a comprehensive overview of RefPerSys's core capabilities, structured to be implementable in systems programming languages while maintaining the system's sophisticated symbolic AI and reflective programming features.