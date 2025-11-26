# Mock Object Space Data (`attic/mock.json`)

**File Path:** `attic/mock.json`

## Overview

This file contains a complete JSON serialization of a RefPerSys object space with 102 objects, representing a minimal but functional RefPerSys system state. It serves as test data, development fixtures, or a reference implementation of the RefPerSys persistence format.

## File Structure

### Object Space Header
```json
{
  "object_space": {
    "meta_comment": "prologue of RefPerSys space file:",
    "format": "RefPerSysFormat2019A",
    "nbobjects": 102,
    "spaceid": "_8J6vNYtP5E800eCr5q"
  }
}
```

**Format Information:**
- **Format Version:** `RefPerSysFormat2019A` - Early persistence format
- **Object Count:** 102 objects in this space
- **Space ID:** `_8J6vNYtP5E800eCr5q` - Unique identifier for this object space

### Objects Array
```json
{
  "objects": [
    // Array of 102 object definitions
  ]
}
```

## Core System Objects

### Fundamental Classes

#### Object Class (`_5yhJGgxLwLp00X0xEQ`)
```json
{
  "meta_comment": "∈object",
  "attrs": [{"at": "_1EBVGSfW2m200z18rx", "va": "object"}],
  "class": "_5yhJGgxLwLp00X0xEQ",
  "class_methodict": [
    {"methclos": {"env": [], "fn": "_02iWbXmFx8f04ldLRt", "vtype": "closure"}, "methosel": "_02iWbXmFx8f04ldLRt"},
    {"methclos": {"env": [], "fn": "_18DO93843oX02UWzq6", "vtype": "closure"}, "methosel": "_1Win5yzaf1L02cBUlV"},
    {"methclos": {"env": [], "fn": "_4x9jd2yAe8A02SqKAx", "vtype": "closure"}, "methosel": "_4ojpzRzyRWz02DNWMe"}
  ],
  "class_name": "object",
  "class_super": "_6XLY6QfcDre02922jz",
  "class_symb": "_7X9eGs8601M021nMue",
  "mtime": 1583088448.8099999,
  "oid": "_5yhJGgxLwLp00X0xEQ",
  "payload": "classinfo"
}
```

**Key Features:**
- Root of the inheritance hierarchy
- Multiple method implementations for different selectors
- Display methods for web interface

#### Value Class (`_6XLY6QfcDre02922jz`)
```json
{
  "meta_comment": "∈class",
  "class": "_41OFI3r0S1t03qdB2E",
  "class_methodict": [],
  "class_name": "value",
  "class_super": "_6XLY6QfcDre02922jz",
  "class_symb": "_7ZRsf01mtit01xPUei",
  "mtime": 1578114964,
  "oid": "_6XLY6QfcDre02922jz",
  "payload": "classinfo"
}
```

### Primitive Type Classes

#### Integer Class (`_2A2mrPpR3Qf03p6o5b`)
```json
{
  "meta_comment": "∈class",
  "attrs": [{"at": "_1EBVGSfW2m200z18rx", "va": "int"}],
  "class": "_41OFI3r0S1t03qdB2E",
  "class_methodict": [
    {"methclos": {"env": [], "fn": "_8KJHUldX8GJ03G5OWp", "vtype": "closure"}, "methosel": "_1Win5yzaf1L02cBUlV"}
  ],
  "class_name": "int",
  "class_super": "_5yhJGgxLwLp00X0xEQ",
  "class_symb": "_2SqiFRt6QTI01CPGf1",
  "mtime": 1583088448.8099999,
  "oid": "_2A2mrPpR3Qf03p6o5b",
  "payload": "classinfo"
}
```

#### String Class (`_62LTwxwKpQ802SsmjE`)
```json
{
  "meta_comment": "∈class",
  "attrs": [{"at": "_1EBVGSfW2m200z18rx", "va": "string"}],
  "class": "_41OFI3r0S1t03qdB2E",
  "class_methodict": [
    {"methclos": {"env": [], "fn": "_2KnFhlj8xW800kpgPt", "vtype": "closure"}, "methosel": "_1Win5yzaf1L02cBUlV"}
  ],
  "class_name": "string",
  "class_super": "_5yhJGgxLwLp00X0xEQ",
  "class_symb": "_3ppZRdS4dt002ZqydW",
  "mtime": 1583088448.8099999,
  "oid": "_62LTwxwKpQ802SsmjE",
  "payload": "classinfo"
}
```

#### Double Class (`_98sc8kSOXV003i86w5`)
```json
{
  "meta_comment": "∈class",
  "attrs": [{"at": "_1EBVGSfW2m200z18rx", "va": "double"}],
  "class": "_41OFI3r0S1t03qdB2E",
  "class_methodict": [
    {"methclos": {"env": [], "fn": "_7oa7eIzzcxv03TmmZH", "vtype": "closure"}, "methosel": "_1Win5yzaf1L02cBUlV"}
  ],
  "class_name": "double",
  "class_super": "_5yhJGgxLwLp00X0xEQ",
  "class_symb": "_1WoBcvgat8T0044rDo",
  "mtime": 1583088448.8099999,
  "oid": "_98sc8kSOXV003i86w5",
  "payload": "classinfo"
}
```

### Collection Classes

#### Set Classes
- **Set:** `_6JYterg6iAu00cV9Ye` - Basic immutable set
- **Mutable Set:** `_0J1C39JoZiv03qA2HA` - Modifiable set

#### Tuple Class (`_6NVM7sMcITg01ug5TC`)
```json
{
  "meta_comment": "∈class",
  "attrs": [{"at": "_1EBVGSfW2m200z18rx", "va": "tuple"}],
  "class": "_41OFI3r0S1t03qdB2E",
  "class_methodict": [
    {"methclos": {"env": [], "fn": "_7oa7eIzzcxv03TmmZH", "vtype": "closure"}, "methosel": "_1Win5yzaf1L02cBUlV"}
  ],
  "class_name": "tuple",
  "class_super": "_5yhJGgxLwLp00X0xEQ",
  "class_symb": "_5KVs9kaMYiU040KwVj",
  "mtime": 1583088448.8099999,
  "oid": "_6NVM7sMcITg01ug5TC",
  "payload": "classinfo"
}
```

### Function Classes

#### Core Function Class (`_9Gz1oNPCnkB00I6VRS`)
```json
{
  "meta_comment": "∈class",
  "attrs": [{"at": "_1EBVGSfW2m200z18rx", "va": "core_function"}],
  "class": "_41OFI3r0S1t03qdB2E",
  "class_methodict": [],
  "class_name": "core_function",
  "class_super": "_9BnrMLXUhfG00llx8X",
  "class_symb": "_37BVwKYeEW800KchHe",
  "mtime": 1583088448.8099999,
  "oid": "_9Gz1oNPCnkB00I6VRS",
  "payload": "classinfo"
}
```

### GUI-Related Classes

#### Qt Integration Classes
- **QObject:** `_7b8jzCpOStA04e0jxs` - Qt object base class
- **RPS Window:** `_1DUx3zfUzIb04lqNVt` - GUI window class
- **Command Text Edit:** `_54CP9eaTmxT00lzbEW` - Command input widget
- **Output Text Edit:** `_1NWEOIzo3WU03mE42Q` - Output display widget

### Web Interface Classes

#### Web Handler (`_1R1LKwGhoTr02e90bn`)
```json
{
  "meta_comment": "∈class",
  "attrs": [{"at": "_1EBVGSfW2m200z18rx", "va": "web_handler"}],
  "class": "_41OFI3r0S1t03qdB2E",
  "class_methodict": [],
  "class_name": "web_handler",
  "class_super": "_41OFI3r0S1t03qdB2E",
  "class_symb": "_9kaQRzkKNHh02gFgyw",
  "mtime": 1601105597.96,
  "oid": "_1R1LKwGhoTr02e90bn",
  "payload": "classinfo"
}
```

#### JSON Class (`_3GHJQW0IIqS01QY8qD`)
```json
{
  "meta_comment": "∈class",
  "attrs": [{"at": "_1EBVGSfW2m200z18rx", "va": "json"}],
  "class": "_41OFI3r0S1t03qdB2E",
  "class_methodict": [
    {"methclos": {"env": [], "fn": "_42cCN1FRQSS03bzbTz", "vtype": "closure"}, "methosel": "_1Win5yzaf1L02cBUlV"}
  ],
  "class_name": "json",
  "class_super": "_5yhJGgxLwLp00X0xEQ",
  "class_symb": "_7vUmzaAjb2f000BAFb",
  "mtime": 1583088448.8099999,
  "oid": "_3GHJQW0IIqS01QY8qD",
  "payload": "classinfo"
}
```

## Core Functions

### Display Functions
The mock data includes numerous core functions for displaying different value types:

- **Integer Display:** `_8KJHUldX8GJ03G5OWp`
- **String Display:** `_2KnFhlj8xW800kpgPt`
- **Double Display:** `_7oa7eIzzcxv03TmmZH`
- **JSON Display:** `_42cCN1FRQSS03bzbTz`
- **Object Display:** `_18DO93843oX02UWzq6`
- **Class Display:** `_8lKdW7lgcHV00WUOiT`

### Object Content Display
```json
{
  "meta_comment": "∈core_function",
  "applying": true,
  "attrs": [{"at": "_0jdbikGJFq100dgX1n", "va": "method object!display_object_content_web"}],
  "class": "_9Gz1oNPCnkB00I6VRS",
  "mtime": 1583090609.79,
  "oid": "_5nSiRIxoYQp00MSnYA"
}
```

## Symbol Objects

### Named Attributes
- **Name:** `_1EBVGSfW2m200z18rx` - The fundamental "name" attribute
- **Named Attribute:** `_4pSwobFHGf301Qgwzh` - Class for named attributes

### Named Selectors
- **Display Value Web:** `_1Win5yzaf1L02cBUlV` - Web display selector
- **Display Object Content Web:** `_02iWbXmFx8f04ldLRt` - Content display selector
- **Display Object Payload Web:** `_14M7WuJSWw702zB0M9` - Payload display selector

## Contributors Data

### Project Contributors
```json
{
  "meta_comment": "∈contributor_to_RefPerSys",
  "attrs": [
    {"at": "_0D6zqQNe4eC02bjfGs", "va": "basile@starynkevitch.net"},
    {"at": "_0LbCts6NacB03SMXz4", "va": "http://starynkevitch.net/Basile/"},
    {"at": "_0XMNvzdABUM03Bj7WP", "va": "caa608885ae3c9c8ec8000"},
    {"at": "_3N8vZ2Cw62z024XxCg", "va": "Basile"},
    {"at": "_6QAanFi9yLx00spBST", "va": "Starynkevitch"}
  ],
  "class": "_5CYWxcChKN002rw1fI",
  "mtime": 1581059623,
  "oid": "_5EuQjoAiaR300kp2q0"
}
```

## Agenda and Concurrency

### Agenda System
```json
{
  "meta_comment": "∈agenda",
  "attrs": [{"at": "_1EBVGSfW2m200z18rx", "va": "the_agenda"}],
  "class": "_3s7ztCCoJsj04puTdQ",
  "mtime": 1599618139.8299999,
  "oid": "_1aGtWm38Vw701jDhZn",
  "payload": "agenda"
}
```

### Tasklet Support
```json
{
  "meta_comment": "∈class",
  "attrs": [{"at": "_1EBVGSfW2m200z18rx", "va": "tasklet"}],
  "class": "_41OFI3r0S1t03qdB2E",
  "class_methodict": [],
  "class_name": "tasklet",
  "class_super": "_41OFI3r0S1t03qdB2E",
  "class_symb": "_9V4nwGbGLFP00cz0VL",
  "mtime": 1599661213,
  "oid": "_8fYqEw8vTED03wsznt",
  "payload": "classinfo"
}
```

## Space Object

### Root Space
```json
{
  "meta_comment": "∈space",
  "class": "_2i66FFjmS7n03HNNBx",
  "mtime": 1575698824,
  "oid": "_8J6vNYtP5E800eCr5q",
  "payload": "space"
}
```

## Usage and Purpose

### Development Testing
This mock data serves as:
- **Unit Test Fixtures:** Known object state for testing
- **Development Data:** Baseline system for experimentation
- **Documentation Examples:** Concrete instances of RefPerSys concepts

### Persistence Format Reference
The file demonstrates:
- **JSON Structure:** Complete object space serialization
- **Object Relationships:** Class hierarchies and references
- **Metadata:** Timestamps, comments, and versioning
- **Payload Types:** Different object payload formats

### System Bootstrap
Can be used to:
- **Initialize Development Systems:** Quick setup with known state
- **Test Persistence:** Verify load/save functionality
- **Demonstrate Features:** Show complete working system

## File Status

**Status:** Active/Reference
**Date:** 2019-2020
**Purpose:** Test data and persistence format reference

## Technical Notes

### Object IDs
- **Format:** Prefixed with underscore, alphanumeric
- **Uniqueness:** Globally unique within space
- **References:** Used for all object relationships

### Timestamps
- **Format:** Unix timestamps with fractional seconds
- **Purpose:** Track object modification times
- **Precision:** Millisecond-level granularity

### Method Dictionaries
- **Structure:** Arrays of method closures and selectors
- **Inheritance:** Methods inherited from superclasses
- **Dynamic Dispatch:** Runtime method resolution

This mock object space provides a comprehensive snapshot of a functional RefPerSys system, containing all core classes, functions, and data structures needed for basic operation. It serves as both a test fixture and a reference implementation of the RefPerSys persistence format.