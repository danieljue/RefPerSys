# Plugin Build Script (`attic/build-plugin.sh`)

**File Path:** `attic/build-plugin.sh`

## Overview

This shell script provides the build system for compiling RefPerSys plugins from C++ source files into dynamically loadable shared objects. It handles dependency scanning, compiler flag extraction, and proper linking for plugin integration with the main RefPerSys executable.

## Script Structure

### Header and Documentation
```bash
#!/bin/dash
# file build-plugin.sh in RefPerSys - see http://refpersys.org/

### invocation as
##     ./build-plugin.sh <C++-plugin-source> <plugin-sharedobject>

### example
##  To compile the C++ plugin source in file  plugins_dir/rpsplug_createclass.cc
##  into the dlopen-able shared object /tmp/rpsplug_createclass.so
##  run the following command
##     ./build-plugin.sh plugins_dir/rpsplug_createclass.cc \
##                       /tmp/rpsplug_createclass.so
```

**Key Features:**
- Dash shell implementation for portability
- Comprehensive inline documentation
- Example usage patterns
- Integration with RefPerSys plugin system

## Command-Line Interface

### Usage Syntax
```bash
./build-plugin.sh <C++-plugin-source> <output-shared-object>
```

### Help System
```bash
if [ "$1" = '--help' ]; then
    echo $0 usage to compile a RefPerSys plugin C++ code on Linux:
    echo $0 "<plugin-C++-source-file>" "<output-shared-object>"
    echo for example: $0 plugins_dir/rpsplug_createclass.cc /tmp/rpsplug_createclass.so
    echo a later execution is: ./refpersys --plugin-after-load=/tmp/rpsplug_createclass.so
    exit
fi
```

**Help Features:**
- Usage instructions
- Example compilation
- Runtime loading instructions
- Source file preview

## Environment Setup

### Required Environment Variables
```bash
if [ -z "$REFPERSYS_TOPDIR" ]; then
    printf "%s: missing REFPERSYS_TOPDIR\n" $0 > /dev/stderr
    exit 1
fi
```

**Environment Dependencies:**
- `REFPERSYS_TOPDIR`: Root directory of RefPerSys project
- Build system integration through make

### Build Configuration Loading
```bash
eval $(/usr/bin/make print-plugin-settings)

### plugincppflags contain compiler flags
### pluginlinkerflags contain linker flags
```

**Configuration Sources:**
- Makefile-based build settings
- Compiler and linker flag extraction
- Plugin-specific configuration

## Dependency Analysis

### Source Code Scanning

#### Compiler Flags Extraction
```bash
if /usr/bin/fgrep -q '@RPSCOMPILEFLAGS=' $cppfile ; then
    plugincppflags=$(/bin/head -$MY_HEAD_LINES_THRESHOLD $cppfile | 
                     /usr/bin/gawk --source '/@RPSCOMPILEFLAGS=/ { 
                         for (i=2; i<=NF; i++) print $i; }')
else
    plugincppflags=()
fi
```

**Directive Format:**
```cpp
// In plugin source file:
// @RPSCOMPILEFLAGS= -O2 -march=native -I/some/include
```

#### Library Dependencies
```bash
if /usr/bin/fgrep -q '@RPSLIBES=' $cppfile ; then
    pluginlinkerflags=$(/bin/head -$MY_HEAD_LINES_THRESHOLD $cppfile | 
                        /usr/bin/gawk --source '/@RPSLIBES=/ { 
                            for (i=2; i<=NF; i++) print $i; }')
fi
```

**Library Specification:**
```cpp
// @RPSLIBES= -lmylib -L/my/lib/path
```

#### Package Configuration Integration
```bash
if  /usr/bin/fgrep -q '//@@PKGCONFIG' $cppfile ; then
    pkglist=$($REFPERSYS_TOPDIR/do-scan-pkgconfig --raw $cppfile)
    plugincppflags="$plugincppflags $(pkg-config --cflags $pkglist)"
    pluginlinkerflags="$pluginlinkerflags $(pkg-config --libs $pkglist)"
fi
```

**Pkg-config Integration:**
```cpp
//@@PKGCONFIG jsoncpp
//@@PKGCONFIG gtk+-3.0
```

### Special Handling

#### JSON-CPP Path Hack
```bash
## ugly hack needed in March 2024 after commit 456fcb27bc57f
if [ -f /usr/include/jsoncpp/json/json.h ]; then
    plugincppflags+=" -I/usr/include/jsoncpp"
fi
```

**Compatibility Workaround:**
- Handles non-standard JSON-CPP installations
- Adds include path when detected

## Compilation Process

### Build Validation
```bash
## check that we have the necessary shell variables set
if [ -z "$RPSPLUGIN_CXX" ]; then
    echo RPSPLUGIN_CXX missing in $0 > /dev/stderr
    exit 1
fi
if [ -z "$RPSPLUGIN_CXXFLAGS" ]; then
    echo RPSPLUGIN_CXXFLAGS missing in $0 > /dev/stderr
    exit 1
fi
if [ -z "$RPSPLUGIN_LDFLAGS" ]; then
    echo RPSPLUGIN_LDFLAGS missing in $0 > /dev/stderr
    exit 1
fi
```

**Required Variables:**
- `RPSPLUGIN_CXX`: C++ compiler command
- `RPSPLUGIN_CXXFLAGS`: Compiler flags
- `RPSPLUGIN_LDFLAGS`: Linker flags

### Compilation Execution
```bash
$RPSPLUGIN_CXX $RPSPLUGIN_CXXFLAGS $plugincppflags -Wall -Wextra -I. -fPIC -shared $cppfile $RPSPLUGIN_LDFLAGS $pluginlinkerflags -o $pluginfile
```

**Compilation Flags:**
- `-Wall -Wextra`: Maximum warning levels
- `-fPIC`: Position-independent code for shared libraries
- `-shared`: Create shared object
- `-I.`: Include current directory

### Error Handling
```bash
|| ( printf "\n%s failed to compile RefPerSys plugin %s to %s\n \
            (°cxxflags %s\n °cppflags %s\n °ldflags %s\n °linkerflags %s\n °pkg-list %s\n)\n" \
         $0 $cppfile $pluginfile "$RPSPLUGIN_CXXFLAGS" "$plugincppflags" \
         "$RPSPLUGIN_LDFLAGS" "$pluginlinkerflags" "$pkglist" > /dev/stderr ; \
     /usr/bin/logger --id=$$ -s  -t $0 -puser.warning \
                     "$0 failed to compile RefPerSys plugin $cppfile to $pluginfile in $(/bin/pwd)\n" ; \
     exit 1 )
```

**Error Reporting:**
- Detailed failure information
- Compiler and linker flags display
- System logging integration
- Process ID tracking

## Logging and Monitoring

### System Logging
```bash
/usr/bin/logger -i -s --id=$$ --tag refpersys-build-plugin  --priority user.info "$0" "$@"

/usr/bin/logger --id=$$ -s  -t "$0:" "starting" cppfile= $1 pluginfile= $2 curdate= $curdate REFPERSYS_TOPDIR= $REFPERSYS_TOPDIR cwd $(/bin/pwd)
```

**Logging Features:**
- Process ID tracking
- Timestamp recording
- Build parameter logging
- Success/failure reporting

### Progress Output
```bash
printf "start %s at %s: C++ file %s, plugin file %s in %s\n" $0 \
       "$curdate" $cppfile $pluginfile "$(/bin/pwd)" > /dev/stderr
```

**User Feedback:**
- Build start confirmation
- File path information
- Working directory display

## Plugin Integration

### Build System Integration
The script integrates with RefPerSys's make-based build system:

```makefile
# Makefile integration
print-plugin-settings:
	@echo "RPSPLUGIN_CXX=g++"
	@echo "RPSPLUGIN_CXXFLAGS=-std=c++17 -O2"
	@echo "RPSPLUGIN_LDFLAGS=-ldl -pthread"
```

### Runtime Loading
Compiled plugins are loaded using:
```bash
./refpersys --plugin-after-load=/tmp/plugin.so
```

## Limitations and Known Issues

### TODO Items
```bash
## TODO: in commit 3dc7163fa129 of May 15, 2024 this is a buggy
## script. I (Basile S.) hope for some Python3 expert to replace it
## with a more robust Python script which would scan the C++ source
## code comments....
```

**Acknowledged Issues:**
- Script described as "buggy"
- Planned replacement with Python implementation
- Dependency scanning limitations

### Technical Limitations
- **Shell Script Complexity:** Large script in dash shell
- **Error Handling:** Basic error detection and reporting
- **Dependency Resolution:** Limited pkg-config integration
- **Cross-Platform:** Linux-specific paths and commands

## File Status

**Status:** Deprecated/Buggy
**Date:** 2020-2024
**Reason for attic:** Script acknowledged as buggy, planned replacement with Python

## Usage Example

### Complete Plugin Build Workflow
```bash
# 1. Create plugin source with directives
cat > myplugin.cc << 'EOF'
//@@PKGCONFIG jsoncpp
//@RPSCOMPILEFLAGS= -O3 -march=native
//@RPSLIBES= -lmylib

#include "refpersys.hh"
// Plugin implementation...
EOF

# 2. Build the plugin
./build-plugin.sh myplugin.cc /tmp/myplugin.so

# 3. Load and test
./refpersys --plugin-after-load=/tmp/myplugin.so
```

## Future Development

### Planned Improvements
1. **Python Rewrite:** More robust dependency scanning
2. **Better Error Handling:** Comprehensive build failure analysis
3. **Cross-Platform Support:** Windows and macOS compatibility
4. **Incremental Builds:** Dependency tracking and partial rebuilds
5. **IDE Integration:** Build system integration with development tools

This plugin build script represents the foundation of RefPerSys's dynamic plugin system, enabling runtime extension of the system through compiled C++ modules. Despite its acknowledged limitations, it successfully demonstrates the integration of build-time dependency analysis with runtime plugin loading.