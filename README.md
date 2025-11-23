# World's Simplest Database

A simple key-value store implementation demonstrating the evolution from the most basic approach to modern database concepts. This educational project showcases four progressively optimized versions, each solving different performance challenges.

## 📚 Documentation

**For a detailed explanation of how the database works and the evolution from V1 to V4, read our blog articles:**

- 🇩🇪 [**BLOGARTIKEL.md**](BLOGARTIKEL.md) - *Wie funktionieren Datenbanken? Eine Reise durch die Evolution der einfachsten Key-Value-Datenbank*
- 🇬🇧 [**BLOGARTIKEL-eng.md**](BLOGARTIKEL-eng.md) - *How Do Databases Work? A Journey Through the Evolution of the Simplest Key-Value Database*

The blog articles explain the fundamental concepts, trade-offs, and include detailed benchmark results.

## 🎯 Overview

This project implements a simple key-value database with four different versions, each solving the fundamental challenge: **writing is easy (just append), but reading efficiently requires optimization**.

### The Core Problem

The simplest database just appends everything to a file. Writing is O(1) and optimal. But reading requires scanning the entire file - O(n) complexity. With millions of entries, this becomes impractical.

### The Evolution

Each version addresses this challenge differently:

| Version | Write | Read | Key Innovation |
|---------|-------|------|----------------|
| **V1** | O(1) | O(n) | Proof that writing is simple |
| **V2** | O(1) | O(n) | Binary format optimization |
| **V3** | O(1) | O(1) | In-memory index for instant lookups |
| **V4** | O(log n) | O(log n×m) | SSTables with LSM-tree principles |

## 🚀 Quick Start

### Requirements

- .NET 8.0 or later

### Build

```bash
dotnet build
```

### Interactive Usage

```bash
cd worldssimplestdb.console
dotnet run
```

Then use commands like:
```
db> set name "John Doe"
OK
db> get name
John Doe
db> exit
```

## 📊 The Four Versions

### Version 1: Text-Based Append-Only

**The simplest possible implementation** - just append text to a file.

- ✅ **Write**: O(1) - Simple append
- ❌ **Read**: O(n) - Full file scan
- 📝 **Format**: `key;value\n` (human-readable)
- 🎯 **Use case**: Prototyping, very small datasets

### Version 2: Binary Format

**Optimized storage format** - more efficient, but still requires scanning.

- ✅ **Write**: O(1) - Binary append
- ❌ **Read**: O(n) - Still full scan, but faster parsing
- 📝 **Format**: Length-prefixed binary `[keyLen][keyData][valueLen][valueData]`
- 🎯 **Use case**: Small to medium datasets, binary format preferred

### Version 3: In-Memory Index

**The breakthrough** - instant lookups via RAM index.

- ✅ **Write**: O(1) - Append + index update
- ✅ **Read**: O(1) - Direct seek via index (no scan!)
- 📝 **Format**: Binary + `Dictionary<string, long>` index
- ⚠️ **Trade-off**: RAM consumption (~50-100 bytes per key)
- 🎯 **Use case**: Many reads, enough RAM available

### Version 4: SSTables (LSM-Tree)

**Modern database architecture** - inspired by LevelDB and RocksDB.

- ✅ **Write**: O(log n) - Sorted dictionary in memory
- ✅ **Read**: O(log n×m) - Binary search in sorted SSTables
- 📝 **Architecture**: Memtable + SSTables + Write-Ahead Log (WAL)
- ✨ **Features**: Crash recovery, non-blocking flush, immutable files
- 🎯 **Use case**: Production-ready, scalable applications

**Architecture:**
```
┌──────────────┐
│  Memtable    │ ← New writes (in-memory, sorted)
└──────┬───────┘
       │ On overflow: Flush
       ▼
┌──────────────┐
│ SSTable 1    │ ← Newest (immutable, sorted)
├──────────────┤
│ SSTable 2    │
└──────────────┘
```

## 🧪 Benchmarking

The project includes a built-in benchmark tool to test performance:

```bash
cd worldssimplestdb.filldata

# Standard benchmark (200MB, all versions, 10 iterations)
dotnet run benchmark

# Custom benchmark
dotnet run benchmark [fillSize] [iterations] [v1|v2|v3|v4|all]

# Examples:
dotnet run benchmark 10mb          # 10MB data
dotnet run benchmark 20             # 20 iterations
dotnet run benchmark v3 v4          # Only test V3 and V4
dotnet run benchmark 1gb 20 v4      # 1GB, 20 iterations, V4 only
```

### Example Results (200MB data, ~200,000 entries)

| Version | Write | Read (avg) | Startup |
|---------|-------|------------|---------|
| V1 | ~2-3s | **1,625ms** | <1ms |
| V2 | ~2-3s | **162ms** | <1ms |
| V3 | ~2-3s | **<0.1ms** | ~500-2000ms* |
| V4 | ~3-4s | **15.8ms** | ~100-500ms |

\* V3 requires one-time index building on startup

**Key Insights:**
- V1/V2: Writing is fast, but reading becomes impractical with large datasets
- V3: Fastest reads, but RAM-intensive and longer startup time
- V4: Best balance - fast reads, moderate RAM, excellent scalability

For detailed benchmark results and analysis, see the [blog articles](BLOGARTIKEL.md).

## 🛠️ Projects

### `worldssimplestdb.console`
Interactive command-line application supporting all four versions.

**Commands:**
- `set <key> <value>` - Store a key-value pair
- `get <key>` - Retrieve value by key
- `help` - Show available commands
- `exit` / `quit` - Exit (V4 automatically flushes and cleans up)

### `worldssimplestdb.filldata`
Utility for generating test data and running benchmarks.

**Fill database:**
```bash
dotnet run <version> <size> <minValueLen> <maxValueLen>
# Example: dotnet run v4 200mb 50 200
```

**Run benchmark:**
```bash
dotnet run benchmark [options]
```

## 📁 File Structure

### V1 Database
- `database.txt` - Text format, human-readable

### V2/V3 Database
- `database.bin` - Binary format
- V3 additionally builds an in-memory index on startup

### V4 Database
- `sstables/sstable_*.sst` - Immutable sorted files
- `wal.log` - Write-Ahead Log for crash recovery

All database files are stored in the solution root directory.

## 🔍 Technical Highlights

### V4 Implementation Details

- **Memtable**: `SortedDictionary<string, string>` (Red-Black Tree) for O(log n) operations
- **SSTables**: Immutable, sorted files with sparse index for binary search
- **WAL**: Write-Ahead Log ensures crash safety
- **Double-buffering**: Non-blocking writes during flush operations
- **Atomic writes**: Temporary files + rename for data integrity

## 🎓 What You'll Learn

This project demonstrates:

1. **Why writing is simple**: Log-based appending is already optimal
2. **Why reading is hard**: Without indexes, you must scan everything
3. **Different optimization strategies**: Format optimization vs. indexing vs. sorting
4. **Trade-offs**: Performance vs. memory vs. complexity
5. **Modern database concepts**: LSM-trees, SSTables, WAL, compaction (prepared)

## 🔮 Future Improvements

V4 is already very close to production-ready databases. Potential enhancements:

- **Compaction**: Merge and compact SSTables to reduce read amplification
- **Bloom Filters**: Fast negative lookups per SSTable
- **Range Queries**: Leverage sorting for range scans
- **Concurrency**: Multi-threaded read/write operations

## 🌟 Production-Ready Alternative

For a production-ready implementation of these concepts, check out [**ZoneTree**](https://github.com/koculu/ZoneTree) - a persistent, high-performance, transactional, and ACID-compliant ordered key-value database for .NET that builds on these same principles.

## 📄 License

This project is for educational purposes, demonstrating different database storage strategies and optimization techniques.

---

**Want to understand the concepts in detail?** Read our [blog articles](BLOGARTIKEL.md) for a comprehensive explanation of how databases work and the evolution from V1 to V4.
