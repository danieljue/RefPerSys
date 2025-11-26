# Python Timestamp Generation Script (`attic/do-generate-python-timestamp.sh`)

**File Path:** `attic/do-generate-python-timestamp.sh`

## Overview

This shell script generates Python timestamp constants containing build and version information for RefPerSys. It creates Python variable declarations with current timestamp, Git information, and project metadata, similar to the C version (`rps-generate-timestamp.sh`) but formatted for Python consumption.

## Script Purpose

### Python Build Metadata Generation
The script enables Python integration by:

- **Timestamp Generation:** Current build time and date information
- **Git Information:** Repository status, commit IDs, and tags
- **Directory Resolution:** Absolute path to project root
- **Checksum Calculation:** MD5 hash of project files
- **Python Constants:** Properly formatted Python variable declarations

## Command-Line Interface

### Usage
```bash
./do-generate-python-timestamp.sh [output_file]
```

### Parameters
- **`$1` (optional):** Output filename for header comment

## Script Implementation

### Locale Configuration
```bash
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
export LC_TIME=en_US.UTF-8
```

**Localization:**
- **UTF-8 Encoding:** Ensures consistent character encoding
- **US Locale:** Standardizes date/time formatting
- **Time Format:** Consistent timestamp representation

### Header Generation
```bash
printf "// generated Python file %s by %s -- DONT EDIT - see refpersys.org\n" $1 $0
```

**File Header:**
- **C-style Comment:** Maintains compatibility with mixed-language projects
- **Generation Info:** Records script and output file names
- **Edit Warning:** Prevents manual modification

## Timestamp Variables

### Current Time Information
```bash
date +"RPSPY_TIMESTAMP=\"%c\"%nRPSPY_TIMELONG=%s;"
```

**Generated Variables:**
- **RPSPY_TIMESTAMP:** Human-readable date/time string
- **RPSPY_TIMELONG:** Unix timestamp (seconds since epoch)

**Example Output:**
```python
RPSPY_TIMESTAMP="Mon Nov 25 15:30:45 2024"
RPSPY_TIMELONG=1732579845;
```

### Directory Information
```bash
printf "RPSPY_TOPDIRECTORY=\"%s\";\n" $(realpath $(pwd))
```

**Project Root:**
- **Absolute Path:** Fully resolved project directory
- **Build Location:** Records where build occurred
- **Reproducibility:** Enables path-dependent operations

## Git Information

### Repository Status Detection
```bash
if git status|grep -q 'nothing to commit' ; then
    endgitid='";'
else
    endgitid='+";'
fi
```

**Status Indication:**
- **Clean Repository:** Standard quote termination
- **Modified Repository:** Plus sign indicates uncommitted changes

### Commit ID Extraction
```bash
(echo -n 'RPSPY_GITID="'; 
 git log --format=oneline -q -1 | cut '-d '  -f1 | tr -d '\n';
 echo $endgitid)
```

**Git Commit Hash:**
- **Latest Commit:** Most recent commit identifier
- **Short Hash:** Abbreviated commit reference
- **Status Marker:** Plus sign for modified working directory

### Tag Information
```bash
(echo -n 'RPSPY_LASTGITTAG="'; (git describe --abbrev=0 --all || echo '*notag*') | tr -d '\n\r\f\"\\\\'; echo '";')
```

**Latest Tag:**
- **Annotated Tags:** Most recent tag description
- **Fallback:** "*notag*" if no tags exist
- **Sanitization:** Removes problematic characters

### Commit Message
```bash
(echo -n 'RPSPY_LASTGITCOMMIT="' ; \
 git log --format=oneline --abbrev=12 --abbrev-commit -q  \
     | head -1 | tr -d '\n\r\f\"\\\\' ; \
 echo '";')
```

**Recent Commit:**
- **Abbreviated Hash:** 12-character commit identifier
- **Subject Line:** First line of commit message
- **Character Filtering:** Removes quotes and backslashes

## Checksum Calculation

### MD5 Hash Generation
```bash
(echo -n 'RPSPY_MD5SUM="' ; cat $(tar tf /tmp/refpersys-$$.tar.gz | grep -v '/$') | md5sum | tr -d '\n -'  ;  echo '";')
```

**File Integrity:**
- **Archive Creation:** Temporary tarball of project files
- **Content Hashing:** MD5 checksum of all source files
- **Build Verification:** Ensures reproducible builds

**Note:** The tar command references `/tmp/refpersys-$$.tar.gz` but the archive creation is not shown in the script. This appears to be incomplete.

## Output Format

### Python Variable Declarations
```python
# Example generated output
RPSPY_TIMESTAMP="Mon Nov 25 15:30:45 2024"
RPSPY_TIMELONG=1732579845;
RPSPY_TOPDIRECTORY="/home/user/projects/refpersys";
RPSPY_GITID="a1b2c3d4e5f6";
RPSPY_LASTGITTAG="v1.0.0";
RPSPY_LASTGITCOMMIT="a1b2c3d4e5f6 Initial commit";
RPSPY_MD5SUM="f5d9b2c8e7a6b4d3c2f1e9a8b7c6d5e4";
```

**Variable Types:**
- **Strings:** Quoted string values
- **Integers:** Numeric timestamp values
- **Paths:** Absolute directory paths

## Integration with Build System

### Usage in Makefiles
```makefile
# Generate Python timestamp constants
python_timestamp.py: do-generate-python-timestamp.sh
    ./do-generate-python-timestamp.sh python_timestamp.py > $@

# Include in Python modules
import timestamp_constants
print(f"Built at: {timestamp_constants.RPSPY_TIMESTAMP}")
```

### Build Process Integration
- **Pre-build Step:** Generate constants before compilation
- **Version Information:** Embed build metadata in Python code
- **Debugging Support:** Include Git information for issue tracking

## Error Handling

### Git Repository Checks
- **Repository Presence:** Assumes Git repository exists
- **Command Availability:** Requires Git to be installed
- **Network Operations:** May fail in offline environments

### File System Operations
- **Path Resolution:** Uses `realpath` for absolute paths
- **Temporary Files:** Creates temporary archives for hashing
- **Permission Issues:** May fail with insufficient access rights

## Performance Considerations

### Execution Time
- **Git Operations:** Repository queries may be slow in large repos
- **File Hashing:** MD5 calculation scales with project size
- **Archive Creation:** Temporary file operations

### Resource Usage
- **Memory:** Minimal memory footprint
- **Disk Space:** Temporary tarball creation
- **External Commands:** Relies on system utilities

## Comparison with C Version

### Similarities
- **Information Scope:** Same metadata categories
- **Git Integration:** Identical repository information
- **Timestamp Format:** Consistent time representation

### Differences
- **Language Target:** Python vs. C preprocessor constants
- **Syntax:** Python variable assignment vs. C `#define`
- **Integration:** Python import vs. C header inclusion

## File Status

**Status:** Complete Implementation
**Date:** Pre-2025 (exact date unknown)
**Purpose:** Python build metadata generation

## Summary

The `do-generate-python-timestamp.sh` script provides essential build metadata generation for Python components of RefPerSys. By extracting timestamp, Git, and project information, it creates Python constants that can be imported by Python modules to access build-time information. The script demonstrates careful handling of Git repository state, character escaping, and cross-platform compatibility. While the MD5 checksum calculation appears incomplete (missing tarball creation), the core functionality for timestamp and version information generation is solid. This script serves as the Python counterpart to the C timestamp generation system, enabling consistent build metadata across different language components of the RefPerSys project.