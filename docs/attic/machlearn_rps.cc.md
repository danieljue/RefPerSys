# Machine Learning Integration Experiment (`attic/machlearn_rps.cc`)

**File Path:** `attic/machlearn_rps.cc`

## Overview

This file represents an experimental integration of machine learning capabilities into RefPerSys using the MLPACK library. It demonstrates how RefPerSys could be extended with ML functionality through custom payloads that store matrices and labels, with persistence support for CSV data files.

## Dependencies and Requirements

### External Libraries
```cpp
// Debian packages required:
#include "mlpack.hpp"  // MLPACK machine learning library

//@@PKGCONFIG mlpack  // Pkg-config integration
```

**Required Packages:**
- `mlpack-bin mlpack-doc libmlpack-dev` - MLPACK core library
- `libensmallen-dev` - Optimization library used by MLPACK

### Version Requirements
```cpp
// Version checking at compile time
#if MLPACK_VERSION_MAJOR != 4 || MLPACK_VERSION_MINOR != 5
#error mlpack.org version 4.5 is required
#endif
```

**MLPACK Version:** Specifically requires version 4.5 (build issues with 4.4)

## Core Components

### Rps_PayloadMachLearn Class

```cpp
class Rps_PayloadMachLearn : public Rps_Payload {
    // MLPACK data structures
    arma::mat _machlearn_matrix;           // Feature matrix
    arma::Row<size_t> _machlearn_labels;   // Label data
    
    // File path methods
    std::string matrix_file_path() const;
    std::string labels_file_path() const;
};
```

**Purpose:** Custom payload for storing machine learning datasets within RefPerSys objects

### Data Storage Design

#### Matrix Storage
```cpp
arma::mat _machlearn_matrix;  // Armadillo matrix for features
```

- **Type:** Armadillo matrix (column-major, double precision)
- **Purpose:** Stores feature vectors for ML algorithms
- **Format:** CSV files for persistence

#### Label Storage
```cpp
arma::Row<size_t> _machlearn_labels;  // Row vector for labels
```

- **Type:** Armadillo row vector (size_t for class labels)
- **Purpose:** Stores classification targets or regression values
- **Format:** CSV files for persistence

## File Path Management

### Matrix File Paths
```cpp
std::string Rps_PayloadMachLearn::matrix_file_path() const {
    char buf[80];
    snprintf(buf, sizeof(buf), "%s_machlearn_mat.csv",
             owner()->string_oid().c_str());
    return std::string(buf);
}
```

**Naming Convention:** `{object_oid}_machlearn_mat.csv`

### Labels File Paths
```cpp
std::string Rps_PayloadMachLearn::labels_file_path() const {
    char buf[80];
    snprintf(buf, sizeof(buf), "%s_machlearn_lab.csv",
             owner()->string_oid().c_str());
    return std::string(buf);
}
```

**Naming Convention:** `{object_oid}_machlearn_lab.csv`

## Persistence Implementation

### JSON Serialization
```cpp
void dump_json_content(Rps_Dumper* du, Json::Value& jv) const {
    // Save matrix to temporary CSV file
    std::string tempmatrix = rps_dumper_temporary_path(du, matrixpath);
    bool matrixok = mlpack::data::Save(tempmatrix, _machlearn_matrix, 
                                      true, false, mlpack::data::FileType::CSVASCII);
    
    // Save labels to temporary CSV file  
    std::string templabels = rps_dumper_temporary_path(du, labelspath);
    bool labelsok = mlpack::data::Save(templabels, _machlearn_labels,
                                      true, false, mlpack::data::FileType::CSVASCII);
    
    // Record file paths in JSON
    jv["machlearn_matrix"] = matrixpath;
    jv["machlearn_labels"] = labelspath;
}
```

**Process:**
1. Generate temporary CSV files in dump directory
2. Use MLPACK's CSV export functionality
3. Backup existing files if present
4. Store file paths in JSON metadata

### Data Loading
```cpp
void rpsldpy_machlearn(Rps_ObjectZone* obz, Rps_Loader* ld, 
                      const Json::Value& jv, Rps_Id spacid, unsigned lineno) {
    auto paylmachlearn = obz->put_new_plain_payload<Rps_PayloadMachLearn>();
    
    // Load matrix from CSV
    bool matrixok = mlpack::data::Load(matrixpath, paylmachlearn->_machlearn_matrix,
                                      true, false, mlpack::data::FileType::CSVASCII);
    
    // Load labels from CSV
    bool labelsok = mlpack::data::Load(labelspath, paylmachlearn->_machlearn_labels,
                                      true, false, mlpack::data::FileType::CSVASCII);
}
```

**Loading Process:**
1. Create payload instance
2. Load matrix data from CSV file
3. Load label data from CSV file
4. Fatal error on loading failures

## Integration Points

### Build System Integration
```cpp
//@@PKGCONFIG mlpack
```

**Pkg-config:** Automatic dependency resolution for MLPACK library

### RefPerSys Object System
```cpp
static Rps_ObjectRef make_machlearn_object(Rps_CallFrame* callframe, 
                                          Rps_ObjectRef obclass=nullptr, 
                                          Rps_ObjectRef obspace=nullptr);
```

**Factory Method:** Creates new objects with ML payloads

### Garbage Collection
```cpp
virtual void gc_mark(Rps_GarbageCollector& gc) const {
    #warning incomplete Rps_PayloadMachLearn::gc_mark
}
```

**Status:** GC marking not implemented (marked as incomplete)

## Usage Scenarios

### Data Storage
```cpp
// Create ML object
Rps_ObjectRef ml_obj = Rps_PayloadMachLearn::make_machlearn_object(&_, class_obj, space_obj);

// Access payload
auto payload = ml_obj->get_dynamic_payload<Rps_PayloadMachLearn>();
payload->_machlearn_matrix = my_feature_matrix;
payload->_machlearn_labels = my_labels;
```

### Persistence
```cpp
// Automatic persistence during dump
rps_dump_into(".", &_);
```

**Result:** Creates `{oid}_machlearn_mat.csv` and `{oid}_machlearn_lab.csv` files

### Loading
```cpp
// Automatic loading during system startup
rps_load_from(".");
```

## Technical Architecture

### MLPACK Integration
- **Library:** Uses Armadillo for matrix operations
- **File I/O:** CSV format for data exchange
- **Version:** Specific version requirements (4.5)

### RefPerSys Integration
- **Payload System:** Custom payload type for ML data
- **Persistence:** CSV files with JSON metadata
- **Object Model:** ML objects as first-class RefPerSys entities

## Limitations and Incomplete Features

### Marked Incomplete
```cpp
#warning incomplete Rps_PayloadMachLearn destructor
#warning incomplete Rps_PayloadMachLearn::gc_mark
#warning very incomplete file machlearn_rps.cc
```

**Missing Features:**
1. **Destructor:** Resource cleanup not implemented
2. **GC Support:** Garbage collection marking incomplete
3. **ML Algorithms:** No actual ML algorithm implementations
4. **Data Validation:** No validation of matrix/label consistency
5. **Error Handling:** Limited error recovery

### Functional Gaps
- **Algorithm Integration:** No actual ML algorithms implemented
- **Data Preprocessing:** No data normalization or feature scaling
- **Model Persistence:** No trained model storage
- **API Integration:** No RefPerSys API for ML operations

## File Status

**Status:** Experimental/Incomplete
**Date:** 2024
**Significance:** Demonstrates potential for ML integration but never completed

## Potential Extensions

### Algorithm Integration
```cpp
// Potential additions:
// - k-NN classification
// - Decision trees
// - Neural networks
// - Clustering algorithms
// - Dimensionality reduction
```

### API Enhancements
```cpp
// Potential methods:
// - train_model(algorithm, parameters)
// - predict(input_data)
// - evaluate_model(test_data)
// - save_model(filename)
// - load_model(filename)
```

### Data Management
```cpp
// Potential features:
// - Data preprocessing pipelines
// - Feature selection
// - Cross-validation support
// - Hyperparameter optimization
```

## Comparison with Other Experiments

### vs. Qt GUI (`qtgui-qrps.cc`)
- **Qt:** Complete GUI implementation with working features
- **ML:** Infrastructure only, no working ML functionality

### vs. ANTLR Grammar (`gramrepl_antlr_rps.g4`)
- **ANTLR:** Complete grammar specification
- **ML:** Partial implementation with data structures

### vs. OCaml Integration (`Create-RefPerSys.ocaml`)
- **OCaml:** Minimal proof-of-concept
- **ML:** Substantial infrastructure with persistence

## Future Development Potential

This ML integration experiment shows promise for extending RefPerSys with data science capabilities:

1. **Data Science Integration:** Foundation for statistical computing
2. **Algorithm Library:** Could host various ML implementations
3. **Big Data Support:** Scalable data structures for large datasets
4. **Research Platform:** Environment for ML algorithm development

The implementation provides a solid foundation with proper persistence and object integration, though it would need completion of the core ML functionality and garbage collection support to be production-ready.

## Build Requirements

### Compilation
```bash
# Requires MLPACK development headers
g++ -std=c++17 machlearn_rps.cc -lmlpack -larmadillo -o test_ml
```

### Runtime Dependencies
- **MLPACK:** Machine learning library
- **Armadillo:** Linear algebra library
- **RefPerSys:** Core system integration

This machine learning experiment represents an ambitious attempt to add data science capabilities to RefPerSys, providing the infrastructure for ML data management and persistence while leaving the algorithmic implementation as future work.