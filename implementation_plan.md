# TinyLSM: 4-Week Implementation Plan

A day-by-day guide for building an LSM-tree storage engine in Rust, focused on the architecture that matters.

---

## Philosophy

**Build what teaches you. Use libraries for the rest.**

You will implement from scratch:
- Lock-free skip list
- Bloom filters  
- LRU cache
- SSTable format
- WAL and crash recovery
- Compaction

You will use libraries for:
- CRC32 (`crc32fast`)
- LZ4 compression (`lz4_flex`)
- Hash functions (`xxhash-rust`)

This keeps focus on storage engine architecture — not reimplementing solved algorithms.

---

## Dependencies

```toml
[dependencies]
crc32fast = "1.3"
lz4_flex = "0.11"
xxhash-rust = { version = "0.8", features = ["xxh3"] }

[dev-dependencies]
criterion = "0.5"
tempfile = "3.10"
rand = "0.8"
```

---

## Week 1: Foundations — Memtable & Write-Ahead Log

### Learning Objectives
- Build lock-free skip list with atomics
- Understand memory ordering semantics
- Implement crash-safe write-ahead log

---

### Day 1: Project Setup & Skip List Theory

**Morning**
- [ ] Initialize Cargo workspace:
  ```
  tinylsm/
  ├── crates/
  │   ├── tinylsm-core/
  │   ├── tinylsm-bench/
  │   └── tinylsm-cli/
  └── Cargo.toml
  ```
- [ ] Configure release profile for performance
- [ ] Set up CI with clippy, fmt, test

**Afternoon**
- [ ] Study skip list structure and probabilistic balancing
- [ ] Study lock-free algorithms (Herlihy & Shavit)
- [ ] Learn memory ordering: `Relaxed`, `Acquire`, `Release`, `SeqCst`
- [ ] Understand epoch-based memory reclamation

**Evening**
- [ ] Design skip list node structure
- [ ] Plan memory layout for cache efficiency
- [ ] Document expected time complexities

---

### Day 2: Skip List — Core Implementation

**Morning**
- [ ] Implement node structure:
  ```rust
  struct Node<K, V> {
      key: K,
      value: V,
      height: usize,
      tower: [AtomicPtr<Node<K, V>>; MAX_HEIGHT],
  }
  ```
- [ ] Implement random height generation (geometric distribution)
- [ ] Implement `find_position` helper

**Afternoon**
- [ ] Implement `insert` with CAS operations
- [ ] Handle concurrent insert conflicts
- [ ] Implement `get` with proper acquire semantics

**Evening**
- [ ] Write single-threaded correctness tests
- [ ] Test sorted order invariant
- [ ] Verify basic operations work

---

### Day 3: Skip List — Iteration & Memory Safety

**Morning**
- [ ] Implement forward iterator
- [ ] Implement `range(start..end)` for scans
- [ ] Handle concurrent modification during iteration

**Afternoon**
- [ ] Implement epoch-based garbage collection:
  ```rust
  struct Epoch {
      global: AtomicU64,
      garbage: Vec<DeferredDrop>,
  }
  ```
- [ ] Implement `pin()` / `unpin()` guards
- [ ] Add memory usage tracking

**Evening**
- [ ] Benchmark vs `BTreeMap` (single-threaded)
- [ ] Benchmark concurrent performance
- [ ] Profile memory allocations

**Performance Targets:**
- Search: O(log n)
- Concurrent writes: >500K ops/sec (8 threads)

---

### Day 4: Memtable Implementation

**Morning**
- [ ] Design internal key format:
  ```rust
  struct InternalKey {
      user_key: Vec<u8>,
      sequence: u64,
      value_type: ValueType,  // Put or Delete
  }
  ```
- [ ] Design memtable state machine: Active → Frozen → Flushing

**Afternoon**
- [ ] Implement `Memtable` wrapper around skip list
- [ ] Implement `put(key, value, seq)`
- [ ] Implement `get(key)` returning latest version
- [ ] Implement `delete(key, seq)` as tombstone

**Evening**
- [ ] Implement `freeze()` for immutable transition
- [ ] Add approximate size tracking
- [ ] Implement `MemtableIterator` for flush

---

### Day 5: WAL Record Format & Serialization

**Morning**
- [ ] Implement varint encoding:
  ```rust
  pub fn encode_varint(value: u64, buf: &mut Vec<u8>)
  pub fn decode_varint(buf: &[u8]) -> Option<(u64, usize)>
  ```
- [ ] Implement zig-zag encoding for signed integers

**Afternoon**
- [ ] Design WAL record format:
  ```
  +--------+--------+----------+------+------+-------+-------+
  | CRC32  | Length | SeqNum   | Type | KLen | Key   | Value |
  | 4B     | 4B     | 8B       | 1B   | var  | var   | var   |
  +--------+--------+----------+------+------+-------+-------+
  ```
- [ ] Implement record serialization using `crc32fast`
- [ ] Implement record deserialization

**Evening**
- [ ] Test serialization round-trip
- [ ] Verify CRC catches corruption
- [ ] Document wire format

---

### Day 6: WAL Writer

**Morning**
- [ ] Study fsync semantics:
  - `write()` → kernel buffer
  - `fsync()` → disk
  - `fdatasync()` → data only
- [ ] Implement `WalWriter`:
  ```rust
  pub struct WalWriter {
      file: File,
      buffer: Vec<u8>,
      offset: u64,
  }
  ```

**Afternoon**
- [ ] Implement buffered append with CRC32
- [ ] Implement sync policies (every write, every N, manual)
- [ ] Implement batch write API for group commit

**Evening**
- [ ] Benchmark write throughput with different sync policies
- [ ] Measure latency distribution (p50, p99)
- [ ] Test partial write handling

**Target:** >100MB/s sequential write with batching

---

### Day 7: WAL Recovery & Integration

**Morning**
- [ ] Implement `WalReader` for sequential scanning
- [ ] Handle corrupted records gracefully
- [ ] Handle partial records at end of file

**Afternoon**
- [ ] Implement recovery logic:
  ```rust
  pub fn recover(path: &Path) -> Result<(Memtable, u64)>
  ```
- [ ] Integrate WAL with memtable writes
- [ ] Implement write batch API

**Evening**
- [ ] Crash testing: kill process, verify recovery
- [ ] Test with corrupted WAL
- [ ] Document Week 1 architecture

---

## Week 2: SSTable Format & Disk Storage

### Learning Objectives
- Design efficient block-based file format
- Implement zero-copy reads with mmap
- Build table cache

---

### Day 8: SSTable Architecture

**Morning**
- [ ] Study LevelDB/RocksDB SSTable format
- [ ] Design file layout:
  ```
  [Data Block 0]
  [Data Block 1]
  ...
  [Filter Block]
  [Index Block]
  [Footer]
  ```

**Afternoon**
- [ ] Design data block format:
  - Key delta encoding
  - Restart points every 16 keys
  - LZ4 compression via `lz4_flex`
  - CRC32 trailer
- [ ] Design index block format
- [ ] Design footer with magic number

**Evening**
- [ ] Document complete file format
- [ ] Calculate overhead for typical workloads
- [ ] Plan format versioning

---

### Day 9: Block Builder

**Morning**
- [ ] Implement `BlockBuilder`:
  ```rust
  pub struct BlockBuilder {
      buffer: Vec<u8>,
      restarts: Vec<u32>,
      last_key: Vec<u8>,
      counter: usize,
  }
  ```
- [ ] Implement key delta encoding

**Afternoon**
- [ ] Implement restart point logic
- [ ] Implement `finish()`:
  - Append restart array
  - Compress with LZ4
  - Append CRC32

**Evening**
- [ ] Test with various key/value sizes
- [ ] Benchmark block building
- [ ] Verify compression ratio

---

### Day 10: Block Reader

**Morning**
- [ ] Implement `BlockReader`:
  ```rust
  pub struct BlockReader<'a> {
      data: &'a [u8],
      restarts_offset: usize,
      num_restarts: usize,
  }
  ```
- [ ] Implement binary search via restart points
- [ ] Implement key decoding

**Afternoon**
- [ ] Implement `BlockIterator`:
  ```rust
  impl<'a> Iterator for BlockIterator<'a> {
      type Item = (&'a [u8], &'a [u8]);  // Zero-copy
  }
  ```
- [ ] Implement `seek(key)`
- [ ] Handle decompression

**Evening**
- [ ] Benchmark point lookup (<1μs target)
- [ ] Benchmark iteration (>5M entries/sec)
- [ ] Test edge cases

---

### Day 11: SSTable Writer

**Morning**
- [ ] Implement `SstWriter`:
  ```rust
  pub struct SstWriter {
      file: BufWriter<File>,
      data_block: BlockBuilder,
      index_block: BlockBuilder,
      offset: u64,
  }
  ```
- [ ] Implement `add(key, value)` with auto block flush

**Afternoon**
- [ ] Implement `finish()`:
  - Flush pending data block
  - Write filter block (placeholder for Week 4)
  - Write index block
  - Write footer
- [ ] Atomic file creation (temp file + rename)

**Evening**
- [ ] Test SSTable creation
- [ ] Verify format correctness
- [ ] Benchmark write throughput (>50MB/s)

---

### Day 12: SSTable Reader

**Morning**
- [ ] Implement `SstReader`:
  ```rust
  pub struct SstReader {
      mmap: Mmap,
      footer: Footer,
      index_block: BlockReader<'static>,
      file_id: u64,
  }
  ```
- [ ] Implement file opening: read footer, load index

**Afternoon**
- [ ] Implement point lookup:
  1. Binary search index block
  2. Load data block
  3. Binary search within block
- [ ] Implement `SstIterator` for range scans

**Evening**
- [ ] Benchmark read latency
- [ ] Test various file sizes
- [ ] Compare mmap vs pread

---

### Day 13: LRU Cache Implementation

**Morning**
- [ ] Implement LRU cache from scratch:
  ```rust
  pub struct LruCache<K, V> {
      map: HashMap<K, NonNull<Node<K, V>>>,
      head: *mut Node<K, V>,
      tail: *mut Node<K, V>,
      capacity: usize,
  }
  ```
- [ ] Implement doubly-linked list for LRU ordering

**Afternoon**
- [ ] Implement `get` with promotion to front
- [ ] Implement `put` with eviction
- [ ] Add size tracking

**Evening**
- [ ] Test eviction behavior
- [ ] Benchmark cache operations
- [ ] Verify capacity limits

---

### Day 14: Table Cache & Flush Pipeline

**Morning**
- [ ] Implement `TableCache`:
  ```rust
  pub struct TableCache {
      cache: Mutex<LruCache<u64, Arc<SstReader>>>,
      db_path: PathBuf,
  }
  ```
- [ ] Limit open file handles

**Afternoon**
- [ ] Implement flush pipeline:
  ```rust
  fn flush_memtable(&self, mem: &Memtable) -> Result<SstMeta>
  ```
- [ ] Generate unique file names
- [ ] Integrate with WAL cleanup

**Evening**
- [ ] Test concurrent reads during flush
- [ ] Benchmark flush throughput
- [ ] Document Week 2 architecture

---

## Week 3: Compaction & Crash Recovery

### Learning Objectives
- Implement efficient k-way merge
- Build robust manifest system
- Handle all crash scenarios

---

### Day 15: Merge Iterator

**Morning**
- [ ] Implement k-way merge with binary heap:
  ```rust
  pub struct MergeIterator<I> {
      heap: BinaryHeap<Reverse<HeapItem<I>>>,
  }
  ```
- [ ] Handle ordering by key, then sequence number

**Afternoon**
- [ ] Implement deduplication (newest wins)
- [ ] Handle tombstones correctly
- [ ] Optimize heap operations

**Evening**
- [ ] Test merge correctness
- [ ] Benchmark with many iterators
- [ ] Verify sorted output

---

### Day 16: Compaction Theory & File Picking

**Morning**
- [ ] Study amplification factors:
  - Write amp: disk writes / user writes
  - Read amp: disk reads / user reads
  - Space amp: disk space / data size
- [ ] Study leveled vs tiered compaction

**Afternoon**
- [ ] Implement level structure:
  ```rust
  pub struct LevelState {
      levels: Vec<Vec<Arc<SstMeta>>>,
      level_max_bytes: Vec<u64>,
  }
  ```
- [ ] Implement compaction picking for L0 → L1

**Evening**
- [ ] Implement overlapping file detection
- [ ] Test file selection logic
- [ ] Document compaction strategy

---

### Day 17: Compaction Execution

**Morning**
- [ ] Implement compaction job:
  ```rust
  pub struct CompactionJob {
      input_level: usize,
      output_level: usize,
      inputs: Vec<Arc<SstMeta>>,
  }
  ```
- [ ] Execute merge → new SSTable(s)

**Afternoon**
- [ ] Implement output splitting (~64MB per file)
- [ ] Implement tombstone dropping:
  - At bottom level
  - When no snapshots reference
- [ ] Handle sequence number filtering

**Evening**
- [ ] Test compaction correctness
- [ ] Verify data integrity
- [ ] Test reads during compaction

---

### Day 18: Manifest Implementation

**Morning**
- [ ] Design version edit format:
  ```rust
  pub enum VersionEdit {
      AddFile { level: u32, meta: SstMeta },
      RemoveFile { level: u32, file_id: u64 },
      SetLogNumber(u64),
      SetLastSequence(u64),
  }
  ```

**Afternoon**
- [ ] Implement manifest writer (append-only)
- [ ] Implement manifest reader
- [ ] Implement CURRENT file pointer

**Evening**
- [ ] Test serialization round-trip
- [ ] Test manifest recovery
- [ ] Handle corruption gracefully

---

### Day 19: Crash Recovery

**Morning**
- [ ] Implement full recovery:
  ```rust
  pub fn open(path: &Path) -> Result<Db> {
      // 1. Read CURRENT
      // 2. Replay manifest → levels
      // 3. Replay WAL → memtable
      // 4. Resume
  }
  ```

**Afternoon**
- [ ] Handle crash during compaction
- [ ] Handle crash during flush
- [ ] Implement consistency checks

**Evening**
- [ ] Crash testing at every point
- [ ] Verify zero data loss
- [ ] Document recovery process

---

### Day 20: Background Compaction

**Morning**
- [ ] Implement compaction thread:
  ```rust
  fn compaction_loop(db: Arc<DbInner>) {
      loop {
          if let Some(job) = db.pick_compaction() {
              db.run_compaction(job);
          } else {
              db.compact_signal.wait();
          }
      }
  }
  ```

**Afternoon**
- [ ] Implement write throttling:
  - Slow down when L0 count high
  - Stop when L0 critically high
- [ ] Implement compaction rate limiting

**Evening**
- [ ] Test under sustained load
- [ ] Measure write stalls
- [ ] Tune thresholds

---

### Day 21: Concurrent Access

**Morning**
- [ ] Implement version reference counting
- [ ] Safe file deletion when unreferenced
- [ ] Read path with version pinning

**Afternoon**
- [ ] Stress test: readers + writers + compaction
- [ ] Profile lock contention
- [ ] Fix any race conditions

**Evening**
- [ ] Long-running stability test
- [ ] Document concurrency model
- [ ] Week 3 review

---

## Week 4: Query Optimizations & Polish

### Learning Objectives
- Implement Bloom filters from scratch
- Build sharded block cache
- Production-ready testing

---

### Day 22: Bloom Filter Theory

**Morning**
- [ ] Study Bloom filter math:
  ```
  False positive rate ≈ (1 - e^(-kn/m))^k
  Optimal k = (m/n) * ln(2)
  ```
- [ ] Understand bits-per-key trade-off
- [ ] Study double hashing technique

**Afternoon**
- [ ] Design filter parameters:
  - 10 bits/key → ~1% false positive
  - 2 base hashes → k probes via double hashing

**Evening**
- [ ] Document parameter selection
- [ ] Plan integration points

---

### Day 23: Bloom Filter Implementation

**Morning**
- [ ] Implement `BloomFilter`:
  ```rust
  pub struct BloomFilter {
      bits: Vec<u64>,
      num_hashes: u32,
  }
  ```
- [ ] Use `xxhash` for base hashes
- [ ] Implement double hashing for k probes

**Afternoon**
- [ ] Implement `add(key: &[u8])`
- [ ] Implement `may_contain(key: &[u8]) -> bool`
- [ ] Implement serialization/deserialization

**Evening**
- [ ] Test false positive rate empirically
- [ ] Verify zero false negatives
- [ ] Benchmark probe time (<100ns)

---

### Day 24: Bloom Filter Integration

**Morning**
- [ ] Build filter during SSTable creation
- [ ] Store in filter block
- [ ] Load on SSTable open

**Afternoon**
- [ ] Integrate into read path:
  ```rust
  fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>> {
      if !self.filter.may_contain(key) {
          return Ok(None);
      }
      // Continue lookup...
  }
  ```

**Evening**
- [ ] Measure read improvement
- [ ] Test filter effectiveness
- [ ] Tune bits-per-key

---

### Day 25: Block Cache

**Morning**
- [ ] Implement sharded block cache:
  ```rust
  pub struct BlockCache {
      shards: [Mutex<LruCache<CacheKey, Arc<Block>>>; 16],
  }
  ```
- [ ] Implement cache key: (file_id, offset)

**Afternoon**
- [ ] Integrate into SSTable reader
- [ ] Add hit/miss statistics
- [ ] Implement size limiting

**Evening**
- [ ] Benchmark cached vs uncached
- [ ] Test under memory pressure
- [ ] Tune shard count

---

### Day 26: Read Path Optimization

**Morning**
- [ ] Profile complete read path
- [ ] Identify hot spots
- [ ] Optimize allocations

**Afternoon**
- [ ] Pin index blocks in cache
- [ ] Pin filter blocks in cache
- [ ] Implement prefetching (optional)

**Evening**
- [ ] Benchmark all read patterns
- [ ] Document performance characteristics

---

### Day 27: Comprehensive Testing

**Morning**
- [ ] Randomized correctness tests:
  ```rust
  fn fuzz_operations() {
      // Random put/get/delete
      // Compare against BTreeMap reference
  }
  ```

**Afternoon**
- [ ] Crash recovery tests
- [ ] Concurrent stress tests
- [ ] Edge case tests

**Evening**
- [ ] Fix discovered bugs
- [ ] Document test coverage

---

### Day 28: Benchmarking & Documentation

**Morning**
- [ ] Create benchmark suite with `criterion`:
  - Sequential write
  - Random write
  - Point read
  - Range scan
  - Mixed workload

**Afternoon**
- [ ] Profile with `perf` / flamegraph
- [ ] Document benchmark results
- [ ] Write architecture docs

**Evening**
- [ ] Final code review
- [ ] Complete README
- [ ] Tag v0.1.0

---

## What You're Building From Scratch

| Component | Why |
|-----------|-----|
| Skip list | Core data structure, concurrency learning |
| LRU cache | Classic algorithm, performance critical |
| Bloom filter | Probability meets systems |
| SSTable format | File format design |
| WAL | Crash safety fundamentals |
| Compaction | LSM core algorithm |
| Manifest | Version control for databases |

## What You're Using Libraries For

| Component | Library | Why |
|-----------|---------|-----|
| CRC32 | `crc32fast` | Solved problem, not educational |
| Compression | `lz4_flex` | Complex, not the focus |
| Hashing | `xxhash-rust` | Need good distribution, not crypto |

---

## Performance Targets

| Operation | Target |
|-----------|--------|
| Sequential write | 100K+ ops/sec |
| Random write | 50K+ ops/sec |
| Point read (cached) | <10μs |
| Point read (disk) | <100μs |
| Range scan | 5M+ entries/sec |
| Crash recovery | <1s per 100MB WAL |

---

## Success Criteria

By end of Week 4:

1. **Crash-safe**: Survives kill -9 at any point
2. **Correct**: Passes fuzz tests
3. **Fast**: Meets performance targets
4. **Clean**: Well-documented code
5. **Understood**: Can explain every component

---

## References

- [LevelDB Implementation Notes](https://github.com/google/leveldb/blob/main/doc/impl.md)
- [RocksDB Wiki](https://github.com/facebook/rocksdb/wiki)
- [Skip Lists Paper (Pugh, 1990)](https://15721.courses.cs.cmu.edu/spring2018/papers/08-oltpindexes1/pugh-skiplists-cacm1990.pdf)
- [Designing Data-Intensive Applications, Chapter 3](https://dataintensive.net/)

