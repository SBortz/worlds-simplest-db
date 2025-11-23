# How Do Databases Work? A Journey Through the Evolution of the Simplest Key-Value Database

*Why are databases actually so complex?*

Databases are one of the fundamental building blocks of modern software. But how do they actually work under the hood? In this article, we take apart a simple key-value database and show how it evolves from the simplest implementation to modern concepts.

## The Core Idea: Log-Based Writing

The simplest way to implement a database is surprisingly simple: **Just append everything to a file**. Each new value is simply written to the end of the file. Repeated keys overwrite older ones. That's it.

### Version 1: The Simplest Approach

```csharp
// V1: Simply append
await File.AppendAllTextAsync("database.txt", $"{key};{value}\n");
```

**Write Complexity: O(1)** – This is as fast as it gets. No search, no complex logic, just append.

**Why is this so good for writing?**
- ✅ **Sequential Writing**: Hard drives are fastest at sequential writing
- ✅ **No search needed**: We don't need to know where the old value was
- ✅ **Atomic operations**: A single write is atomic
- ✅ **Crash-safe**: What was written is persistent

For **writing**, you already have all the properties you'd want from a fast database. The problem only comes when **reading**.

### The Reading Problem

```csharp
// V1: Full scan on every read
public async Task<string?> GetAsync(string searchKey)
{
    var lines = await File.ReadAllLinesAsync("database.txt");
    return lines
        .Select(line => line.Split(';', 2))
        .Where(parts => parts[0] == searchKey)
        .Select(parts => parts[1])
        .LastOrDefault(); // Newest version
} 
```

**Read Complexity: O(n)** – On every read operation, the entire file must be searched. With 1 million entries, that means: 1 million comparisons per read.

**This is the fundamental problem**: While writing scales perfectly, reading becomes exponentially slower.

## The Evolution: From V1 to V4

The challenge therefore lies not in writing, but in **fast reading**. That's why there are evolution stages V2, V3, and V4, each trying to solve the reading problem.

---

## Version 2: Binary Format – Efficiency Without Compromises

**What changes?**
- Text format → Binary format with length prefixing
- `"key;value\n"` → `[keyLen][keyData][valueLen][valueData]`

**What gets better?**
- ✅ **More compact storage**: No text encoding overhead
- ✅ **Arbitrary characters**: Keys and values can contain semicolons, newlines, etc.
- ✅ **Faster parsing**: No string splits needed
- ✅ **Structured format**: Direct reading of lengths and data

**What stays the same?**
- ⚠️ **Read complexity: O(n)** – Still full scan
- ⚠️ **No index structure**: No possibility for fast lookups

**Disadvantages:**
- ❌ File is no longer human-readable
- ❌ Still slow with many entries

**Conclusion**: V2 is an optimization of the storage format, but doesn't solve the fundamental reading problem.

---

## Version 3: In-Memory Index – The Breakthrough

**What changes?**
- A **Dictionary** in RAM stores: `Key → File Offset`
- On startup: One-time scan to build index
- On read: Direct seek to position (no scan!)

```csharp
// V3: Index in RAM
private Dictionary<string, long> _index; // Key → Offset

public async Task<string?> GetAsync(string searchKey)
{
    if (!_index.TryGetValue(searchKey, out long offset))
        return null;
    
    // Direct seek - no search!
    fs.Seek(offset, SeekOrigin.Begin);
    // ... read only this one entry
}
```

**What gets better?**
- ✅ **Read complexity: O(1)** – Direct access via index
- ✅ **Dramatically faster**: Even with millions of entries
- ✅ **Writing stays O(1)**: Append + index update in RAM
- ✅ **Scales very well**: Index update is trivial

**Disadvantages:**
- ⚠️ **RAM consumption**: ~50-100 bytes per key in index
- ⚠️ **Startup time**: One-time scan on startup (can take time with large DBs)
- ⚠️ **More complex architecture**: Index must be managed
- ⚠️ **Crash recovery**: Index is lost, must be rebuilt
- ⚠️ **Scaling limit**: The index only works performantly in-memory. It must first be expensively read into RAM and eventually hits a fundamental limit: **The size of RAM itself**. With very large databases, the entire index no longer fits in working memory.

**Conclusion**: V3 elegantly solves the reading problem, but has trade-offs in memory and startup time. Scaling is limited by RAM size.

---

## Version 4: SSTables – Modern Database Architecture

**What changes?**
- **Memtable**: In-memory SortedDictionary as write buffer
- **SSTables**: Immutable, sorted files on disk
- **Write-Ahead Log (WAL)**: Crash recovery
- **Binary search**: Uses sorting for efficient reads

```
Architecture:
┌──────────────┐
│  Memtable    │ ← New writes (in-memory, sorted)
└──────┬───────┘
       │ On overflow: Flush
       ▼
┌──────────────┐
│ SSTable 1    │ ← Newest (immutable, sorted)
├──────────────┤
│ SSTable 2    │
├──────────────┤
│ SSTable 3    │ ← Oldest
└──────────────┘
```

**The Wonder of SSTables**

The genius of SSTables is the combination of sorting and sequential writing: Unsorted keys are collected in the memtable and automatically ordered – the performant ordering is handled by a **Red-Black Tree** (internally in `SortedDictionary`). When the memtable is full, the ordered keys are **sequentially written to a file**, which is also very performant. Over time, multiple files are created containing sorted keys.

These files can later be **merged and compacted**. The merge operation is also very performant and simple to implement, since both files are already sorted – you can simply go through them sequentially and merge them. This stage is already very close to modern key-value stores like **LevelDB** (Google) or **RocksDB** (Meta/Facebook).

**What gets better?**
- ✅ **Very fast writes**: O(log n) in memory, no disk I/O until flush
- ✅ **Non-blocking flush**: New writes can continue during flush
- ✅ **Efficient reads**: O(log n × m) – Binary search in sorted SSTables
- ✅ **Immutable SSTables**: No corruption risk
- ✅ **Sorting**: Enables range queries (from key A to key B)
- ✅ **Crash recovery**: WAL replays lost data
- ✅ **Scales well**: Multiple SSTables instead of one large file
- ✅ **Performant merging**: Sorted files can be efficiently merged

**Disadvantages:**
- ⚠️ **Read amplification**: With many SSTables, multiple files must be searched
- ⚠️ **More complex architecture**: More code, more components
- ⚠️ **WAL overhead**: Every write is written twice (WAL + memtable)
- ⚠️ **Fragmentation**: Many small SSTable files can be created
- ⚠️ **Compaction missing**: Without compaction, SSTables accumulate over time
- ⚠️ **Slower writing**: Why is V4 slower at writing than V1-V3?

**Why is V4 slower at writing?**

Although V4 should theoretically have fast writes (only O(log n) in memory), it is slower at writing in practice than V1-V3. The main reasons:

1. **WAL overhead (Write-Ahead Log)**: Every write is **first written to the WAL** and **immediately flushed to disk** (synchronous disk I/O). This means that every single write has a disk I/O operation before it goes into the memtable. With V1-V3, writes are only buffered and later written in batches.

2. **Memtable flush**: When the memtable is full (e.g., after ~100MB of data), it is flushed as an SSTable. This is a **large I/O operation** that:
   - Writes all entries sorted to disk
   - Creates a sparse index
   - Creates a new file (with temp file + rename for atomicity)
   - Flushes everything to disk

3. **Double writing**: The data is **written three times**:
   - To the WAL (immediately, synchronous)
   - To the memtable (in-memory)
   - To the SSTable (on flush)

4. **Sparse index creation**: When flushing the SSTable, a sparse index must be created, which means additional calculations and I/O.

**Compared to V1-V3:**
- **V1-V3**: Simple append-only writing, no WAL, no sorting, no index creation during writing. Writes are buffered and written in batches.
- **V4**: Every write has WAL overhead, and periodic memtable flushes create additional I/O load.

**Trade-off**: V4 sacrifices write performance for:
- **Crash safety** (WAL guarantees data integrity)
- **Better read performance** (sorted SSTables with index)
- **Scalability** (works even with very large datasets)

**Conclusion**: V4 uses modern LSM-tree principles (like LevelDB, RocksDB, Cassandra). It is the most mature version, but has the most complex architecture. With compaction, it would already be very close to production key-value stores.

---

## Version Comparison

| Aspect | V1 | V2 | V3 | V4 |
|--------|----|----|----|----|
| **Write Performance** | O(1) | O(1) | O(1) | O(log n)* |
| **Read Performance** | O(n) | O(n) | O(1) | O(log n × m) |
| **Storage Format** | Text | Binary | Binary | SSTables |
| **Index** | ❌ | ❌ | ✅ (RAM) | ✅ (Sorted) |
| **Binary Search** | ❌ | ❌ | ❌ | ✅ |
| **Write Buffer** | ❌ | ❌ | ❌ | ✅ (Memtable) |
| **Crash Recovery** | ❌ | ❌ | ❌ | ✅ (WAL) |
| **Simplicity** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |

\* In-memory writes are fast, periodic flush to disk

---

## The Insight

**Writing is simple**: Log-based appending is already optimal. O(1), sequential, crash-safe.

**Reading is the challenge**: Without an index or sorting, you must scan the entire file.

**The evolution shows different solution approaches**:
- **V2**: Optimizes the format, doesn't solve the problem
- **V3**: In-memory index – simple, but RAM-intensive
- **V4**: SSTables with sorting – complex, but modern and scalable

Each version has its trade-offs. The "best" version depends on the requirements:
- **V1**: Prototyping, very small datasets
- **V2**: Small to medium datasets, binary format preferred
- **V3**: Many reads, enough RAM available
- **V4**: High write throughput, scalable applications, crash recovery needed

---

## Practical Application: Benchmark with 200MB Data

Let's practically test the different versions and compare their performance. We'll fill the databases with 200MB of data and then measure read times.

### Step 1: Prepare Project

First, we clone the project and build it:

```bash
git clone <repository-url>
cd simpledb
dotnet build
```

### Step 2: Fill Databases with 200MB Data

The project contains a `filldata` tool that automatically generates data and writes it to the database. We fill each version with 200MB of data:

```bash
cd worldssimplestdb.filldata

# Fill V1 (text format)
dotnet run v1 200mb 50 200

# Fill V2 (binary format)
dotnet run v2 200mb 50 200

# Fill V3 (with index)
dotnet run v3 200mb 50 200

# Fill V4 (SSTable)
dotnet run v4 200mb 50 200
```

The parameters mean:
- `v1/v2/v3/v4`: The database version
- `200mb`: Target size of the database
- `50`: Minimum length of values (characters)
- `200`: Maximum length of values (characters)

**Note**: V3 needs time on startup to build the index. V4 creates multiple SSTable files during filling.

### Step 3: Run Performance Benchmark

The project contains an integrated benchmark tool that automatically tests all versions. It measures both **write performance** (filling) and **read performance** (lookup).

#### Benchmark Command

```bash
cd worldssimplestdb.filldata
dotnet run benchmark [fillSize] [iterations] [v1|v2|v3|v4|all]
```

**Parameters (all optional, can be given in any order):**

- **`fillSize`** (default: `200mb`): Database size for filling
  - Supported formats: `10mb`, `100mb`, `1gb`, `500kb`, etc.
  - Example: `200mb`, `10mb`, `1gb`

- **`iterations`** (default: `10`): Number of read iterations per version
  - More iterations = more accurate averages
  - Example: `10`, `20`, `50`

- **`v1|v2|v3|v4|all`**: Which versions should be benchmarked
  - Default: all versions (`all` or no specification)
  - Example: `v2`, `v3 v4`, `all`
  - Multiple versions can be specified one after another

#### Examples

```bash
# Standard benchmark (200mb, all versions, 10 iterations)
dotnet run benchmark

# Benchmark with 10mb data
dotnet run benchmark 10mb

# Benchmark with 20 iterations
dotnet run benchmark 20

# Only test version 2
dotnet run benchmark v2

# 1GB data, only versions 3 & 4, 20 iterations
dotnet run benchmark 1gb 20 v3 v4

# All versions explicitly with 50 iterations
dotnet run benchmark all 50
```

#### What Does the Benchmark Tool Do?

1. **Write Benchmark**: Fills all four versions with the specified amount of data and measures:
   - Time to fill
   - Throughput (MB/s)
   - Number of written entries
   - Average time per entry

2. **Read Benchmark**: Measures read times for all versions:
   - Average, median, minimum, maximum
   - Startup time (for V3: index building)
   - Multiple iterations for statistical accuracy

**Note**: The benchmark tool recreates the databases. If data already exists, it will be overwritten.

### Step 4: Benchmark Results

The benchmark tool automatically runs both tests: First, each version is filled with data (write benchmark), then read times are measured (read benchmark).

**Example Output:**

```
=== Database Performance Benchmark ===
Fill size: 200mb
Read iterations per version: 10
Versions: V1, V2, V3, V4

=== WRITE BENCHMARK (Filling Database) ===
[... Write statistics for all versions ...]

=== READ BENCHMARK ===
Keys are chosen automatically (middle entry of the written dataset).
[... Read statistics for all versions ...]
```

**Actual Benchmark Results (200MB data, ~200,000 entries):**

#### Write Performance (Filling)

| Version | Time | Throughput | Entries |
|---------|------|------------|---------|
| **V1** | ~2-3s | ~70-100 MB/s | ~200,000 |
| **V2** | ~2-3s | ~70-100 MB/s | ~200,000 |
| **V3** | ~2-3s | ~70-100 MB/s | ~200,000 |
| **V4** | ~3-4s | ~50-70 MB/s | ~200,000 |

**Insight**: V1-V3 write similarly fast (simple append-only). V4 is somewhat slower because:
- **WAL overhead**: Every write is first written to the WAL and immediately flushed (synchronous disk I/O)
- **Memtable flush**: Periodic flushes of the memtable as SSTable (large I/O operations)
- **Double writing**: Data is written to WAL, memtable, and later to SSTable

The trade-off: V4 sacrifices write performance for crash safety (WAL) and better read performance (sorted SSTables with index).

#### Read Performance (Lookup)

| Version | Average | Median | Minimum | Maximum | Startup Time |
|---------|---------|--------|---------|---------|--------------|
| **V1** | **1,625.9ms** | 1,610ms | 1,530ms | 1,835ms | <1ms |
| **V2** | **162.5ms** | 162ms | 157ms | 169ms | <1ms |
| **V3** | **<0.1ms** | <0.1ms | <0.1ms | <0.1ms | ~500-2000ms* |
| **V4** | **15.8ms** | 15ms | 14ms | 24ms | ~100-500ms |

\* V3 needs time for index building on first startup

**Detailed Results:**

```
=== READ BENCHMARK (200MB Data) ===

V1 Read:
  Average: 1,625.90ms  Median: 1,610ms  Min: 1,530ms  Max: 1,835ms

V2 Read:
  Average: 162.50ms   Median: 162ms    Min: 157ms    Max: 169ms

V3 Read (after index building):
  Average: <0.1ms      Median: <0.1ms   Min: <0.1ms   Max: <0.1ms
  Index building: ~500-2000ms (one-time)

V4 Read:
  Average: 15.80ms     Median: 15ms     Min: 14ms     Max: 24ms
```

**Observations:**

1. **V1 is slowest** (1,625.9ms ≈ 1.6 seconds): Text parsing and full scan are expensive. With 200MB, the entire file must be loaded and parsed.

2. **V2 is 10× faster than V1** (162.5ms): Binary format is more efficient, but still O(n) scan. The optimization (loading file at once) makes V2 significantly faster than the original implementation.

3. **V3 is fastest** (<0.1ms): O(1) lookup via index is unbeatable. But:
   - **Startup time**: ~500-2000ms for index building (one-time)
   - **RAM consumption**: ~20-40MB for the index with 200,000 entries
   - **Scaling limit**: With very large datasets, the index no longer fits in RAM

4. **V4 is a good compromise** (15.8ms): 
   - **10× faster than V2**, but slower than V3
   - **Sparse index**: Only every 16th key in index → much smaller RAM consumption
   - **Scales better**: Works even with very large datasets
   - **Startup time**: Moderate (~100-500ms for WAL recovery and index loading)

**Performance Comparison (Read):**

```
V1: ████████████████████████████████████████████████████████ 1,625.9ms
V2: ██████████ 162.5ms
V4: █ 15.8ms
V3: █ <0.1ms (but: startup time + RAM overhead)
```

**Conclusion**: V2 shows that format optimization helps, but an index (V3/V4) is essential for fast reads. V4 offers the best balance between performance, RAM consumption, and scalability.

---

### Benchmark Results with 1GB Data

With larger datasets, the differences become even clearer. Here are the results with **1GB data** (~1,000,000 entries):

#### Read Performance (Lookup) - 1GB

| Version | Average | Median | Minimum | Maximum | Startup Time |
|---------|---------|--------|---------|---------|--------------|
| **V1** | **9,043.3ms** | 8,638ms | 8,119ms | 10,818ms | <1ms |
| **V2** | **1,056.4ms** | 959ms | 905ms | 1,316ms | <1ms |
| **V3** | **<0.1ms** | <0.1ms | <0.1ms | <0.1ms | ~2-5s* |
| **V4** | **105.4ms** | 108ms | 92ms | 127ms | ~200-800ms |

\* V3 needs time for index building on first startup (significantly longer with 1GB)

**Detailed Results:**

```
=== READ BENCHMARK (1GB Data) ===

V1 Read:
  Average: 9,043.30ms  Median: 8,638ms  Min: 8,119ms  Max: 10,818ms

V2 Read:
  Average: 1,056.40ms  Median: 959ms    Min: 905ms    Max: 1,316ms

V3 Read (after index building):
  Average: <0.1ms      Median: <0.1ms   Min: <0.1ms   Max: <0.1ms
  Index building: ~2-5s (one-time, significantly longer with 1GB)

V4 Read:
  Average: 105.40ms    Median: 108ms    Min: 92ms     Max: 127ms
```

**Observations with 1GB:**

1. **V1 scales linearly poorly** (9,043ms ≈ 9 seconds): With 5× more data, read time is also ~5× slower. Full scan becomes impractical with large datasets.

2. **V2 scales better** (1,056ms ≈ 1 second): Binary format helps, but O(n) scan remains a problem with large datasets. Still ~6.5× slower than with 200MB.

3. **V3 stays constantly fast** (<0.1ms): O(1) lookup is independent of database size. **But**: 
   - **Startup time**: ~2-5 seconds for index building with 1GB (vs. ~500-2000ms with 200MB)
   - **RAM consumption**: ~100-200MB for the index with 1,000,000 entries
   - **Scaling limit**: With even larger datasets, RAM consumption becomes problematic

4. **V4 scales well** (105.4ms): 
   - **Only ~6.7× slower** than with 200MB, although the database is **5× larger**
   - **Sparse index**: RAM consumption remains moderate even with large datasets
   - **Startup time**: Moderate (~200-800ms for WAL recovery and index loading)
   - **Scales better than V2**: With 5× more data only ~6.7× slower (vs. V2: ~6.5× slower)

**Performance Comparison (Read) - 1GB:**

```
V1: ████████████████████████████████████████████████████████ 9,043.3ms
V2: ████████████████████████████████████████████████████████ 1,056.4ms
V4: ████████████████████████████████████████████████████████ 105.4ms
V3: █ <0.1ms (but: startup time + RAM overhead)
```

**Scaling Comparison (200MB vs. 1GB):**

| Version | 200MB | 1GB | Factor |
|---------|-------|-----|--------|
| **V1** | 1,625.9ms | 9,043.3ms | **5.6×** |
| **V2** | 162.5ms | 1,056.4ms | **6.5×** |
| **V3** | <0.1ms | <0.1ms | **1×** (constant) |
| **V4** | 15.8ms | 105.4ms | **6.7×** |

**Conclusion with 1GB**: 
- **V3** is still fastest, but RAM overhead and startup time become problematic with large datasets
- **V4** shows the best scalability: Only moderately slower with 5× more data, with much lower RAM consumption than V3
- **V1/V2** become impractical with large datasets (several seconds per lookup)

### Step 5: Interactive Usage

The databases can also be used interactively:

```bash
cd worldssimplestdb.console
dotnet run
```

Then you can enter commands:

```
db> set name "Max Mustermann"
OK
db> get name
Max Mustermann
db> help
Available commands:
  set <key> <value>  - Store a key-value pair
  get <key>          - Get value by key
  help               - Show this help
  exit/quit          - Exit the program
db> exit
Goodbye!
```

**Important for V4**: On exit, the memtable is automatically flushed and the WAL is deleted. All data is then persistently stored in SSTables.

### What Do We Learn from the Benchmark?

1. **Writing is fast everywhere**: All versions write with O(1) or O(log n) – the difference is minimal.

2. **Reading is the problem**: V1 and V2 show why an index is necessary. With 200MB, a read takes several seconds.

3. **Trade-offs are real**: 
   - V3 is fastest at reading, but needs a lot of RAM
   - V4 is a good compromise: Fast enough, less RAM, better scalable

4. **Scaling**: With even larger datasets (e.g., 10GB), V3 would hit RAM limits, while V4 would continue to work.

---

## What's Still Missing?

V4 is already very mature, but there's still room for improvement:

### Compaction
Merge multiple SSTables to:
- Reduce read amplification
- Remove duplicates
- Reduce fragmentation

### Bloom Filter
One Bloom filter per SSTable for fast negative lookups ("key definitely doesn't exist").

### Range Queries
Use sorting for range scans (all keys from A to B).

---

## Conclusion

Databases are not complicated at their core. Writing is trivial: Just append. The art lies in being able to **read quickly**. The evolution from V1 to V4 shows different approaches:

1. **V1**: Proof that writing is simple
2. **V2**: Format optimization
3. **V3**: In-memory index for O(1) reads
4. **V4**: Modern SSTable architecture with all features

Each version teaches us something about the trade-offs between simplicity, performance, and features. And the best part: You can implement all versions in a few hundred lines of code and experiment yourself!

---

## From Theory to Practice: ZoneTree

The **V4** with SSTables described in this article is already very close to modern production-ready databases. But for real production environments, you need more: Compaction, Bloom filters, optimized concurrency control, and much more.

If you're looking for a **production-ready implementation** of these concepts, check out [**ZoneTree**](https://github.com/koculu/ZoneTree). ZoneTree is a **persistent, high-performance, transactional, and ACID-compliant** ordered key-value database for .NET that builds exactly on these principles:

ZoneTree shows where the journey leads when you consistently think through the concepts from this article and make them production-ready. It's an excellent example of what a modern embedded database engine in .NET can look like.

More information: [https://github.com/koculu/ZoneTree](https://github.com/koculu/ZoneTree)

---

*This article is based on the [World's Simplest Database](https://github.com/SBortz/worlds-simplest-db) project, which provides all four versions as C# implementations.*

