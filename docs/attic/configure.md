# Build Configuration Script (`attic/configure`)

**File Path:** `attic/configure`

## Overview

This shell script serves as a build configuration utility for RefPerSys, handling dependency installation, environment setup, and basic configuration management. It was designed to automate the setup process for building RefPerSys on Debian-based systems, ensuring all required development packages and tools are properly installed.

## Script Purpose

### Build Environment Setup
The script enables automated configuration by:

- **Dependency Management:** Installing required Debian packages
- **Environment Configuration:** Setting up build variables and paths
- **Version Information:** Displaying project version and licensing details
- **Help System:** Providing usage information and documentation
- **Error Handling:** Robust error checking and user feedback

## Command-Line Interface

### Usage Options
```bash
./configure [-h | -l | -v]
```

### Options
- **`-h`:** Show usage help
- **`-l`:** Display license information
- **`-v`:** Show version information

### Option Constraints
- Options are mutually exclusive
- Maximum one option allowed
- Invalid combinations result in error

## Script Functions

### Information Display
```bash
info() {
  echo "[INFO] $1..." 1>&2
}
```

**Info Function:**
- Displays informational messages to stderr
- Prefixed with "[INFO]" for clarity
- Used for progress indication

### Error Handling
```bash
fail() {
  echo "[FAIL] $1..." 1>&2
  exit 1
}
```

**Fail Function:**
- Displays error messages to stderr
- Prefixed with "[FAIL]" for visibility
- Terminates script execution with error code

### Package Installation
```bash
install() {
  if dpkg-query -W --showformat='${Status}\n' "$1" \
  | grep -q 'not-installed'; then
    info "Installing $1"
    sudo apt install -y "$1"
  fi
}
```

**Install Function:**
- Checks if package is already installed
- Installs package only if missing
- Uses non-interactive apt installation
- Provides progress feedback

## Configuration Actions

### Local Makefile Creation
```bash
info 'Creating local makefile configuration file ~/.refpersys.mk'
if [ -f $HOME/.refpersys.mk ]; then
    /bin/mv -v $HOME/.refpersys.mk $HOME/.refpersys.mk%
fi
{
  echo 'CC= gcc-12';
  echo 'CXX= g++-12';
  echo 'RPS_BUILD_CC= gcc-12';
  echo 'RPS_BUILD_CXX= g++-12';
} > "$HOME/.refpersys.mk"
```

**Configuration File:**
- Creates `~/.refpersys.mk` for local build settings
- Backs up existing file with `%` suffix
- Sets GCC 12 as the default compiler
- Defines both host and build compiler variables

### System Updates
```bash
info 'Updating APT database'
sudo apt update
```

**Package Database:**
- Updates APT package index
- Ensures package information is current
- Required before package installations

### Dependency Installation
```bash
sudo apt install time
sudo apt install libunistring-dev
sudo apt install libreadline-dev
install libjsoncpp-dev
install libcurl4-openssl-dev
install zlib1g-dev
```

**Required Packages:**
- **`time`:** Command timing utility
- **`libunistring-dev`:** Unicode string library development files
- **`libreadline-dev`:** GNU readline library development files
- **`libjsoncpp-dev`:** JSON parsing library
- **`libcurl4-openssl-dev`:** cURL library with OpenSSL
- **`zlib1g-dev`:** Compression library development files

## Version Information

### Git Integration
```bash
if [ $VERSION -eq 0 ]; then
  echo "Commit: $(git rev-parse HEAD)"
  echo "Date: $(git log -1 --format=%cd)"
  exit 0
fi
```

**Version Display:**
- Shows current Git commit hash
- Displays last commit date
- Provides build identification information

## License Information

### GPLv3+ Display
```bash
if [ $LICENSE -eq 0 ]; then
  echo ''
  echo 'RefPerSys is free software: you can redistribute it and/or modify'
  echo 'it under the terms of the GNU General Public License as published'
  echo 'by the Free Software Foundation, either version 3 of the License,'
  echo 'or (at your option) any later version.'
  # ... full license text
fi
```

**License Details:**
- Displays complete GPLv3+ license text
- Includes warranty disclaimer
- Provides license reference information

## Error Handling

### Command-Line Validation
```bash
[ $ARGC -gt 2 ] && fail "$EMSG"

if [ $ARGC -gt 1 ]; then
  [ $HELP -ne 0 ] && [ $LICENSE -ne 0 ] && [ $VERSION -ne 0 ] && fail "$EMSG"
fi
```

**Validation Rules:**
- Maximum 2 arguments allowed
- Multiple options not permitted
- Clear error messages for invalid usage

## System Requirements

### Target Environment
- **Debian-based Linux distributions**
- **APT package manager**
- **sudo privileges for package installation**
- **Git repository access for version information**

### Compiler Requirements
- **GCC 12** (specifically configured)
- **G++ 12** for C++ compilation
- **Development libraries** for all dependencies

## Usage Examples

### Basic Setup
```bash
# Run configuration with default settings
./configure

# Show help information
./configure -h

# Display license
./configure -l

# Show version information
./configure -v
```

### Integration with Build System
```bash
# Configure and then build
./configure
make

# Clean configuration
rm ~/.refpersys.mk
```

## File Status

**Status:** Legacy Configuration Script
**Date:** 2022
**Purpose:** Automated build environment setup

## Limitations and Issues

### Current Limitations
1. **Debian-Specific:** Only works on Debian-based systems
2. **Hardcoded Versions:** GCC 12 specifically required
3. **Limited Options:** Few configuration choices available
4. **No Cross-Compilation:** Only native builds supported

### Potential Issues
1. **Package Availability:** May fail if packages are unavailable
2. **Permission Issues:** sudo access required for installations
3. **Network Dependencies:** Requires internet for package downloads
4. **Version Conflicts:** May conflict with existing compiler installations

## Evolution and Replacement

### Why Archived
This script represents an early approach to build configuration that has likely been replaced by more sophisticated build systems like:

- **CMake or Meson** for cross-platform builds
- **Container-based builds** (Docker, etc.)
- **Package managers** (Conan, vcpkg, etc.)
- **CI/CD systems** with automated dependency management

### Lessons Learned
- **Cross-platform compatibility** is essential
- **Flexible configuration** options are valuable
- **Error handling** should be comprehensive
- **Documentation** integration is important

## Summary

The `configure` script represents an early attempt at automating the RefPerSys build environment setup. While functional for its intended Debian-based environment, it demonstrates the challenges of build system configuration in complex C++ projects. The script successfully handles dependency installation, compiler configuration, and basic error checking, but its Debian-specific nature and hardcoded compiler versions limit its applicability. As a historical artifact in the attic directory, it provides insight into the evolution of build systems and the importance of flexible, cross-platform configuration tools in modern software development. The script's approach to user feedback, error handling, and modular function design remains a good example of shell scripting practices, even as build system technologies have advanced significantly.