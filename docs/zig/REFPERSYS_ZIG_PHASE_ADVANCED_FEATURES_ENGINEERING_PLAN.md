# RefPerSys Zig Phase 3: Advanced Features Engineering Plan

## Overview

Phase 3 implements the advanced capabilities that make RefPerSys a powerful symbolic AI system: persistent storage, dynamic plugin loading, and runtime code generation. These features build upon the runtime core to provide extensibility, durability, and computational flexibility.

## Data Structures

### 1. Persistence Engine

#### Space
**Purpose**: Named container for object collections with transactional semantics
**Key Components**:
- `name`: Unique space identifier
- `objects`: HashMap of OID to object references
- `manifest`: Metadata about space contents
- `version`: Space version for consistency checking

**Zig-Specific Optimizations**:
- `std.StringHashMap` for object storage
- `std.json` for manifest serialization
- `std.fs` for atomic file operations

**Performance Considerations**:
- Lazy loading of objects
- Incremental updates to reduce I/O
- Memory-mapped access for large spaces

#### Dumper
**Purpose**: Object graph serialization coordinator
**Key Components**:
- `spaces`: Array of spaces to dump
- `object_map`: OID to JSON mapping
- `temp_dir`: Temporary directory for atomic writes
- `manifest`: Global manifest structure

**Zig-Specific Optimizations**:
- `std.ArrayList` for dynamic space management
- `std.fs.selfExePathAlloc` for executable-relative paths
- `std.json.WriteStream` for efficient JSON generation

#### Loader
**Purpose**: Object graph deserialization and reconstruction
**Key Components**:
- `spaces`: Map of loaded spaces
- `object_cache`: OID to object mapping for forward references
- `manifest`: Parsed manifest data
- `phase`: Two-phase loading state

**Zig-Specific Optimizations**:
- `std.AutoHashMap` for object caching
- `std.json.Parser` for manifest processing
- `std.mem.Allocator` for allocation management

### 2. Plugin System

#### Plugin
**Purpose**: Dynamically loaded extension module
**Key Components**:
- `handle`: Dynamic library handle
- `name`: Plugin identifier
- `init_function`: Initialization callback
- `metadata`: Version and dependency information

**Zig-Specific Optimizations**:
- `std.DynLib` for cross-platform dynamic loading
- `std.meta` for type-safe function loading
- Compile-time plugin interface validation

**Performance Considerations**:
- Lazy symbol resolution
- Minimal overhead for loaded plugins
- Efficient plugin discovery and loading

#### PluginManager
**Purpose**: Plugin lifecycle and dependency management
**Key Components**:
- `loaded_plugins`: Map of loaded plugin instances
- `plugin_paths`: Search paths for plugin discovery
- `dependencies`: Plugin dependency graph
- `registry`: Extension point registry

**Zig-Specific Optimizations**:
- `std.StringHashMap` for plugin storage
- `std.fs.selfExePath` for relative plugin paths
- `std.meta` for plugin interface reflection

#### ExtensionPoint
**Purpose**: Plugin extension registration system
**Key Components**:
- `name`: Extension point identifier
- `interface`: Required function signatures
- `implementations`: Array of registered implementations
- `priority`: Ordering for multiple implementations

**Zig-Specific Optimizations**:
- `comptime` interface definition and validation
- `std.ArrayList` for implementation storage
- `std.sort` for priority-based ordering

### 3. Code Generation

#### JITCompiler
**Purpose**: Runtime code generation and compilation
**Key Components**:
- `context`: LLVM context for code generation
- `module`: Current compilation module
- `builder`: IR builder for instruction generation
- `engine`: Execution engine for compiled code

**Zig-Specific Optimizations**:
- C ABI compatibility with LLVM C API
- `std.c` for C library integration
- `comptime` code generation templates

**Performance Considerations**:
- Lazy compilation of hot code paths
- Caching of compiled functions
- Optimization level tuning

#### CodeGenerator
**Purpose**: High-level code generation coordinator
**Key Components**:
- `templates`: Code generation templates
- `symbols`: Symbol table for generated code
- `types`: Type information for code generation
- `output`: Generated code buffer

**Zig-Specific Optimizations**:
- `comptime` string concatenation
- `std.fmt` for code formatting
- `std.meta` for type reflection

#### FunctionBuilder
**Purpose**: Helper for building function definitions
**Key Components**:
- `signature`: Function signature specification
- `body`: Instruction sequence
- `locals`: Local variable management
- `labels`: Jump target management

**Zig-Specific Optimizations**:
- `std.ArrayList` for instruction sequences
- `std.StringHashMap` for symbol management
- Type-safe instruction building

## Algorithms

### 1. Persistence Operations

#### Two-Phase Loading
**Algorithm Overview**:
1. **Phase 1**: Parse manifests and create object skeletons
2. **Phase 2**: Resolve references and populate object contents
3. **Validation**: Verify object graph consistency

**Zig Implementation Strategy**:
```zig
// High-level algorithm structure
fn loadSpaces(allocator: std.mem.Allocator, load_dir: []const u8) !SpaceMap {
    var spaces = SpaceMap.init(allocator);
    errdefer spaces.deinit();

    // Phase 1: Parse manifests and create skeletons
    try parseManifests(&spaces, load_dir);

    // Phase 2: Resolve references and populate
    try resolveReferences(&spaces);

    // Validation
    try validateObjectGraph(&spaces);

    return spaces;
}

fn parseManifests(spaces: *SpaceMap, load_dir: []const u8) !void {
    // Read global manifest
    const manifest_path = try std.fs.path.join(allocator, &[_][]const u8{load_dir, "manifest.json"});
    defer allocator.free(manifest_path);

    const manifest_content = try std.fs.cwd().readFileAlloc(allocator, manifest_path, max_file_size);
    defer allocator.free(manifest_content);

    // Parse JSON and create space skeletons
    var parser = std.json.Parser.init(allocator, false);
    defer parser.deinit();

    var tree = try parser.parse(manifest_content);
    defer tree.deinit();

    // Process each space
    var space_iter = tree.root.Object.iterator();
    while (space_iter.next()) |entry| {
        const space_name = entry.key_ptr.*;
        const space_manifest = entry.value_ptr.*;

        var space = try Space.init(allocator, space_name);
        errdefer space.deinit();

        try spaces.put(space_name, space);
    }
}
```

**Performance Considerations**:
- Streaming JSON parsing for large manifests
- Parallel loading of independent spaces
- Memory-mapped file access for performance

#### Atomic Dumping
**Algorithm Overview**:
1. Create temporary directory for new dump
2. Serialize all spaces to temporary files
3. Update manifest atomically
4. Move temporary files to final location

**Zig Implementation Strategy**:
```zig
// High-level algorithm structure
fn dumpSpaces(allocator: std.mem.Allocator, spaces: SpaceMap, dump_dir: []const u8) !void {
    // Create temporary directory
    const temp_dir = try std.fs.selfExePathAlloc(allocator, &[_][]const u8{dump_dir, "temp_dump"});
    defer allocator.free(temp_dir);

    try std.fs.cwd().makeDir(temp_dir);

    // Serialize spaces to temporary files
    var space_iter = spaces.iterator();
    while (space_iter.next()) |entry| {
        const space_name = entry.key_ptr.*;
        const space = entry.value_ptr.*;

        try dumpSpace(space, temp_dir, space_name);
    }

    // Update manifest
    try updateManifest(spaces, temp_dir);

    // Atomic move to final location
    const final_dir = try std.fs.selfExePathAlloc(allocator, &[_][]const u8{dump_dir, "spaces"});
    defer allocator.free(final_dir);

    try std.fs.rename(temp_dir, final_dir);
}
```

### 2. Plugin Management

#### Dynamic Loading
**Algorithm Overview**:
1. Locate plugin file using search paths
2. Load dynamic library with dlopen equivalent
3. Resolve initialization function symbol
4. Call initialization with plugin context
5. Register plugin capabilities

**Zig Implementation Strategy**:
```zig
// High-level algorithm structure
fn loadPlugin(allocator: std.mem.Allocator, name: []const u8, search_paths: []const []const u8) !Plugin {
    // Find plugin file
    const plugin_path = try findPluginFile(allocator, name, search_paths);
    defer allocator.free(plugin_path);

    // Load library
    var lib = try std.DynLib.open(plugin_path);
    errdefer lib.close();

    // Resolve init function
    const init_fn = lib.lookup(*const fn (*PluginContext) callconv(.C) void, "rps_plugin_init") orelse
        return error.MissingInitFunction;

    // Create plugin context
    var context = PluginContext{
        .allocator = allocator,
        .name = try allocator.dupe(u8, name),
        .lib = lib,
    };

    // Initialize plugin
    init_fn(&context);

    return Plugin{
        .name = name,
        .lib = lib,
        .context = context,
    };
}
```

**Performance Considerations**:
- Plugin loading on-demand
- Symbol resolution caching
- Minimal runtime overhead for loaded plugins

#### Extension Registration
**Algorithm Overview**:
1. Plugin declares extension points it implements
2. Validate interface compatibility
3. Register implementations with priority
4. Update extension dispatch tables

**Zig Implementation Strategy**:
```zig
// High-level algorithm structure
fn registerExtension(manager: *PluginManager, extension_name: []const u8, impl: ExtensionImpl, priority: i32) !void {
    // Validate interface
    const extension_point = manager.extension_points.get(extension_name) orelse
        return error.UnknownExtensionPoint;

    try validateInterface(impl, extension_point.interface);

    // Register implementation
    const entry = ExtensionEntry{
        .implementation = impl,
        .priority = priority,
    };

    try extension_point.implementations.append(entry);

    // Sort by priority
    std.sort.sort(ExtensionEntry, extension_point.implementations.items, {}, comparePriority);
}
```

### 3. Code Generation

#### LLVM IR Generation
**Algorithm Overview**:
1. Create LLVM context and module
2. Define function signatures
3. Generate IR instructions
4. Optimize and compile to machine code

**Zig Implementation Strategy**:
```zig
// High-level algorithm structure
fn generateFunction(compiler: *JITCompiler, signature: FunctionSignature, body_gen: BodyGenerator) !*const fn () callconv(.C) void {
    // Create LLVM function
    const func_type = try createFunctionType(compiler.context, signature);
    const function = LLVMAddFunction(compiler.module, signature.name.ptr, func_type);

    // Create entry block
    const entry_block = LLVMAppendBasicBlock(function, "entry");
    LLVMPositionBuilderAtEnd(compiler.builder, entry_block);

    // Generate function body
    try body_gen(compiler, function);

    // Verify and optimize
    try verifyFunction(function);
    try optimizeFunction(compiler, function);

    // JIT compile
    return try compileToMachineCode(compiler, function);
}
```

**Performance Considerations**:
- Lazy compilation of hot functions
- Caching of compiled code
- Optimization level tuning based on usage patterns

## References

### Inspiration from Existing Codebase

#### Persistence Engine
- **REFPERSYS_PERSISTENT_STORAGE_ENGINE_ANALYSIS.md**: Detailed persistence analysis
- **load_rps.cc**: Two-phase loading implementation
- **dump_rps.cc**: Object serialization algorithms
- **Rps_Loader** and **Rps_Dumper**: Core persistence classes

#### Plugin System
- **REFPERSYS_PLUGIN_LOADING_MECHANISM_ANALYSIS.md**: Plugin architecture analysis
- **Plugin loading patterns**: dlopen/dlsym usage
- **rps_plugins_vector**: Plugin registry implementation
- **Rps_Plugin**: Plugin structure definition

#### Code Generation
- **REFPERSYS_CODE_GENERATION_PIPELINE_ANALYSIS.md**: Code generation analysis
- **cppgen_rps.cc**: C++ code generation implementation
- **gccjit_rps.cc**: GCC JIT integration
- **lightgen_rps.cc**: GNU Lightning code generation

### Zig-Specific Adaptations

#### JSON Processing
- `std.json` instead of jsoncpp
- Streaming parser for large manifests
- Compile-time JSON schema validation

#### Dynamic Loading
- `std.DynLib` instead of dlfcn.h
- Cross-platform library loading
- Type-safe function resolution

#### LLVM Integration
- C ABI compatibility with LLVM C API
- `std.c` for C library integration
- Error handling with Zig's error unions

## Integration Points

### With Phase 1: Foundation
- **Memory Management**: Persistence uses GC-managed allocators
- **Value System**: Serialization depends on complete RpsValue representation
- **Object Model**: Persistence operates on object graphs

### With Phase 2: Runtime Core
- **Event Loop**: Persistence I/O coordinated through event loop
- **Agenda System**: Code generation and plugin loading as agenda tasks
- **Threading**: Plugin operations thread-safe

### With Phase 4: User Interfaces
- **REPL**: Plugin commands and code generation integration
- **GUI**: Plugin-provided UI components
- **Persistence**: UI state saving/loading

### With Phase 5: Ecosystem
- **Build System**: Plugin compilation integration
- **Testing**: Plugin and persistence testing frameworks
- **Packaging**: Plugin distribution mechanisms

### Testing Strategy
- **Persistence Tests**: Load/save cycles, data integrity, performance
- **Plugin Tests**: Loading/unloading, interface compliance, error handling
- **Code Generation Tests**: Correctness, performance, optimization
- **Integration Tests**: Plugin + persistence + code generation
- **Cross-Platform Tests**: Different operating systems and architectures

This engineering plan establishes the advanced capabilities that make RefPerSys extensible and powerful, providing the foundation for a rich ecosystem of plugins and persistent applications.