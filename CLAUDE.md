# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a research/documentation project that analyzes and documents data structure implementations (especially dynamic arrays) in production C codebases. The project studies how popular open-source C libraries implement memory-managed data structures.

**This is a documentation-only project** - the main work involves analyzing code in the `/repos` submodules and writing notes in `/notes`.

## Repository Structure

```
c-abstractions/
├── notes/           # Documentation of data structures found in each library
│   ├── curl.md      # curl library data structures
│   └── jq.md        # jq library data structures
└── repos/           # Git submodules of C projects being analyzed
    ├── curl/        # HTTP client library
    ├── jq/          # JSON query processor
    ├── libgit2/     # Git library
    ├── inih/        # INI file parser
    └── mongoose/    # Embedded web server
```

## Dynamic Array Criteria

A data structure qualifies as a dynamic array candidate if it meets:

**Structural requirements:**
- A `struct` containing a pointer field, length field, and capacity field

**Behavioral requirements:**
- Capacity growth via `realloc`
- Insert logic increments length
- Access via `data[index]` or `data[len]`

**Excluded:**
- Macro-only arrays (stb_ds, uthash)
- Arrays without capacity tracking
- Pure stack arrays

## Documentation Format

When documenting data structures in `notes/`, follow the existing format:
1. Summary table with data structure name, type, and file location (with line numbers)
2. Struct definitions with file paths and line numbers
3. Key operations (insert, reallocation) with implementation code

Example file reference format: `repos/curl/lib/llist.h:33-41`

## Working with the Repos

The `/repos` directory contains git submodules. Use grep to search across them:
```bash
grep -r "pattern" repos/curl/lib/
```

Common patterns to look for:
- `realloc` calls for dynamic growth
- Structs with `length`/`len`/`size` and `capacity`/`alloc`/`alloc_length` fields
- Flexible array members (`data[]`, `elements[]`)
