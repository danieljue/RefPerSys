# RefPerSys Zig Reimplementation Roadmap

## Overview

This document outlines the prioritized development roadmap for rebuilding RefPerSys from scratch using Zig. It serves as the precursor to detailed architectural and engineering documents, establishing the foundation for a clean, modern implementation that leverages Zig's safety, performance, and low-level control capabilities.

## Development Philosophy

### Core Principles
- **Safety First**: Leverage Zig's compile-time safety and optional runtime safety
- **Performance**: Match or exceed C++ performance with zero-cost abstractions
- **Modularity**: Clean separation of concerns with minimal interdependencies
- **Testability**: Comprehensive unit testing from day one
- **Maintainability**: Self-documenting code with clear APIs

### Development Approach
- **Incremental**: Build and test each component independently
- **Bottom-up**: Start with foundational components, build upward
- **Parallel**: Develop independent subsystems concurrently
- **Validated**: Each phase must pass comprehensive test suites

## Phase 1: Foundation (Weeks 1-12)

### Priority 1: Memory Management & GC (Weeks 1-6)
**Rationale**: Garbage collection is the most complex and fundamental component. Must be rock-solid before building anything else.

#### 1.1 Core Memory Infrastructure
- **Manual memory management foundation**
- **mmap-based zone allocation** (`QuasiZone`, `MemoryZone`)
- **Arena allocators** for different allocation patterns
- **Memory safety primitives** (bounds checking, overflow detection)

#### 1.2 Basic GC Infrastructure
- **Mark-and-sweep collector** (single-threaded first)
- **Root tracking** (stack scanning, global roots)
- **Object header management**
- **Basic allocation/deallocation**

#### 1.3 Write Barriers & Generational GC
- **Card marking system**
- **Write barrier implementation**
- **Young/old generation separation**
- **Incremental collection**

#### 1.4 Multi-threaded GC
- **Concurrent mutator support**
- **Thread coordination** (stop-the-world, concurrent phases)
- **Call frame scanning** across threads
- **Performance optimization**

**Testing**: Comprehensive unit tests for allocation patterns, GC correctness, memory safety, and performance benchmarks.

### Priority 2: Value System & Object Model (Weeks 7-12)
**Rationale**: Core data structures that everything else depends on.

#### 2.1 Tagged Union Value System
- **RpsValue** tagged union (integers, floats, objects, etc.)
- **Type safety** with compile-time checks
- **Value operations** (arithmetic, comparison, conversion)

#### 2.2 Object Identity & OID System
- **128-bit OID generation** (cryptographically strong)
- **OID hash maps** and lookup tables
- **Persistent identity** across sessions

#### 2.3 Basic Object System
- **RpsObject** structure with headers and payloads
- **Class system** (single inheritance)
- **Attribute storage** (mutable associative arrays)
- **Component vectors**

#### 2.4 Meta-Object Protocol
- **Runtime introspection**
- **Dynamic method dispatch**
- **Class modification** capabilities

**Testing**: Value operations, object lifecycle, identity preservation, meta-protocol functionality.

## Phase 2: Runtime Core (Weeks 13-26)

### Priority 3: Event Loop System (Weeks 13-18)
**Rationale**: Main execution loop - critical for responsiveness and I/O handling.

#### 3.1 POSIX System Integration
- **Direct system calls** (`poll`, `signalfd`, `timerfd`, `pipe`)
- **Signal handling** infrastructure
- **File descriptor management**

#### 3.2 Event Loop Core
- **Poll-based multiplexing** (up to 128 FDs)
- **Event handler registration**
- **Timeout management**
- **Self-pipe communication**

#### 3.3 Signal & Timer Integration
- **Signal file descriptors**
- **Timer file descriptors**
- **Event prioritization**
- **Error handling**

#### 3.4 JSONRPC Support
- **FIFO communication** setup
- **Message parsing** and dispatch
- **External GUI integration**

**Testing**: Event handling correctness, signal processing, timer accuracy, JSONRPC communication.

### Priority 4: Concurrency & Agenda System (Weeks 19-26)
**Rationale**: Multi-threading foundation for task execution.

#### 4.1 Thread Management
- **Worker thread pool** (configurable 3-24 threads)
- **Thread lifecycle** management
- **Thread-local storage**

#### 4.2 Agenda Implementation
- **Priority queues** (High/Normal/Low)
- **Tasklet scheduling**
- **Thread-safe operations**

#### 4.3 Coordination Mechanisms
- **Condition variables** and mutexes
- **Atomic operations**
- **GC thread coordination**

#### 4.4 Performance Optimization
- **Lock contention minimization**
- **Work stealing** algorithms
- **Load balancing**

**Testing**: Thread safety, deadlock prevention, performance scaling, task execution correctness.

## Phase 3: Advanced Features (Weeks 27-42)

### Priority 5: Persistence Engine (Weeks 27-32)
**Rationale**: Data durability - essential for practical usage.

#### 5.1 JSON Serialization
- **Object graph traversal**
- **Value serialization**
- **Cycle detection**
- **Type preservation**

#### 5.2 Space-Based Storage
- **Named spaces** for organization
- **Incremental updates**
- **Version compatibility**
- **Atomic commits**

#### 5.3 Loading System
- **Two-pass loading** (skeleton then relationships)
- **Forward references** resolution
- **Error recovery**

#### 5.4 Performance Optimization
- **Lazy loading**
- **Caching strategies**
- **Memory mapping** for large datasets

**Testing**: Serialization correctness, loading fidelity, performance benchmarks, error handling.

### Priority 6: Plugin Architecture (Weeks 33-38)
**Rationale**: Extensibility - enables ecosystem growth.

#### 6.1 Dynamic Loading
- **Shared library loading** (`dlopen` equivalent)
- **Symbol resolution**
- **Plugin lifecycle** management

#### 6.2 Plugin Interface
- **Standardized APIs**
- **Type safety** across plugin boundaries
- **Error isolation**

#### 6.3 Plugin Discovery
- **Plugin registration**
- **Dependency management**
- **Version compatibility**

#### 6.4 Security
- **Sandboxing** capabilities
- **Resource limits**
- **Access control**

**Testing**: Plugin loading/unloading, API compatibility, error isolation, security boundaries.

### Priority 7: Code Generation (Weeks 39-42)
**Rationale**: JIT capabilities - advanced feature requiring other systems.

#### 7.1 LLVM Integration
- **Basic LLVM IR** generation
- **Compilation** to machine code
- **Runtime execution**

#### 7.2 Code Generation Pipeline
- **Template instantiation**
- **Optimization passes**
- **Error handling**

#### 7.3 Alternative Backends
- **Interpreter fallback**
- **External compiler** integration
- **Cross-compilation** support

**Testing**: Code generation correctness, performance benchmarks, error handling.

## Phase 4: User Interfaces (Weeks 43-52)

### Priority 8: REPL Interface (Weeks 43-46)
**Rationale**: Primary user interaction - essential for development and debugging.

#### 8.1 Command Processing
- **Expression evaluation**
- **Object manipulation**
- **Debug commands**

#### 8.2 Terminal Integration
- **ANSI escape codes**
- **Line editing**
- **History management**

#### 8.3 Error Handling
- **Parse error reporting**
- **Runtime error display**
- **Recovery mechanisms**

**Testing**: Command execution, error handling, terminal compatibility.

### Priority 9: GUI Integration (Weeks 47-52)
**Rationale**: Visual interface - important for user adoption.

#### 9.1 Cross-Platform GUI
- **Native bindings** (GTK, Qt, or custom)
- **Event handling**
- **Widget system**

#### 9.2 RefPerSys Integration
- **Object visualization**
- **Interactive debugging**
- **Real-time monitoring**

#### 9.3 Performance
- **Efficient rendering**
- **Low latency** updates
- **Memory management**

**Testing**: GUI responsiveness, integration correctness, cross-platform compatibility.

## Phase 5: Ecosystem & Distribution (Weeks 53-60)

### Priority 10: Build System & Packaging
- **Zig build integration**
- **Cross-platform builds**
- **Package management**
- **Distribution packaging**

### Priority 11: Testing Framework
- **Comprehensive test suites**
- **Continuous integration**
- **Performance regression** detection
- **Fuzz testing**

### Priority 12: Documentation & Examples
- **API documentation**
- **Tutorial development**
- **Plugin development** guides
- **Migration guides**

## Files Not Expected in Zig Implementation

### C++ Compiler Specific Files
- **`GNUmakefile`**: Replaced by `build.zig`
- **`refpersys.hh`**: Split into multiple Zig modules
- **`generated/rps-*.hh`**: Generated Zig code instead
- **`_config-refpersys.mk`**: Zig build configuration
- **`Make-dependencies/`**: Zig dependency management

### GUI Framework Specific Files
- **`fltk_rps.cc`**: Native Zig GUI bindings
- **`docs/attic/qtgui-qrps.cc.md`**: Qt integration not planned initially
- **Browser plugins**: WebAssembly support instead

### Legacy/Experimental Code
- **`attic/` directory**: Historical implementations
- **`oldplugins/`**: Legacy plugin interfaces
- **`docs/attic/`**: Outdated documentation
- **Bismon references**: Clean break from predecessor

### Platform-Specific Generated Code
- **`generated/rpsdata_GNU_Linux_*.h`**: Zig cross-compilation
- **Compiler-specific workarounds**: Zig's portability
- **Assembly code**: Zig's inline assembly or intrinsics

### Build Tools & Scripts
- **`do-build-refpersys-plugin.cc`**: Zig build system
- **`rps-generate-gitid.sh`**: Zig build script
- **`create-refpersys-class.sh`**: Code generation tools
- **Autotools files**: Modern Zig build system

## Development Milestones

### Month 3: GC & Values Working
- Basic allocation/deallocation
- Value system operations
- Unit test coverage >90%

### Month 6: Runtime Core Complete
- Event loop functional
- Multi-threading working
- Basic persistence operational

### Month 9: Advanced Features
- Plugin system functional
- Code generation working
- REPL interface complete

### Month 12: Production Ready
- Full feature set implemented
- Comprehensive testing
- Documentation complete
- Performance benchmarks met

## Risk Assessment

### High Risk
- **Custom GC Implementation**: Most complex component
- **JIT Code Generation**: LLVM integration challenges
- **Thread Coordination**: Race condition prevention

### Medium Risk
- **Plugin ABI Stability**: Cross-version compatibility
- **Performance Optimization**: Meeting C++ performance
- **Cross-Platform Support**: POSIX vs Windows differences

### Low Risk
- **Value System**: Straightforward tagged union implementation
- **Event Loop**: Direct POSIX system call mapping
- **Persistence**: JSON processing well-supported

## Success Criteria

### Functional Completeness
- All core RefPerSys capabilities implemented
- API compatibility with C++ version (where applicable)
- Plugin ecosystem support

### Performance Targets
- Memory usage: ≤110% of C++ version
- Runtime performance: ≥90% of C++ version
- Startup time: ≤120% of C++ version

### Quality Metrics
- Test coverage: >95%
- Memory safety: Zero leaks, no undefined behavior
- Thread safety: No race conditions or deadlocks
- Code maintainability: Clear documentation, modular design

This roadmap provides the foundation for a successful Zig reimplementation, prioritizing stability and correctness over feature completeness in early phases.