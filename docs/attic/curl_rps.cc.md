# Web Client Implementation (`attic/curl_rps.cc`)

**File Path:** `attic/curl_rps.cc`

## Overview

This file contains the web client implementation for RefPerSys using libcurl and curlpp (C++ wrapper for libcurl). It provides functionality for HTTP communication, version reporting, and data publishing to web services. The implementation includes initialization, version information, and a framework for publishing RefPerSys data to remote web services.

## File Purpose

### Web Integration and Publishing
The file enables web connectivity by:

- **HTTP Communication:** RESTful API interactions with web services
- **Version Reporting:** Library and version information display
- **Data Publishing:** Sending RefPerSys metadata to web services
- **Git Integration:** User identification from Git configuration
- **JSON Processing:** Web service response parsing and validation

## Implementation Status

### Current State
```cpp
#warning missing C++ code in rps_curl_publish_me
#warning rps_curl_publish_me incomplete, using CURL easy interface
```

**Status:** Framework complete, publishing functionality incomplete

### Design Intent
- **Web Service Integration:** Communication with RefPerSys web services
- **Data Publication:** Sharing system metadata and user information
- **Version Compatibility:** Library version reporting and compatibility
- **Error Handling:** Robust HTTP error management and recovery

## Dependencies

### Required Libraries
```cpp
//@@PKGCONFIG curlpp
#include <curlpp/cURLpp.hpp>
#include <curlpp/Easy.hpp>
#include <curlpp/Options.hpp>
#include <curlpp/Exception.hpp>
```

**Dependencies:**
- **libcurl:** Core HTTP library
- **curlpp:** C++ wrapper for libcurl
- **JsonCpp:** JSON parsing and generation

## Core Functions

### Version Information
```cpp
std::string rps_curl_version(void)
```
**Purpose:** Returns version information for curl libraries

**Implementation:**
- Retrieves libcurl version string
- Formats output with line breaks
- Includes curlpp library version
- Provides Git commit information

**Output Format:**
```
CURL [version info]
    [additional version details]
LibCurlPp [version]
```

### Library Initialization
```cpp
void rps_initialize_curl(void)
```
**Purpose:** Initializes curl library for the application

**Implementation:**
- Calls `curlpp::initialize(CURL_GLOBAL_ALL)`
- Registers cleanup with `atexit(curlpp::terminate)`
- Ensures proper library lifecycle management

### Data Publishing Framework
```cpp
void rps_curl_publish_me(const char* url)
```
**Purpose:** Publishes RefPerSys data to a web service

**Current Implementation:**
- **Git Configuration Parsing:** Extracts user name and email from `~/.gitconfig`
- **Status Request:** Makes HTTP GET request to `/status` endpoint
- **JSON Response Parsing:** Validates and processes web service status
- **Framework for POST:** Structure in place for data submission

## Git Configuration Parsing

### User Information Extraction
```cpp
std::string path_gitconf = std::string(homedir) + "/.gitconfig";
FILE* fgitconf = fopen(path_gitconf.c_str(), "r");
// Parse lines for name and email entries
```

**Parsed Information:**
- **User Name:** From `[user] name = ...` entries
- **User Email:** From `[user] email = ...` entries
- **Error Handling:** Fatal error if Git config unavailable

## HTTP Request Implementation

### Status Endpoint Query
```cpp
curlpp::options::Url mystaturl(statusurlstr);
curlpp::Easy mystatusreq;
mystatusreq.setOpt(mystaturl);

// Set User-Agent header
std::string headua("User-Agent: RefPerSys/");
headua += rps_shortgitid;
statheaders.push_back(headua);

// Configure response handling
std::ostringstream outs;
curlpp::options::WriteStream ws(&outs);
mystatusreq.setOpt(ws);

// Execute request
mystatusreq.perform();
```

**Request Features:**
- **URL Construction:** Base URL + "/status" endpoint
- **User-Agent:** Identifies RefPerSys version
- **Response Capture:** Streams response to string buffer
- **Error Handling:** curlpp exceptions for network issues

## JSON Response Processing

### Status Validation
```cpp
Json::Value jstatus;
Json::CharReaderBuilder jsonreaderbuilder;
std::unique_ptr<Json::CharReader> pjsonreader(jsonreaderbuilder.newCharReader());

if (!pjsonreader->parse(startp, endp, &jstatus, &errstr))
  RPS_FATALOUT("failed to parse JSON response");

if (!jstatus.isObject())
  RPS_FATALOUT("status response is not a JSON object");
```

**Validation Steps:**
- **JSON Parsing:** Validates response syntax
- **Object Structure:** Ensures response is JSON object
- **Error Reporting:** Detailed parsing failure information

## Intended Publishing Workflow

### Planned Implementation
```cpp
/** TODO:
 * This function should do one or a few HTTP requests to the web service
 * Initially on http://refpersys.org:8080/ probably
 * Sending the various public data in __timestamp.c as HTTP POST parameters
 * Parse the .git/config file for user name and email
 **/
```

**Publishing Data:**
- **System Metadata:** Version, Git information, build details
- **User Information:** Name and email from Git configuration
- **System State:** Current RefPerSys configuration and status
- **Performance Metrics:** Optional system performance data

## Web Service Architecture

### Expected Endpoints
- **`/status`:** GET request for service status and capabilities
- **`/publish`:** POST request for data submission (planned)
- **`/query`:** GET request for data retrieval (future)

### Data Exchange Format
- **Request Format:** HTTP POST with form data or JSON payload
- **Response Format:** JSON objects with status and result information
- **Authentication:** User-Agent based identification
- **Error Handling:** HTTP status codes and JSON error objects

## Error Handling

### Network Errors
- **Connection Failures:** curlpp exceptions for network issues
- **Timeout Handling:** Configurable request timeouts
- **SSL/TLS Errors:** Certificate validation and SSL configuration

### Data Validation Errors
- **JSON Parsing:** Syntax error detection and reporting
- **Schema Validation:** Response structure verification
- **Data Integrity:** Content validation and checksums

## Integration Points

### System Components
- **Version System:** Integration with `__timestamp.c` data
- **Git Integration:** User identification from repository configuration
- **Debug System:** Debug logging for request/response tracing
- **Error System:** Fatal error handling for critical failures

### External Services
- **RefPerSys Web Service:** Primary publication target
- **Local Development:** Support for localhost debugging endpoints
- **Alternative Services:** Framework for multiple publication targets

## Performance Considerations

### Request Optimization
- **Connection Reuse:** curl handle reuse for multiple requests
- **Timeout Configuration:** Reasonable timeouts for responsiveness
- **Compression:** Automatic gzip/deflate support
- **Caching:** Response caching for repeated requests

### Memory Management
- **Stream Buffering:** Efficient response buffering
- **Resource Cleanup:** Proper curl handle cleanup
- **Memory Limits:** Response size limits for security

## Security Considerations

### Data Transmission
- **HTTPS Requirement:** SSL/TLS encryption for data transmission
- **Certificate Validation:** Proper certificate chain verification
- **Data Sanitization:** Safe encoding of user data
- **Privacy Protection:** Minimal data transmission principles

### Authentication
- **User-Agent Identification:** Version-based service identification
- **Git Information:** User attribution without sensitive data
- **Access Control:** Service-side authorization mechanisms

## File Status

**Status:** Partially Implemented Framework
**Date:** 2019-2024
**Purpose:** Web service integration and data publishing

## Limitations and Future Work

### Current Limitations
1. **Incomplete Publishing:** POST request implementation missing
2. **Single Endpoint:** Only status endpoint implemented
3. **No Authentication:** Basic User-Agent identification only
4. **Limited Error Recovery:** Fatal errors on most failures

### Future Enhancements
1. **Complete Publishing:** Implement full data submission workflow
2. **Authentication:** Add proper authentication mechanisms
3. **Batch Operations:** Support for bulk data operations
4. **Monitoring:** Request/response metrics and logging
5. **Configuration:** Configurable endpoints and timeouts

## Summary

The `curl_rps.cc` file provides the foundation for RefPerSys's web service integration, implementing HTTP communication using libcurl and curlpp. While the core infrastructure for version reporting, library initialization, and basic HTTP requests is complete, the primary publishing functionality remains unimplemented. The file demonstrates a well-structured approach to web service integration with proper error handling, JSON processing, and Git integration for user identification. The framework is designed to support data publishing to RefPerSys web services, enabling system monitoring, data sharing, and collaborative features. Despite being incomplete, the implementation provides a solid foundation for web service integration that could be extended to support the full range of planned publishing and data exchange capabilities.