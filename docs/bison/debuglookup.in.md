# BisonC++ Debug Lookup Template (`debuglookup.in`)

## Overview

The `debuglookup.in` file provides debug output for the parser's state lookup operations. It generates detailed logging when the parser performs state transitions during parsing, showing the lookup process and resulting actions.

## File Structure

### Lookup Debug Implementation
```cpp
#pragma message "RefPerSys bisonc++ debuglookup.in"

if (d_debug_)
{
    s_out_ <<  "encountered " << symbol_(token_()) << " in state " <<
                state_() << ": ";

    if (action < 0)             // a reduction was found
        s_out_ << ": reducing by rule " << -action;
    else if (action == 0)
        s_out_ <<  "ACCEPT";
    else
        s_out_ <<  "next state: " << action;

    s_out_ << '\n' << dflush_;
}
```

## Debug Output Details

### Lookup Process Logging
The debug output shows the state lookup process:

```cpp
// Format: "encountered TOKEN in state STATE: ACTION"
encountered 'IDENTIFIER' in state 5: next state: 12
encountered '+' in state 12: reducing by rule 3
encountered 'EOF' in state 0: ACCEPT
```

### Action Type Interpretation
- **Positive Action**: Shift to next state (`next state: N`)
- **Negative Action**: Reduce by rule (`reducing by rule N`)
- **Zero Action**: Accept input (`ACCEPT`)

## Integration with Parser Operations

### Lookup Function Enhancement
```cpp
// In @Base::lookup_() from bisonc++.cc
int @Base::lookup_() const {
    SR_ const *sr = s_state[d_state];
    SR_ const *last = sr + sr->d_lastIdx;

    for ( ; ++sr != last; ) {
        if (sr->d_token == d_token)
            return sr->d_action;
    }

    if (sr == last) {
        // Default reduction or error
        $insert 12 debug "LOOKUP: [" << state_() << ", " << symbol_(d_token) << "] -> default reduce using rule " << -sr->d_action
        return sr->d_action;
    }

    // Not at last element - return action
    int action = sr->d_action;
    $insert 0 debuglookup  // Inserts the debug code above
    return action;
}
```

## Debug Output Examples

### Shift Operations
```
encountered 'IDENTIFIER' in state 3: next state: 8
encountered '(' in state 8: next state: 12
```

### Reduction Operations
```
encountered '+' in state 15: reducing by rule 5
encountered ')' in state 20: reducing by rule 8
```

### Accept Operations
```
encountered end-of-file in state 2: ACCEPT
```

### Default Reductions
```
LOOKUP: [7, ';'] -> default reduce using rule 12
```

## Performance Impact

### Conditional Execution
- **Debug Only**: Code only executes when `d_debug_` is true
- **Minimal Overhead**: When disabled, just a boolean check
- **Buffered Output**: Uses `s_out_` for efficient I/O

### Output Frequency
- **Per Lookup**: Called for every state transition
- **High Frequency**: Can generate significant output for complex parses
- **Configurable**: Can be disabled per debug level

## Customization Options

### Enhanced Debug Output
Users can modify the debug output for additional information:

```cpp
// Custom debuglookup.in
if (d_debug_)
{
    s_out_ << "LOOKUP: state " << state_() 
           << " + token " << symbol_(token_())
           << " (" << token_() << ") -> ";
    
    if (action < 0)
        s_out_ << "REDUCE rule " << -action;
    else if (action == 0)
        s_out_ << "ACCEPT";
    else
        s_out_ << "SHIFT to state " << action;
    
    s_out_ << " [stack depth: " << stackSize_() << "]" << '\n' << dflush_;
}
```

### Selective Debug Output
```cpp
// Only debug certain states or tokens
if (d_debug_ && (state_() >= 10 || token_() == SPECIAL_TOKEN))
{
    // Debug output for specific cases
    s_out_ << "Special case: " << symbol_(token_()) 
           << " in state " << state_() << std::endl << dflush_;
}
```

## Integration with Debug System

### Debug Level Control
```cpp
// Different debug levels can control lookup verbosity
if (d_debug_ && debugLevel >= DEBUG_LOOKUP)
{
    // Detailed lookup information
}
```

### Cross-Reference with Other Debug
- **`debugfunctions2.in`**: Provides `symbol_()` for token names
- **`debugfunctions1.in`**: Provides `dflush_()` for output
- **`debugdecl.in`**: Declares the debug stream

## Error Case Handling

### Unmatched Tokens
When no transition is found for a token, the debug output shows the failed lookup before error recovery begins.

### Default Reductions
The separate debug insertion at line 12 shows when default reductions are used, providing insight into grammar ambiguity resolution.

This debug lookup implementation provides detailed visibility into the parser's decision-making process, showing exactly how tokens are matched to state transitions and what actions result from each lookup operation.