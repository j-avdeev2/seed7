# Segmentation Fault Risk Analysis

This document records potential crash sources discovered during a brief review of the Seed7 runtime sources.  Each entry highlights the code location, explains why it can provoke a segmentation fault, and sketches possible remediation ideas.

## 1. Dangling pointer dereference in `unsetenv7`
- **Location:** `src/cmd_unx.c`, lines 260-292.【F:src/cmd_unx.c†L205-L292】
- **Issue:** The function remembers the address of an environment entry in the local pointer `found` while it scans the `environ7` array.  After the scan it shrinks the array with `realloc()`.  If `realloc()` moves the array, `found` becomes a dangling pointer into freed storage.  The subsequent `free(*found);` and `*found = environ7[nameCount - 1];` dereference that stale pointer, which can crash the interpreter immediately.
- **Suggested fix:** Store the array index instead of the raw pointer, or defer any `realloc()` until after the entry has been removed with pointer arithmetic based on the new array base address.

## 2. Unbounded concatenation in `list_node_names`
- **Location:** `src/traceutl.c`, lines 1372-1499.【F:src/traceutl.c†L1372-L1499】
- **Issue:** `trace_nodes()` builds diagnostic strings in a fixed `char buffer[4096]` and repeatedly calls `list_node_names()` to append identifiers and attributes.  The helper appends numerous strings with `strcat()` without guarding the remaining capacity.  Deeply nested declarations or long identifier names can easily exceed 4096 bytes, overflowing the buffer and corrupting the stack, which ultimately produces a segmentation fault.
- **Suggested fix:** Track the current buffer length, limit each append to the available space (e.g., with `strncat`/`snprintf`), or switch to dynamically sized `striType`/`std::string` equivalents for diagnostic output.

