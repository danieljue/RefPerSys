# FLTK GUI Window Implementation (`attic/fltkwindow_rps.cc`)

**File Path:** `attic/fltkwindow_rps.cc`

## Overview

This file contains the Fast Light Toolkit (FLTK) graphical user interface implementation for RefPerSys, providing window management, menu systems, and widget infrastructure. It implements a complete GUI framework with command windows, output windows, menu bars, and widget type identification, serving as an alternative to the Qt-based GUI implementation.

## File Purpose

### FLTK GUI Framework Implementation
The file enables FLTK-based graphical interfaces by:

- **Window Management:** Creating and managing GUI windows with RefPerSys integration
- **Menu Systems:** Implementing menu bars with application controls
- **Widget Type System:** Providing widget identification and debugging support
- **Command Interface:** Building command windows for user interaction
- **Output Display:** Creating output windows for result visualization
- **Event Handling:** Managing GUI events and user interactions

## Implementation Status

### Current State
**Status:** Mostly complete FLTK GUI implementation with some unimplemented features

### Design Intent
- **FLTK Integration:** Complete FLTK-based GUI framework for RefPerSys
- **Window Lifecycle:** Full window creation, management, and destruction
- **Widget System:** Comprehensive widget type identification and handling
- **Menu Interface:** Application menu system with standard operations
- **Debug Support:** Extensive debugging and logging capabilities

## Dependencies

### Required Libraries
```cpp
#include "headfltk_rps.hh"  // FLTK-specific headers
```

**FLTK Components:**
- `Fl_Double_Window`: Double-buffered windows
- `Fl_Menu_Bar`: Menu bar widgets
- `Fl_Pack`: Container widgets
- `Fl_Text_Buffer`: Text buffer management
- `Fl_Text_Editor`: Text editing widgets

## Core Classes

### RpsGui_ShowWidget Class

Widget display and debugging support:

```cpp
class RpsGui_ShowWidget {
    Fl_Widget* shown_widget;
public:
    void output(std::ostream* pout) const;
};
```

**Widget Type Identification:**
- **RefPerSys Widgets:** Command windows, output windows, menu bars, packs
- **FLTK Widgets:** Buttons, browsers, counters, dials, inputs, outputs, clocks
- **Type Enumeration:** Comprehensive widget type mapping
- **Debug Output:** Formatted widget information with addresses and labels

### RpsGui_Window Class

Base window class with RefPerSys integration:

```cpp
class RpsGui_Window : public Fl_Double_Window {
    static std::set<RpsGui_Window*> _set_of_gui_windows_;
    RpsGui_MenuBar* guiwin_menubar;
    Rps_Id guiwin_ownoid;
    std::string guiwin_label;
};
```

**Window Management:**
- **Global Registry:** Set of all GUI windows for tracking
- **Object Ownership:** Association with RefPerSys objects
- **Menu Integration:** Menu bar management
- **Thread Safety:** GUI thread assertions

### RpsGui_CommandWindow Class

Command interface window:

```cpp
class RpsGui_CommandWindow : public RpsGui_Window {
    RpsGui_Pack* cmdwin_pack;
};
```

**Command Interface:**
- **Menu Bar:** Application menus (Dump, Exit, Quit)
- **Pack Container:** Widget layout management
- **Delayed Initialization:** Asynchronous widget setup
- **Event Handling:** Menu callbacks and user interactions

### RpsGui_OutputWindow Class

Output display window:

```cpp
class RpsGui_OutputWindow : public RpsGui_Window {
    Fl_Text_Buffer* outwin_buffer;
    Fl_Text_Editor* outwin_uppereditor;
    Fl_Text_Editor* outwin_lowereditor;
};
```

**Output Display:**
- **Text Buffer:** Content management
- **Dual Editors:** Upper and lower text editing areas
- **Display Formatting:** Structured output presentation

## Key Functions

### Widget Type Display
```cpp
void RpsGui_ShowWidget::output(std::ostream* pout) const
```

**Type Mapping:**
- **RefPerSys Types:** `rps-command-window`, `rps-output-window`, `rps-menubar`
- **FLTK Types:** `normal-button`, `hold-browser`, `float-input`, etc.
- **Address Display:** Widget memory addresses
- **Label Information:** Widget labels and identifiers

### Window Lifecycle
```cpp
RpsGui_Window::RpsGui_Window(int w, int h, const std::string& lab)
RpsGui_Window::~RpsGui_Window()
```

**Window Operations:**
- **Registration:** Add to global window set
- **Object Association:** Link with RefPerSys objects
- **Cleanup:** Remove from registry and clear ownership
- **Event Loop Control:** Stop event loop when no windows remain

### Menu System
```cpp
void RpsGui_CommandWindow::initialize_menubar(void)
```

**Menu Structure:**
- **App Menu:** Dump, Exit, Quit operations
- **Keyboard Shortcuts:** `FL_F+1`, `^x`, `^q`
- **Visual Styling:** Colors and box types
- **Callback Integration:** Menu action handlers

### Menu Callbacks
```cpp
static void menu_dump_cb(Fl_Widget* widg, void* ptr)
static void menu_exit_cb(Fl_Widget* widg, void* ptr)
static void menu_quit_cb(Fl_Widget* widg, void* ptr)
```

**Menu Actions:**
- **Dump:** Save system state to directory
- **Exit:** Save and terminate application
- **Quit:** User choice dialog for exit behavior

## Widget Initialization

### Delayed Initialization Pattern
```cpp
RPS_FLTK_ADD_DELAYED_LABELED_TODO_0(curfltkframe,
    "Todo:RpsGui_CommandWindow-initmenubar-TODO",
    0.25,
    [=](Rps_CallFrame* cf, void*, void*) {
        this->initialize_menubar();
    });
```

**Asynchronous Setup:**
- **Delayed Execution:** Postpone widget creation
- **Call Frame Integration:** RefPerSys call frame context
- **Lambda Callbacks:** C++11 lambda for initialization
- **Timing Control:** Configurable delays

### Layout Management
```cpp
void RpsGui_CommandWindow::initialize_pack(void)
```

**Widget Layout:**
- **Pack Container:** Vertical widget stacking
- **Size Calculation:** Dynamic sizing based on window dimensions
- **Border Management:** Proper spacing and margins
- **Color Schemes:** Visual differentiation

## Event Handling

### Window Events
```cpp
int RpsGui_Window::handle(int evtype)
```

**Event Processing:**
- **Escape Key:** Override default FLTK behavior
- **Window Hide:** Widget deletion on window close
- **Event Logging:** Debug information for all events
- **Base Handling:** Delegate to FLTK parent class

### Thread Safety
```cpp
RPS_ASSERT(rps_is_main_gui_thread());
```

**GUI Thread Enforcement:**
- **Thread Assertions:** Ensure GUI operations on main thread
- **Synchronization:** Proper locking for shared resources
- **Event Loop Safety:** FLTK event loop thread requirements

## Object Integration

### Payload System
```cpp
class Rps_PayloadWindow : public Rps_Payload {
public:
    virtual ~Rps_PayloadWindow();
    virtual bool is_erasable() const;
    virtual void gc_mark(Rps_GarbageCollector& gc) const;
    virtual void dump_scan(Rps_Dumper* du) const;
    virtual void dump_json_content(Rps_Dumper* du, Json::Value&) const;
};
```

**RefPerSys Integration:**
- **Garbage Collection:** Mark GUI objects during GC
- **Persistence:** Dump and load window state
- **Erasability:** Allow payload removal
- **Ownership:** Associate windows with RefPerSys objects

### Object Ownership
```cpp
void RpsGui_Window::clear_owning_object(void)
```

**Ownership Management:**
- **OID Tracking:** Object identifier association
- **Payload Cleanup:** Remove window payloads from objects
- **Mutex Protection:** Thread-safe object access
- **Reference Management:** Proper cleanup on window destruction

## Configuration and Styling

### Color Schemes
```cpp
this->color(fl_rgb_color(240,248,255)); // Azure for command windows
cmdwin_pack->color(fl_rgb_color(255,250,240)); // FloralWhite for packs
guiwin_menubar->color(fl_rgb_color(255,228,225)); // MistyRose for menus
```

**Visual Design:**
- **Consistent Colors:** FLTK RGB color specifications
- **Box Types:** `FL_FLAT_BOX`, `FL_BORDER_BOX`, `FL_ROUNDED_BOX`
- **Layout Spacing:** Configurable borders and gaps

### Constants and Dimensions
```cpp
const int guiwin_border = 4;
const int menu_height = 24;
const int right_menu_gap = 8;
```

**Layout Parameters:**
- **Border Width:** Window border dimensions
- **Menu Height:** Menu bar vertical space
- **Gap Management:** Spacing between elements

## Debugging Support

### Debug Logging
```cpp
RPS_DEBUG_LOG(GUI, "RpsGui_CommandWindow new this=" << RpsGui_ShowFullWidget(this));
```

**Debug Information:**
- **Widget Details:** Full widget information display
- **Call Traces:** Backtrace logging for debugging
- **State Tracking:** Construction and destruction logging
- **Event Monitoring:** GUI event logging

### Widget Display Functions
```cpp
RpsGui_ShowWidget show_widget(widget_ptr);
show_widget.output(&std::cout);
```

**Debug Output:**
- **Type Information:** Widget type identification
- **Memory Addresses:** Pointer values for debugging
- **Label Display:** Widget labels and identifiers
- **Hierarchy:** Parent-child relationships

## File Status

**Status:** Complete FLTK GUI implementation with minor incomplete features
**Date:** 2019-2020
**Purpose:** FLTK-based graphical user interface for RefPerSys

## Limitations and Incomplete Features

### Current Limitations
1. **Output Window:** `RpsGui_OutputWindow` initialization unimplemented
2. **Text Editors:** Upper and lower editor setup missing
3. **Buffer Management:** Text buffer initialization incomplete
4. **Menu System:** Output window menu bar not implemented

### Known Issues
1. **Delayed Initialization:** Complex asynchronous setup pattern
2. **Thread Constraints:** Strict GUI thread requirements
3. **FLTK Version Dependencies:** Specific FLTK version requirements
4. **Integration Complexity:** Coordination between FLTK and RefPerSys

## Summary

The `fltkwindow_rps.cc` file provides a comprehensive FLTK-based GUI implementation for RefPerSys, featuring window management, menu systems, widget type identification, and RefPerSys object integration. The implementation demonstrates sophisticated GUI programming with proper thread safety, event handling, and debugging support. While mostly complete, it includes some placeholder implementations for output windows. The code showcases the complexity of integrating a C++ GUI toolkit with a reflective persistent system, providing both user interface capabilities and deep system integration. As an alternative to the Qt-based GUI, this FLTK implementation offers a lighter-weight GUI solution with extensive debugging and logging capabilities built into the widget system.