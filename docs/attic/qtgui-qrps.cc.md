# Qt GUI Implementation Experiment (`attic/qtgui-qrps.cc`)

**File Path:** `attic/qtgui-qrps.cc`

## Overview

This file contains an experimental Qt-based graphical user interface implementation for RefPerSys. It was developed as an alternative to the FLTK-based GUI experiments, providing a more modern and feature-rich interface using the Qt framework. The implementation includes multiple windows, object browsing, command editing, and integration with the RefPerSys runtime.

## Historical Context

This Qt GUI experiment was part of RefPerSys's exploration of different GUI frameworks. It appears to have been a more complete implementation compared to the FLTK experiments, featuring:

- Multi-window support with object browsers
- Command-line interface integration
- Garbage collection coordination
- Real-time object inspection and manipulation

## Key Components

### RpsTemp_Application Class

The main Qt application class that manages the GUI lifecycle and coordinates with the RefPerSys runtime.

```cpp
class RpsTemp_Application : public QApplication {
    // Application-level operations
    void do_dump();           // Dump heap to disk
    void do_exit();           // Exit with heap dump
    void do_quit();           // Exit without dump
    void do_new_window();     // Create new GUI window
    void do_garbage_collect(); // Trigger GC
    void do_open_command();   // Open command window
    
    void xtra_gc_mark(Rps_GarbageCollector*); // GC integration
};
```

**Key Features:**
- Thread-safe operations using `RPSQT_WITH_LOCK()` macro
- Integration with RefPerSys garbage collection
- Window management and lifecycle coordination

### RpsTemp_MainWindow Class

Represents individual GUI windows containing object browsers and controls.

```cpp
class RpsTemp_MainWindow : public QMainWindow {
    static std::set<RpsTemp_MainWindow*> mainwin_set_; // All windows
    
    void create_menus();      // Build menu bar
    void fill_vbox();         // Populate window contents
    void do_enter_shown_object(); // Handle object input
    
    static void garbage_collect_all_main_windows(Rps_GarbageCollector*);
};
```

**Features:**
- Menu bar with App operations (Dump, Quit, Exit, New Window, GC, Command)
- Object browser for inspecting RefPerSys objects
- Input field for entering object IDs/names
- Automatic window ranking and management

### RpsTemp_ObjectBrowser Class

A Qt text browser widget specialized for displaying RefPerSys objects.

```cpp
class RpsTemp_ObjectBrowser : public QTextBrowser {
    struct shown_object_st {
        Rps_ObjectRef shob_obref;    // Object reference
        int shob_depth;              // Display depth
        std::string shob_subtitle;   // HTML subtitle
    };
    
    std::vector<shown_object_st> objbr_shownobvect; // Displayed objects
    std::unordered_map<Rps_ObjectRef, int> objbr_mapshownob; // Object lookup
    
    void show_one_object_in_frame(Rps_CallFrame*, shown_object_st&);
    void refresh_object_browser();
    bool refpersys_object_is_shown(Rps_ObjectRef, int* pix = nullptr);
    void add_shown_object(Rps_ObjectRef, std::string, int depth = 0);
};
```

**Capabilities:**
- HTML-based object display using `display_object_content_web` selector
- Configurable display depth for object inspection
- Multiple object display in single browser
- Garbage collection integration for displayed objects

### Input Widgets

#### RpsTemp_ObjectLineEdit
```cpp
class RpsTemp_ObjectLineEdit : public QLineEdit {
    RpsTemp_ObjectCompleter* oblined_completer;
};
```

A line edit widget for entering object identifiers with autocompletion.

#### RpsTemp_ObjectCompleter
```cpp
class RpsTemp_ObjectCompleter : public QCompleter {
    // Object name completion functionality
};
```

Provides autocompletion for object names and IDs.

#### RpsTemp_CommandEdit
```cpp
class RpsTemp_CommandEdit : public QTextEdit {
    void garbage_collect_command_edit(Rps_GarbageCollector*);
};
```

A text edit widget for entering RefPerSys commands.

## Integration with RefPerSys Runtime

### Garbage Collection Coordination

The GUI maintains proper integration with RefPerSys's garbage collector:

```cpp
void RpsTemp_Application::do_garbage_collect() {
    RPS_LOCALFRAME(/*descr:*/nullptr, /*callerframe:*/nullptr);
    
    _.set_additional_gc_marker([this](Rps_GarbageCollector* gc) {
        this->xtra_gc_mark(gc);
    });
    
    rps_garbage_collect(&gcfun);
}
```

**GC Integration Points:**
- Application-level GC triggering
- Per-window object marking
- Object browser content preservation
- Command edit state preservation

### Thread Safety

Extensive use of mutexes for thread-safe GUI operations:

```cpp
#define RPSQT_WITH_LOCK() \
    std::lock_guard<std::recursive_mutex> _qtguard_(rpsqt_mtx)
```

All GUI operations are protected by the global `rpsqt_mtx` mutex.

### Call Frame Integration

GUI operations create proper RefPerSys call frames:

```cpp
RPS_LOCALFRAME(/*descr:*/nullptr, /*callerframe:*/nullptr,
               Rps_ObjectRef showob; /* local variables */);
```

This ensures proper garbage collection and debugging integration.

## Object Display System

### Web-Based Display Protocol

Objects are displayed using a web-based protocol:

```cpp
// Send display_object_content_web selector
Rps_TwoValues two = Rps_Value(_f.showob).send2(&_,
    RPS_ROOT_OB(_02iWbXmFx8f04ldLRt), // "display_object_content_web"
    _f.strbufob,                       // String buffer
    Rps_Value::make_tagged_int(displaydepth)); // Display depth
```

**Display Process:**
1. Create string buffer object for output
2. Send `display_object_content_web` message to target object
3. Retrieve HTML content from buffer
4. Insert HTML into Qt text browser

### Object Browser Features

- **Multi-Object Display**: Show multiple objects simultaneously
- **Depth Control**: Configurable inspection depth
- **HTML Rendering**: Rich formatting with Qt's HTML support
- **Incremental Updates**: Refresh individual objects or entire browser

## Window Management

### Window Lifecycle

```cpp
RpsTemp_MainWindow::RpsTemp_MainWindow() {
    mainwin_set_.insert(this);
    mainwin_rank = mainwin_set_.size();
    
    // Connect destruction signal
    connect(this, &QObject::destroyed, this, [=](){
        mainwin_set_.erase(this);
        if (mainwin_set_.empty()) {
            rpsqt_app->exit();  // Exit when last window closes
        }
    });
}
```

**Features:**
- Automatic window ranking
- Proper cleanup on destruction
- Application exit when no windows remain

### Menu System

Comprehensive menu bar with application operations:

```cpp
void create_menus() {
    auto mbar = menuBar();
    auto appmenu = mbar->addMenu("App");
    
    // Dump, Quit, Exit, New Window, GC, Command actions
}
```

## Command Interface

### Command Window

```cpp
void RpsTemp_Application::do_open_command() {
    if (!_app_cmdwin) {
        _app_cmdwin = new QMainWindow();
        _app_cmdedit = new RpsTemp_CommandEdit();
        _app_cmdwin->setCentralWidget(_app_cmdedit);
    }
    _app_cmdwin->show();
}
```

Provides a separate window for command-line interaction with RefPerSys.

## Error Handling

### Input Validation

```cpp
void RpsTemp_MainWindow::do_enter_shown_object() {
    std::string obshowstring = mainwin_shownobject->text().toStdString();
    _f.showob = Rps_ObjectRef::find_object_or_null_by_string(&_, obshowstring);
    
    if (!_f.showob) {
        // Show error dialog for invalid input
        QMessageBox::warning(this, "no object for window", 
                           "Input is not a valid RefPerSys object");
        mainwin_shownobject->clear();
        return;
    }
    // Process valid object...
}
```

### Exception Safety

All GUI operations include proper exception handling and logging:

```cpp
RPS_DEBUG_LOG(GUI, "Operation completed successfully");
RPS_WARNOUT("Incomplete implementation: " << __FUNCTION__);
```

## Performance Considerations

### GUI Responsiveness

- **Asynchronous Updates**: Uses `QTimer::singleShot()` for delayed operations
- **Thread Coordination**: Proper synchronization with RefPerSys runtime
- **Incremental Display**: Objects displayed individually to avoid blocking

### Memory Management

- **Qt Object Ownership**: Proper parent-child relationships
- **RefPerSys GC Integration**: GUI objects participate in garbage collection
- **Resource Cleanup**: Automatic cleanup on window destruction

## Limitations and Incomplete Features

### Marked Incomplete Sections

```cpp
#warning incomplete RpsTemp_MainWindow::RpsTemp_MainWindow constructor
#warning incomplete RpsTemp_ObjectBrowser::RpsTemp_ObjectBrowser constructor
#warning for some reason this Qt warning appears twice.
```

The implementation includes several warnings about incomplete functionality, indicating this was experimental code.

### Missing Features

- Full command interpretation in command window
- Advanced object manipulation capabilities
- Integration with RefPerSys REPL
- Comprehensive error recovery

## Integration Points

### Build System

Requires Qt5 development libraries and proper compilation flags:

```bash
# Example compilation
g++ -std=c++17 qtgui-qrps.cc -lQt5Widgets -lQt5Core -lQt5Gui
```

### RefPerSys Integration

Activated via `-Q` command-line argument:

```cpp
void rps_qtgui_init_progarg(int &argc, char**argv) {
    rpsqt_app = new RpsTemp_Application(argc, argv);
    // Create initial window and setup
}

void rps_qtgui_run() {
    rpsqt_app->exec();  // Run Qt event loop
}
```

## Comparison with Other GUI Experiments

### vs. FLTK (`guifltk_rps.cc`)

**Qt Advantages:**
- More modern and feature-rich widget set
- Better HTML rendering capabilities
- More comprehensive object browser
- Better integration with C++ standard library

**Qt Disadvantages:**
- Heavier dependencies
- More complex build requirements
- Larger runtime footprint

### vs. GTKmm (`guigtkmm_rps.cc`)

**Similarities:**
- Both are C++ GUI bindings
- Similar architecture and patterns
- Integration with RefPerSys runtime

**Qt Specific Features:**
- Better documentation and community support
- More consistent API design
- Superior HTML/text rendering

## Future Development Potential

This Qt GUI implementation demonstrates significant potential for RefPerSys GUI development:

1. **Object Inspection**: Rich object browsing capabilities
2. **Command Integration**: Foundation for graphical command interface
3. **Multi-Window Support**: Proper window management
4. **GC Integration**: Proper memory management coordination

The implementation provides a solid foundation for a graphical interface to RefPerSys, with room for expansion into more advanced features like object editing, visual programming interfaces, and integrated development tools.

## File Status

**Status:** Experimental/Abandoned
**Date:** 2021-2022
**Reason for attic:** Likely replaced by other GUI approaches or deemed too complex for the project's needs at the time.

This Qt GUI experiment represents one of the more sophisticated attempts at providing a graphical interface for RefPerSys, showcasing good integration with the runtime system and modern GUI design principles.