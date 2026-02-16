# Improving LevelDB Write Path Using Bloom-Filter-Based Duplicate Detection

<p align="center">
  <img src="https://img.shields.io/badge/Language-C++-00599C?style=for-the-badge&logo=cplusplus&logoColor=white"/>
  <img src="https://img.shields.io/badge/Database-LevelDB-green?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Architecture-LSM--Tree-blue?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Throughput-+67%25-success?style=for-the-badge"/>
</p>

---

## 📋 Overview

This project redesigns LevelDB's write pipeline to support **duplicate detection using a Bloom filter** integrated directly into the write path. By eliminating redundant `Get()` operations before each `Put()`, we achieve significant throughput improvements without modifying LevelDB's external semantics.

**Course:** CS543 - Database Systems, University of Southern California

---

## 🎯 Problem Statement

LevelDB is an LSM-tree-based key-value store optimized for high write throughput. However, it lacks a built-in mechanism for detecting duplicate keys during writes.

**The Common Pattern:**
```cpp
// Applications often do this to avoid overwriting:
if (!db->Get(key).ok()) {
    db->Put(key, value);
}
```

**The Problem:**
- Every `Get()` traverses memtable → immutable memtables → Level-0 SSTables → lower levels
- Even for non-existent keys, the lookup may touch multiple structures
- This **read-before-write pattern** adds significant latency and reduces throughput

---

## 💡 Solution

We integrated a **Bloom filter** directly into LevelDB's write path:

- **If Bloom filter says "definitely not present"** → Skip the `Get()`, proceed with `Put()`
- **If Bloom filter says "maybe present"** → Perform normal `Get()` to confirm

Since Bloom filters have **zero false negatives**, correctness is fully preserved.

---

## 🏗️ Architecture

```
                    Put(key, value)
                          │
                          ▼
                ┌─────────────────┐
                │  Check Bloom    │
                │  Filter         │
                └────────┬────────┘
                         │
                ┌────────┴────────┐
                │                 │
            "Not Present"    "Maybe Present"
                │                 │
                ▼                 ▼
          Skip Get()         Query DB
                │                 │
                ▼                 ▼
          Add to Filter     Add to Filter
                │                 │
                └────────┬────────┘
                         │
                         ▼
                ┌─────────────────┐
                │   Write Batch   │
                │   Memtable+WAL  │
                └─────────────────┘
```

---

## ⚙️ Implementation

### Duplicate Detection Modes

Three configurable modes exposed via LevelDB's `Options` interface:

```cpp
// Mode 1: No duplicate checking (baseline)
options.duplicate_check = kNoDuplicateCheck;

// Mode 2: Always Get before Put (naive approach)
options.duplicate_check = kAlwaysGetDuplicateCheck;

// Mode 3: Bloom filter optimization (our solution)
options.duplicate_check = kBloomFilterDuplicateCheck;
```

### Bloom Filter Configuration

```cpp
// Filter parameters
bits_per_key = 10;
num_hash_functions = 7;  // k = 7
```

### Key Files Modified

| File | Changes |
|------|---------|
| `include/leveldb/options.h` | Added `DuplicateCheckMode` enum |
| `db/db_impl.cc` | Integrated Bloom filter into `Write()` |
| `util/bloom_filter.cc` | Implemented write-path Bloom filter |
| `db/db_bench.cc` | Added `writerandomdup` benchmark |

---

## 📊 Results

### Throughput Comparison (ops/sec)

| Duplicate Ratio | NoCheck | Bloom Filter | Always-Get |
|-----------------|---------|--------------|------------|
| 0% | 327,769 | **478,696** | 286,942 |
| 10% | 490,148 | **385,652** | 288,963 |
| 25% | 502,084 | **369,118** | 303,154 |
| 50% | 497,446 | **349,487** | 323,334 |
| 75% | 519,878 | **362,346** | 343,344 |
| 90% | 506,819 | **385,616** | 377,501 |

### Bloom Filter vs Always-Get (Normalized)

| Duplicate Ratio | Speedup |
|-----------------|---------|
| 0% | **1.67x** |
| 10% | 1.33x |
| 25% | 1.22x |
| 50% | 1.08x |
| 75% | 1.06x |
| 90% | 1.02x |

### Key Findings

- ✅ **67% throughput improvement** at 0% duplication
- ✅ **Consistent gains** across all duplicate ratios
- ✅ **No regression** even at 90% duplication
- ✅ **Zero false negatives** — correctness preserved

---

## 🧪 Benchmarking

### Workloads

```bash
# Custom benchmark with configurable duplicate ratio
./db_bench --benchmarks=writerandomdup --dup_ratio=0.25 --num=1000000

# Fixed 50% duplication workload
./db_bench --benchmarks=random50dup --num=1000000
```

### Metrics Collected

- **Write latency** (µs/op)
- **Write throughput** (ops/sec)
- **Memory overhead** (Bloom filter size)

---

## 🚀 Getting Started

### Prerequisites

- C++ compiler with C++11 support
- CMake 3.10+
- Git
- ~500MB disk space for benchmarks

### Build

```bash
# Step 1: Clone original LevelDB
git clone https://github.com/google/leveldb.git
cd leveldb

# Step 2: Apply the patch
git apply /path/to/leveldb-staged-changes.patch

# Step 3: Build
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
```

### Run Benchmarks

```bash
# Test with configurable duplicate ratio (0-100)
./db_bench --benchmarks=writerandomdup --dup_percent=25 --num=1000000

# Fixed 50% duplicate workload
./db_bench --benchmarks=random50dup --num=1000000
```

> **Note:** To switch between duplicate-check modes, modify the `options.duplicate_check_mode` in `benchmarks/db_bench.cc` and rebuild.

---

## 📈 When to Use

| Scenario | Recommendation |
|----------|----------------|
| Low duplicate ratio (0-30%) | ✅ **Use Bloom filter** — maximum benefit |
| Medium duplicate ratio (30-70%) | ✅ Use Bloom filter — still beneficial |
| High duplicate ratio (70%+) | ⚠️ Bloom filter works but gains are minimal |
| No duplicate checking needed | Use `kNoDuplicateCheck` for baseline speed |

---

## 🔮 Future Work

- [ ] Mixed read/write workload evaluation
- [ ] Dynamic Bloom filter sizing based on workload
- [ ] Multi-threaded ingestion testing
- [ ] Periodic filter rebuilding for long-running deployments

---

## 📚 References

- [LevelDB Official Repository](https://github.com/google/leveldb)
- [Bloom Filter — Wikipedia](https://en.wikipedia.org/wiki/Bloom_filter)
- [LSM-Tree Paper](https://www.cs.umb.edu/~poneil/lsmtree.pdf)

---

## 📄 License

This project is for educational purposes as part of USC CS543.

---

<p align="center">
  <b>⭐ Star this repo if you found it helpful!</b>
</p>
