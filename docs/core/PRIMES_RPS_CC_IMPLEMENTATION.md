# Primes Implementation Analysis (`primes_rps.cc`)

## Overview

The `primes_rps.cc` file implements prime number utilities for the Reflective Persistent System (RefPerSys), providing efficient access to a curated set of prime numbers and functions to find primes above or below given values. This module serves as a mathematical foundation for various system components that require prime numbers, such as hash functions, random number generation, and data structure sizing.

## File Structure and Dependencies

### Header and License
- **License**: GPL-3.0-or-later
- **Authors**: Basile Starynkevitch, Abhishek Chakravarti
- **Copyright**: © 2019-2025 The Reflective Persistent System Team

### Includes and External Declarations
```cpp
#include <cstdint>

// Git versioning information
extern "C" const char rps_primes_gitid[];
extern "C" const char rps_primes_date[];
extern "C" const char rps_primes_shortgitid[];
extern "C" const char rps_primes_timestamp[];

// External function declarations
extern "C" int64_t rps_prime_above(int64_t n);
extern "C" int64_t rps_prime_below(int64_t n);
```

### Key Dependencies
- **cstdint**: Fixed-width integer types for 64-bit prime numbers
- **primesieve**: External tool used to generate the prime number table
- **makeprimes.c**: Custom utility for processing primesieve output

## Prime Number Array

### Array Structure and Generation
```cpp
// Computer-generated array of 265 prime numbers
static const int64_t rps_primes_tab[] = {
  // First few primes (manually listed for clarity)
  2, 3, 5, 7, 11, 13, 17, 19, 23, 29,
  // ... continues up to 265 primes ...
  2176373521033  // Largest prime: ~2.18 × 10^12
};
```

**Array Characteristics:**
- **Size**: 265 prime numbers
- **Range**: From 2 to ~2.18 × 10^12 (2.18 trillion)
- **Generation**: Computed using `primesieve -t18 -p 2 2333444555666`
- **Performance**: Generated on AMD Ryzen Threadripper 2970WX in 13,439 seconds
- **Coverage**: 0.00000% of primes up to the limit (very sparse sampling)

### Generation Process
```bash
# Command used to generate primes
makeprimes 2333444555666 10 'primesieve -t18 -p'

# Where:
# 2333444555666 = upper limit for prime generation
# 10 = density parameter (every 10th prime or so)
# primesieve -t18 -p = use 18 threads, print primes
```

**Purpose**: Provides a carefully selected subset of primes spanning a wide range, optimized for practical use in computational algorithms.

## Core Functions

### Prime Access by Rank
```cpp
int64_t rps_prime_ranked(int rk)
{
  constexpr unsigned numprimes = sizeof(rps_primes_tab) / sizeof(rps_primes_tab[0]);
  if (rk < 0) return 0;
  if (rk < (int)numprimes) return rps_primes_tab[rk];
  return 0;
}
```

**Purpose**: Direct access to primes by their index in the array (0-based).
- **Parameters**: `rk` - rank/index (0 = first prime = 2)
- **Returns**: Prime at given rank, or 0 if out of bounds
- **Time Complexity**: O(1) - direct array access

### Finding Prime Above a Number
```cpp
int64_t rps_prime_above(int64_t n)
{
  // Binary search to find insertion point
  int lo = 0, hi = numprimes;
  while (lo + 8 < hi) {
    int md = (lo + hi) / 2;
    if (rps_primes_tab[md] > n) hi = md;
    else lo = md;
  }

  // Linear search in narrowed range
  for (int ix = lo; ix < hi; ix++)
    if (rps_primes_tab[ix] > n)
      return rps_primes_tab[ix];

  return 0; // No prime found above n
}
```

**Algorithm**: Hybrid binary search + linear search
1. **Binary Search Phase**: Narrow down to ~8-element range using binary search
2. **Linear Search Phase**: Scan the narrowed range for the first prime > n
3. **Edge Cases**: Returns 2 for n < 2, 0 if n >= largest prime

### Finding Prime Below a Number
```cpp
int64_t rps_prime_below(int64_t n)
{
  // Similar binary search setup
  int lo = 0, hi = numprimes;
  while (lo + 8 < hi) {
    int md = (lo + hi) / 2;
    if (rps_primes_tab[md] > n) hi = md;
    else lo = md;
  }

  // Linear search backward from narrowed range
  for (int ix = hi; ix >= 0; ix--)
    if (rps_primes_tab[ix] < n)
      return rps_primes_tab[ix];

  return 0; // No prime found below n
}
```

**Algorithm**: Binary search + backward linear search
- **Purpose**: Find largest prime < n
- **Strategy**: Search backward from the binary search boundary

## Rank Information Functions

### Prime Above with Rank
```cpp
int64_t rps_prime_greaterequal_ranked(int64_t n, int* prank)
{
  // Similar to rps_prime_above but also returns rank
  // prank parameter gets the index of the returned prime
  // Returns 0 if no prime >= n exists
}
```

**Purpose**: Find smallest prime >= n and optionally return its rank/index.

### Prime Below with Rank
```cpp
int64_t rps_prime_lessequal_ranked(int64_t n, int* prank)
{
  // Similar to rps_prime_below but also returns rank
  // prank parameter gets the index of the returned prime
  // Returns 0 if no prime <= n exists
}
```

**Purpose**: Find largest prime <= n and optionally return its rank/index.

## Algorithm Analysis

### Binary Search Optimization
```cpp
// Hybrid approach: binary search + linear search
while (lo + 8 < hi) {
  int md = (lo + hi) / 2;
  if (rps_primes_tab[md] > n) hi = md;
  else lo = md;
}
```

**Rationale**: 
- **Binary search** reduces 265 elements to ~8 elements quickly
- **Linear search** handles the final comparison efficiently
- **Threshold of 8** balances binary search overhead vs linear search simplicity

### Performance Characteristics
- **Time Complexity**: O(log N) for binary search + O(1) for linear search = O(log N)
- **Space Complexity**: O(1) - fixed-size array
- **Lookup Time**: ~10-15 operations for worst case
- **Memory Usage**: 265 × 8 = 2,120 bytes for the array

### Edge Case Handling
```cpp
// Boundary conditions
if (n >= rps_primes_tab[numprimes - 1]) return 0;  // Too large
if (n < 2) return 2;                                // Too small, return first prime
```

**Purpose**: Robust handling of inputs outside the supported range.

## Usage Patterns and Examples

### Basic Prime Lookup
```cpp
// Get prime by rank
int64_t prime5 = rps_prime_ranked(4);  // Returns 11 (0-based index)

// Find next prime above a number
int64_t next = rps_prime_above(10);    // Returns 11
int64_t next2 = rps_prime_above(11);   // Returns 13

// Find previous prime below a number
int64_t prev = rps_prime_below(10);    // Returns 7
int64_t prev2 = rps_prime_below(7);    // Returns 5
```

### Prime Lookup with Rank Information
```cpp
int rank;
int64_t prime = rps_prime_greaterequal_ranked(10, &rank);
// prime = 11, rank = 4

int64_t prime2 = rps_prime_lessequal_ranked(10, &rank);
// prime2 = 7, rank = 3
```

### Hash Table Sizing
```cpp
// Common pattern: size hash tables to prime numbers
size_t desired_size = 1000;
size_t table_size = rps_prime_above(desired_size);
// table_size will be next prime >= 1000
```

### Random Number Generation
```cpp
// Use primes for random number generation seeds
int64_t seed_prime = rps_prime_ranked(random_index % 265);
// Guaranteed prime seed for better randomization
```

## Array Statistics

### Prime Distribution
- **First Prime**: 2 (index 0)
- **Last Prime**: 2,176,373,521,033 (index 264)
- **Range**: 12 orders of magnitude (2 to 2.18 × 10^12)
- **Density**: Very sparse - only 265 primes out of ~1.8 × 10^11 possible numbers

### Size Classes
```cpp
// Small primes (first 25)
2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97

// Large primes (last 10)
2176373521033, 1978521382723, 1798655802451, 1635141638587, 1486492398631,
1351356725987, 1228506114527, 1116823740479, 1015294309507, 922994826779
```

## Design Rationale

### Why Precomputed Prime Array?
**Why not compute primes on demand?**
- **Performance**: O(1) lookup vs O(sqrt(n)) primality testing
- **Predictability**: Fixed, known prime values for reproducible behavior
- **Simplicity**: No primality testing algorithms needed
- **Reliability**: No computational errors in prime determination

### Why Sparse Sampling?
**Why not include all primes up to a limit?**
- **Memory Efficiency**: 265 primes use minimal space
- **Wide Range**: Covers 12 orders of magnitude
- **Practical Coverage**: Sufficient for most algorithmic needs
- **Generation Cost**: Dense prime tables would be impractically large

### Why Hybrid Search Algorithm?
**Why binary search + linear search?**
- **Optimality**: Binary search for large arrays, linear for small ranges
- **Simplicity**: Avoid complex interpolation search
- **Cache Efficiency**: Small linear searches are cache-friendly
- **Maintainability**: Easy to understand and verify correctness

### Why 64-bit Integers?
**Why int64_t instead of smaller types?**
- **Future-Proofing**: Accommodates very large prime requirements
- **Compatibility**: Matches system integer sizes
- **Hash Functions**: Large primes needed for good hash distribution
- **Mathematical Operations**: Prevents overflow in calculations

## Applications in RefPerSys

### Hash Function Support
```cpp
// Primes used for hash table sizing and universal hashing
size_t hash_table_size = rps_prime_above(desired_capacity);
uint64_t hash_multiplier = rps_prime_ranked(some_index);
```

### Random Number Generation
```cpp
// Prime seeds for pseudo-random number generators
int64_t rng_seed = rps_prime_ranked(thread_id % 265);
```

### Data Structure Sizing
```cpp
// Ensure data structures use prime sizes for better distribution
size_t optimal_size = rps_prime_above(minimum_size);
```

### Cryptographic Operations
```cpp
// Large primes for cryptographic parameter generation
int64_t crypto_prime = rps_prime_ranked(high_rank_index);
```

This implementation provides a lightweight, efficient prime number utility that supports various algorithmic needs within RefPerSys while maintaining minimal memory footprint and fast lookup times.