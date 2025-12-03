# LSM-Tree Storage Engine in Rust: 4-Week Implementation Plan
## Build Everything From Scratch Edition

A comprehensive day-by-day guide for building a production-grade LSM-Tree storage engine in Rust with minimal dependencies. Maximum learning, maximum control.

---

## Philosophy

**Build it yourself. Understand every byte.**

This plan minimizes external dependencies. You will implement:
- Skip list (lock-free)
- CRC32 checksum
- Hash functions (for Bloom filters)
- LZ4 compression (simplified)
- Memory reclamation (optional: replace crossbeam-epoch)

**Only tooling libraries remain** — things that help you test and measure, not things that do the work for you.

---

## Dependencies

```toml
[dependencies]
# Zero runtime dependencies for core logic!
# Everything below is optional tooling.

[dev-dependencies]
criterion = "0.5"      # Benchmarking
loom = "0.7"           # Concurrency testing
tempfile = "3.10"      # Test utilities
rand = "0.8"           # Test utilities

[optional]
crossbeam-epoch = "0.9"  # Only if you skip DIY memory reclamation
```

**That's it.** The core engine uses only `std`.

---

## What You'll Build From Scratch

| Component | Why Build It |
|-----------|--------------|
| Skip List | Core data structure — must understand deeply |
| CRC32 | Simple algorithm, critical for data integrity |
| Hash Function | Foundation for Bloom filters |
| LZ4 Compression | Teaches you block compression fundamentals |
| Epoch-Based Reclamation | Advanced memory management (optional) |
| LRU Cache | Classic algorithm, cache is performance-critical |
| Bloom Filter | Probability meets systems programming |
| Varint Encoding | Space-efficient serialization |

---

## Target Performance

| Operation | Target | Notes |
|-----------|--------|-------|
| Sequential write | 100K+ ops/sec | Batched WAL |
| Random write | 50K+ ops/sec | Compaction dependent |
| Point read (cached) | <10μs | Bloom + cache hit |
| Point read (disk) | <100μs | NVMe SSD |
| Range scan | 5M+ entries/sec | Sequential I/O |
| Recovery | <1s per 100MB WAL | |

---

## Week 1: Foundations — From-Scratch Primitives

### Learning Objectives
- Implement fundamental algorithms (CRC32, hashing)
- Build lock-free skip list with atomic operations
- Understand memory ordering deeply
- Create crash-safe WAL

---

### Day 1: Project Setup & CRC32 Implementation

**Morning Setup**
- [ ] Initialize Cargo workspace:
  ```
  lsm-engine/
  ├── crates/
  │   ├── lsm-core/       # Main library
  │   ├── lsm-bench/      # Benchmarks  
  │   └── lsm-cli/        # CLI tool
  └── Cargo.toml
  ```
- [ ] Configure release profile:
  ```toml
  [profile.release]
  lto = "thin"
  codegen-units = 1
  panic = "abort"
  ```
- [ ] Set up CI: `cargo clippy`, `cargo fmt`, `cargo test`

**Afternoon — CRC32 Implementation**
- [ ] Study CRC32 algorithm (polynomial division)
- [ ] Implement lookup table generation:
  ```rust
  const fn generate_crc32_table() -> [u32; 256] {
      // IEEE polynomial: 0xEDB88320
  }
  static CRC32_TABLE: [u32; 256] = generate_crc32_table();
  ```
- [ ] Implement `crc32(data: &[u8]) -> u32`
- [ ] Optimize with slice-by-8 technique (optional)

**Evening Tasks**
- [ ] Test against known CRC32 test vectors
- [ ] Benchmark throughput (target: >1GB/s)
- [ ] Document the algorithm

**CRC32 Details:**
- Polynomial: 0xEDB88320 (IEEE standard, same as zlib)
- Initial value: 0xFFFFFFFF
- Final XOR: 0xFFFFFFFF

---

### Day 2: Hash Function Implementation

**Morning Study**
- [ ] Study non-cryptographic hash functions:
  - FNV-1a (simple, decent distribution)
  - xxHash (fast, good distribution)
  - MurmurHash3 (widely used)
- [ ] Understand avalanche effect
- [ ] Learn about hash quality metrics

**Afternoon Implementation**
- [ ] Implement FNV-1a (warmup):
  ```rust
  pub fn fnv1a_64(data: &[u8]) -> u64 {
      const FNV_OFFSET: u64 = 0xcbf29ce484222325;
      const FNV_PRIME: u64 = 0x100000001b3;
      // ...
  }
  ```
- [ ] Implement simplified xxHash64:
  ```rust
  pub fn xxhash64(data: &[u8], seed: u64) -> u64
  ```
- [ ] Focus on correctness first, then optimize

**Evening Tasks**
- [ ] Test against official xxHash test vectors
- [ ] Benchmark throughput (target: >5GB/s for xxHash)
- [ ] Test distribution quality with chi-squared test

**Hash Function Requirements:**
- Deterministic (same input → same output always)
- Good distribution (for Bloom filter effectiveness)
- Fast (will be called frequently)

---

### Day 3: Varint & Serialization Utilities

**Morning Tasks**
- [ ] Implement varint encoding (protobuf-style):
  ```rust
  pub fn encode_varint(value: u64, buf: &mut Vec<u8>)
  pub fn decode_varint(buf: &[u8]) -> Option<(u64, usize)>
  ```
- [ ] Implement zig-zag encoding for signed integers:
  ```rust
  pub fn zigzag_encode(n: i64) -> u64
  pub fn zigzag_decode(n: u64) -> i64
  ```

**Afternoon Tasks**
- [ ] Create buffer utilities (replacing `bytes` crate):
  ```rust
  pub trait BufRead {
      fn get_u8(&mut self) -> Option<u8>;
      fn get_u32_le(&mut self) -> Option<u32>;
      fn get_u64_le(&mut self) -> Option<u64>;
      fn get_slice(&mut self, len: usize) -> Option<&[u8]>;
  }
  
  pub trait BufWrite {
      fn put_u8(&mut self, v: u8);
      fn put_u32_le(&mut self, v: u32);
      fn put_u64_le(&mut self, v: u64);
      fn put_slice(&mut self, data: &[u8]);
  }
  ```
- [ ] Implement for `&[u8]` and `Vec<u8>`

**Evening Tasks**
- [ ] Test varint round-trip for edge cases
- [ ] Benchmark encoding/decoding speed
- [ ] Document wire format

**Varint Encoding:**
- 7 bits of data per byte
- High bit = continuation flag
- Little-endian byte order
- Saves space for small numbers (common case)

---

### Day 4: Skip List Theory & Node Design

**Morning Study**
- [ ] Read Pugh's original skip list paper
- [ ] Understand probabilistic balancing
- [ ] Study lock-free skip list algorithms (Herlihy & Shavit)
- [ ] Learn memory ordering:
  - `Relaxed` — no synchronization
  - `Acquire` — see writes before release
  - `Release` — make writes visible
  - `SeqCst` — total ordering (expensive)

**Afternoon Design**
- [ ] Design node structure:
  ```rust
  struct Node<K, V> {
      key: K,
      value: V,
      height: usize,
      // Tower of atomic pointers
      tower: [AtomicPtr<Node<K, V>>; MAX_HEIGHT],
  }
  ```
- [ ] Plan memory layout for cache efficiency
- [ ] Design allocation strategy

**Evening Tasks**
- [ ] Document expected time complexities
- [ ] Sketch insertion/deletion algorithms
- [ ] Plan test strategy

**Skip List Properties:**
- Expected height: O(log n)
- Search: O(log n) expected
- Insert: O(log n) expected
- Memory: O(n) with ~2n pointers average

---

### Day 5: Skip List Implementation — Core Operations

**Morning Tasks**
- [ ] Implement node allocation:
  ```rust
  impl<K, V> Node<K, V> {
      fn new(key: K, value: V, height: usize) -> Box<Self>
  }
  ```
- [ ] Implement random height generation:
  ```rust
  fn random_height() -> usize {
      // Geometric distribution, p = 0.5
      let mut height = 1;
      while height < MAX_HEIGHT && random_bit() {
          height += 1;
      }
      height
  }
  ```
- [ ] Implement `find_position` helper (locates splice points)

**Afternoon Tasks**
- [ ] Implement `insert` with CAS operations:
  ```rust
  pub fn insert(&self, key: K, value: V) -> bool
  ```
- [ ] Handle concurrent insert conflicts
- [ ] Implement `get`:
  ```rust
  pub fn get(&self, key: &K) -> Option<&V>
  ```

**Evening Tasks**
- [ ] Write single-threaded correctness tests
- [ ] Test with random insertions
- [ ] Verify sorted order invariant

---

### Day 6: Skip List — Iteration & Memory Safety

**Morning Tasks**
- [ ] Implement forward iterator:
  ```rust
  pub struct SkipListIter<'a, K, V> {
      current: *const Node<K, V>,
      _marker: PhantomData<&'a SkipList<K, V>>,
  }
  
  impl<'a, K, V> Iterator for SkipListIter<'a, K, V> {
      type Item = (&'a K, &'a V);
  }
  ```
- [ ] Implement `range(start..end)` iterator
- [ ] Handle concurrent modification during iteration

**Afternoon — Memory Reclamation**

**Option A: Use crossbeam-epoch (simpler)**
- [ ] Add `crossbeam-epoch` dependency
- [ ] Wrap operations in `epoch::pin()`
- [ ] Defer node deallocation

**Option B: Build your own (educational)**
- [ ] Implement simple epoch-based reclamation:
  ```rust
  struct Epoch {
      global: AtomicU64,
      local: ThreadLocal<AtomicU64>,
      garbage: ThreadLocal<Vec<DeferredDrop>>,
  }
  ```
- [ ] Implement `pin()` / `unpin()` guards
- [ ] Implement garbage collection when safe

**Evening Tasks**
- [ ] Test with `loom` for concurrency bugs
- [ ] Verify no memory leaks with valgrind
- [ ] Benchmark concurrent operations

**Memory Reclamation Challenge:**
- Can't free node while readers might access it
- Epoch-based: track "eras", free when all threads advanced
- Alternative: hazard pointers (more complex)

---

### Day 7: Memtable Implementation

**Morning Tasks**
- [ ] Design memtable interface:
  ```rust
  pub struct Memtable {
      data: SkipList<InternalKey, Vec<u8>>,
      size: AtomicUsize,
      state: AtomicU8,  // Active, Frozen, Flushing
  }
  ```
- [ ] Define `InternalKey`:
  ```rust
  struct InternalKey {
      user_key: Vec<u8>,
      sequence: u64,
      value_type: ValueType,  // Put or Delete
  }
  ```

**Afternoon Tasks**
- [ ] Implement `put(key, value, seq)`:
  ```rust
  pub fn put(&self, key: &[u8], value: &[u8], seq: u64) -> Result<()>
  ```
- [ ] Implement `get(key)` returning latest version
- [ ] Implement `delete(key, seq)` as tombstone
- [ ] Track approximate memory size

**Evening Tasks**
- [ ] Implement `freeze()` state transition
- [ ] Implement `MemtableIterator` for flush
- [ ] Test concurrent read/write

**Memtable Sizing:**
- Default: 64MB
- Track: key size + value size + node overhead (~40 bytes)
- Freeze when threshold reached

---

### Day 8: WAL Design & Record Format

**Morning Study**
- [ ] Understand fsync guarantees:
  - `write()` — to kernel buffer
  - `fsync()` — to disk (or disk cache)
  - `fdatasync()` — data only, not metadata
- [ ] Study torn write scenarios
- [ ] Learn about sector sizes (512B or 4KB)

**Afternoon Design**
- [ ] Design WAL record format:
  ```
  +----------+----------+----------+------+------+-------+
  | CRC32    | Length   | SeqNum   | Type | KLen | Key   |
  | 4 bytes  | 4 bytes  | 8 bytes  | 1 B  | var  | var   |
  +----------+----------+----------+------+------+-------+
  | VLen     | Value    | Padding (optional)             |
  | varint   | var      |                                |
  +----------+----------+--------------------------------+
  ```
- [ ] Plan record types: `Put`, `Delete`, `BatchStart`, `BatchEnd`
- [ ] Design batch encoding

**Evening Tasks**
- [ ] Implement record serialization
- [ ] Implement record deserialization
- [ ] Test round-trip correctness

---

### Day 9: WAL Writer Implementation

**Morning Tasks**
- [ ] Implement `WalWriter`:
  ```rust
  pub struct WalWriter {
      file: File,
      buffer: Vec<u8>,
      offset: u64,
      sync_policy: SyncPolicy,
  }
  
  pub enum SyncPolicy {
      EveryWrite,
      EveryNWrites(usize),
      EveryNBytes(usize),
      Manual,
  }
  ```
- [ ] Implement buffered append
- [ ] Add CRC32 checksum to each record

**Afternoon Tasks**
- [ ] Implement `sync()` with proper fsync:
  ```rust
  pub fn sync(&mut self) -> io::Result<()> {
      self.file.write_all(&self.buffer)?;
      self.buffer.clear();
      self.file.sync_data()?;  // fdatasync
      Ok(())
  }
  ```
- [ ] Implement batch API for group commit
- [ ] Handle write errors gracefully

**Evening Tasks**
- [ ] Benchmark write throughput:
  - Sync every write
  - Sync every 100 writes
  - Sync every 1MB
- [ ] Measure latency distribution (p50, p99, p999)
- [ ] Test partial write handling

**WAL Performance:**
- Buffering critical for throughput
- fdatasync sufficient (metadata sync unnecessary)
- Target: >100MB/s sequential write

---

### Day 10: WAL Recovery Implementation

**Morning Tasks**
- [ ] Implement `WalReader`:
  ```rust
  pub struct WalReader {
      file: File,
      buffer: Vec<u8>,
      offset: u64,
  }
  ```
- [ ] Implement record parsing with CRC verification
- [ ] Handle corrupted records:
  - Log warning
  - Skip to next valid record
  - Or stop at corruption (configurable)

**Afternoon Tasks**
- [ ] Implement recovery logic:
  ```rust
  pub fn recover(wal_path: &Path) -> Result<(Memtable, u64)> {
      let mut memtable = Memtable::new();
      let mut max_seq = 0;
      for record in WalReader::open(wal_path)? {
          match record? {
              Record::Put { key, value, seq } => {
                  memtable.put(&key, &value, seq);
                  max_seq = max_seq.max(seq);
              }
              Record::Delete { key, seq } => {
                  memtable.delete(&key, seq);
                  max_seq = max_seq.max(seq);
              }
          }
      }
      Ok((memtable, max_seq))
  }
  ```
- [ ] Handle WAL file rotation

**Evening Tasks**
- [ ] Crash testing:
  - Kill process during write
  - Kill during sync
  - Corrupt random bytes
- [ ] Verify no data loss for synced writes
- [ ] Benchmark recovery time

---

### Day 11: Integration — Write Path

**Morning Tasks**
- [ ] Integrate WAL + Memtable:
  ```rust
  impl Db {
      pub fn put(&self, key: &[u8], value: &[u8]) -> Result<()> {
          let seq = self.next_sequence();
          
          // Durability first
          self.wal.lock().append_put(key, value, seq)?;
          
          // Then memory
          self.memtable.put(key, value, seq);
          
          // Check flush threshold
          self.maybe_schedule_flush();
          
          Ok(())
      }
  }
  ```

**Afternoon Tasks**
- [ ] Implement atomic write batches:
  ```rust
  pub fn write_batch(&self, batch: &WriteBatch) -> Result<()>
  ```
- [ ] Implement sequence number allocation
- [ ] Add write throttling hook (for Week 3)

**Evening Tasks**
- [ ] End-to-end test: write → crash → recover → verify
- [ ] Stress test with concurrent writers
- [ ] Document Week 1 architecture

---

## Week 2: SSTable Format & Disk Storage

### Learning Objectives
- Design efficient binary file format
- Implement zero-copy reads
- Build block-based compression

---

### Day 12: LZ4 Compression — Theory & Core

**Morning Study**
- [ ] Study LZ4 algorithm:
  - Sequence format: literal length + match offset + match length
  - Sliding window (64KB)
  - Hash table for match finding
- [ ] Understand block vs frame format
- [ ] Study decompression (simpler than compression)

**Afternoon Implementation**
- [ ] Implement LZ4 decompression first (easier):
  ```rust
  pub fn lz4_decompress(src: &[u8], dst: &mut Vec<u8>) -> Result<()>
  ```
- [ ] Parse sequence headers
- [ ] Copy literals and matches

**Evening Tasks**
- [ ] Test with official LZ4 test vectors
- [ ] Verify decompression correctness
- [ ] Benchmark decompression speed (target: >1GB/s)

---

### Day 13: LZ4 Compression — Encoder

**Morning Tasks**
- [ ] Implement hash table for match finding:
  ```rust
  struct HashTable {
      table: [u16; 4096],  // Position table
  }
  ```
- [ ] Implement match finder

**Afternoon Tasks**
- [ ] Implement LZ4 compression:
  ```rust
  pub fn lz4_compress(src: &[u8], dst: &mut Vec<u8>) -> Result<()>
  ```
- [ ] Encode literals and matches
- [ ] Handle edge cases (small inputs, no matches)

**Evening Tasks**
- [ ] Test compression round-trip
- [ ] Benchmark compression ratio and speed
- [ ] Compare with official LZ4 (sanity check)

**LZ4 Targets:**
- Compression: >200MB/s
- Decompression: >1GB/s
- Ratio: 2-3x typical (workload dependent)

---

### Day 14: SSTable Architecture & Block Format

**Morning Study**
- [ ] Study LevelDB/RocksDB table format
- [ ] Design your SSTable layout:
  ```
  [Data Block 0]       // Sorted key-value pairs
  [Data Block 1]
  ...
  [Data Block N]
  [Filter Block]       // Bloom filter
  [Index Block]        // Block handles
  [Footer]             // Offsets + magic
  ```

**Afternoon Design**
- [ ] Design data block format:
  ```
  +------------------+
  | Entry 0          |  KLen(var) | Key | VLen(var) | Value
  | Entry 1          |  (delta-encoded key)
  | ...              |
  | Restart 0        |  Full key every N entries
  | ...              |
  +------------------+
  | Restarts[]: u32  |  Offsets of restart points
  | NumRestarts: u32 |
  | Compression: u8  |
  | CRC32: u32       |
  +------------------+
  ```
- [ ] Design index block format (same structure, different content)
- [ ] Design footer format

**Evening Tasks**
- [ ] Document complete file format specification
- [ ] Calculate overhead for typical workloads
- [ ] Plan versioning for format evolution

---

### Day 15: Block Builder Implementation

**Morning Tasks**
- [ ] Implement `BlockBuilder`:
  ```rust
  pub struct BlockBuilder {
      buffer: Vec<u8>,
      restarts: Vec<u32>,
      last_key: Vec<u8>,
      entry_count: usize,
      restart_interval: usize,  // Default: 16
  }
  ```
- [ ] Implement key delta encoding:
  ```rust
  fn add(&mut self, key: &[u8], value: &[u8]) {
      let shared = common_prefix_len(&self.last_key, key);
      let non_shared = key.len() - shared;
      // Encode: shared_len | non_shared_len | non_shared_bytes | value_len | value
  }
  ```

**Afternoon Tasks**
- [ ] Implement restart point logic
- [ ] Implement `finish()`:
  - Append restart array
  - Append restart count
  - Optionally compress
  - Append CRC32

**Evening Tasks**
- [ ] Test with various key/value sizes
- [ ] Verify compression integration
- [ ] Benchmark block building throughput

---

### Day 16: Block Reader Implementation

**Morning Tasks**
- [ ] Implement `BlockReader`:
  ```rust
  pub struct BlockReader<'a> {
      data: &'a [u8],
      restarts_offset: usize,
      num_restarts: usize,
  }
  ```
- [ ] Implement restart point binary search
- [ ] Implement key decoding

**Afternoon Tasks**
- [ ] Implement `BlockIterator`:
  ```rust
  impl<'a> BlockIterator<'a> {
      pub fn seek(&mut self, target: &[u8]);
      pub fn next(&mut self) -> Option<(&'a [u8], &'a [u8])>;
  }
  ```
- [ ] Zero-copy: return slices into block data
- [ ] Handle decompression (decompress once, iterate many)

**Evening Tasks**
- [ ] Benchmark point lookup (<1μs target)
- [ ] Benchmark iteration (>5M entries/sec)
- [ ] Test edge cases

---

### Day 17: SSTable Writer

**Morning Tasks**
- [ ] Implement `SstWriter`:
  ```rust
  pub struct SstWriter {
      file: BufWriter<File>,
      data_block: BlockBuilder,
      index_block: BlockBuilder,
      pending_index_entry: Option<IndexEntry>,
      offset: u64,
      props: TableProperties,
  }
  ```
- [ ] Implement `add(key, value)`:
  - Add to current data block
  - Flush block if full (default: 4KB)
  - Record index entry

**Afternoon Tasks**
- [ ] Implement `finish()`:
  ```rust
  pub fn finish(mut self) -> Result<SstMeta> {
      self.flush_data_block()?;
      let filter_offset = self.write_filter_block()?;
      let index_offset = self.write_index_block()?;
      self.write_footer(filter_offset, index_offset)?;
      self.file.sync_all()?;
      Ok(self.props.into())
  }
  ```
- [ ] Implement atomic file creation (write temp, rename)

**Evening Tasks**
- [ ] Test SSTable creation
- [ ] Verify format with hex dump inspection
- [ ] Benchmark write throughput (>50MB/s)

---

### Day 18: SSTable Reader

**Morning Tasks**
- [ ] Implement `SstReader`:
  ```rust
  pub struct SstReader {
      // Memory-mapped file for zero-copy reads
      mmap: Mmap,
      footer: Footer,
      index_block: BlockReader<'static>,
      filter: Option<BloomFilter>,
      file_id: u64,
  }
  ```
- [ ] Implement file opening: read footer, parse index

**Afternoon Tasks**
- [ ] Implement point lookup:
  ```rust
  pub fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>> {
      // 1. Check bloom filter (Week 4)
      // 2. Binary search index block
      // 3. Load and search data block
  }
  ```
- [ ] Implement `SstIterator` for range scans

**Evening Tasks**
- [ ] Benchmark read latency
- [ ] Test with various file sizes
- [ ] Compare mmap vs pread performance

---

### Day 19: Table Cache & Flush Pipeline

**Morning Tasks**
- [ ] Implement LRU cache from scratch:
  ```rust
  pub struct LruCache<K, V> {
      map: HashMap<K, NonNull<Node<K, V>>>,
      head: *mut Node<K, V>,  // Most recent
      tail: *mut Node<K, V>,  // Least recent
      capacity: usize,
  }
  ```
- [ ] Implement `get`, `put`, eviction

**Afternoon Tasks**
- [ ] Implement `TableCache`:
  ```rust
  pub struct TableCache {
      cache: Mutex<LruCache<u64, Arc<SstReader>>>,
      db_path: PathBuf,
  }
  ```
- [ ] Implement flush pipeline:
  ```rust
  fn flush_memtable(&self, mem: &Memtable) -> Result<SstMeta>
  ```

**Evening Tasks**
- [ ] Test cache eviction behavior
- [ ] Test flush correctness
- [ ] Benchmark flush throughput

---

## Week 3: Compaction & Crash Recovery

### Learning Objectives
- Implement multi-way merge efficiently
- Design robust manifest system
- Handle all crash scenarios

---

### Day 20: Merge Iterator

**Morning Tasks**
- [ ] Implement binary heap for k-way merge:
  ```rust
  pub struct MergeIterator<I: Iterator> {
      heap: BinaryHeap<Reverse<HeapItem<I>>>,
  }
  
  struct HeapItem<I> {
      key: Vec<u8>,
      value: Vec<u8>,
      iter: I,
      iter_id: usize,  // For tie-breaking
  }
  ```
- [ ] Implement proper ordering (by key, then by sequence number)

**Afternoon Tasks**
- [ ] Implement deduplication (keep newest version only)
- [ ] Handle tombstones
- [ ] Optimize to reduce allocations

**Evening Tasks**
- [ ] Test merge correctness with overlapping ranges
- [ ] Benchmark merge throughput
- [ ] Test with many input iterators (10, 50, 100)

---

### Day 21: Compaction — Core Logic

**Morning Study**
- [ ] Review leveled compaction algorithm
- [ ] Understand compaction picking strategies
- [ ] Study write amplification math

**Afternoon Tasks**
- [ ] Implement level structure:
  ```rust
  pub struct LevelState {
      levels: Vec<Vec<Arc<SstMeta>>>,
      level_max_bytes: Vec<u64>,
  }
  ```
- [ ] Implement compaction job:
  ```rust
  pub struct CompactionJob {
      input_level: usize,
      output_level: usize,
      inputs: Vec<Arc<SstMeta>>,
  }
  ```

**Evening Tasks**
- [ ] Implement file picking for L0 → L1
- [ ] Test compaction selection
- [ ] Document compaction algorithm

---

### Day 22: Compaction — Execution

**Morning Tasks**
- [ ] Implement compaction execution:
  ```rust
  fn execute_compaction(&self, job: &CompactionJob) -> Result<Vec<SstMeta>> {
      // 1. Create merge iterator over inputs
      // 2. Write to new SSTable(s)
      // 3. Split at target file size
      // 4. Return new file metadata
  }
  ```
- [ ] Implement output file splitting (~64MB target)

**Afternoon Tasks**
- [ ] Implement tombstone dropping logic:
  - Drop if at bottom level
  - Drop if no snapshots reference it
- [ ] Handle sequence number filtering

**Evening Tasks**
- [ ] Test compaction correctness
- [ ] Verify data integrity after compaction
- [ ] Test concurrent reads during compaction

---

### Day 23: Manifest Design & Implementation

**Morning Study**
- [ ] Study MANIFEST file format
- [ ] Understand version edits
- [ ] Plan atomic version switches

**Afternoon Tasks**
- [ ] Implement `VersionEdit`:
  ```rust
  pub enum VersionEdit {
      AddFile { level: u32, meta: SstMeta },
      RemoveFile { level: u32, file_id: u64 },
      SetLogNumber(u64),
      SetLastSequence(u64),
      SetNextFileNumber(u64),
  }
  ```
- [ ] Implement manifest writer (append-only)
- [ ] Implement manifest reader

**Evening Tasks**
- [ ] Test manifest serialization round-trip
- [ ] Implement CURRENT file pointer
- [ ] Test manifest recovery

---

### Day 24: Crash Recovery

**Morning Tasks**
- [ ] Implement full recovery sequence:
  ```rust
  pub fn open(path: &Path) -> Result<Db> {
      // 1. Read CURRENT file
      // 2. Replay manifest → reconstruct levels
      // 3. Replay WAL → reconstruct memtable
      // 4. Resume operations
  }
  ```
- [ ] Handle missing/corrupt manifest

**Afternoon Tasks**
- [ ] Handle crash during compaction:
  - Orphaned output files → delete
  - Incomplete compaction → rollback
- [ ] Handle crash during flush
- [ ] Implement consistency verification

**Evening Tasks**
- [ ] Comprehensive crash testing:
  - Crash during put
  - Crash during flush
  - Crash during compaction
  - Crash during manifest write
- [ ] Verify zero data loss

---

### Day 25: Background Compaction & Scheduling

**Morning Tasks**
- [ ] Implement background compaction thread:
  ```rust
  fn compaction_thread(db: Arc<DbInner>) {
      loop {
          let job = db.pick_compaction();
          if let Some(job) = job {
              db.execute_compaction(&job)?;
              db.apply_compaction_result(&job)?;
          } else {
              db.compact_cond.wait();
          }
      }
  }
  ```

**Afternoon Tasks**
- [ ] Implement write throttling:
  ```rust
  fn maybe_throttle(&self) {
      let l0_count = self.level_state.l0_count();
      if l0_count > L0_SLOWDOWN_TRIGGER {
          thread::sleep(compute_delay(l0_count));
      }
      if l0_count > L0_STOP_TRIGGER {
          self.stall_cond.wait();
      }
  }
  ```
- [ ] Implement compaction rate limiting

**Evening Tasks**
- [ ] Test under sustained write load
- [ ] Measure write stalls
- [ ] Tune throttling parameters

---

### Day 26: Concurrent Access & File Lifecycle

**Morning Tasks**
- [ ] Implement version reference counting:
  ```rust
  pub struct Version {
      levels: Vec<Vec<Arc<SstMeta>>>,
      refs: AtomicUsize,
  }
  ```
- [ ] Implement safe file deletion (only when unreferenced)

**Afternoon Tasks**
- [ ] Implement read path with version pinning:
  ```rust
  pub fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>> {
      let version = self.current_version();  // Pins version
      // Search memtable, then levels...
      // Version unpinned on drop
  }
  ```
- [ ] Test concurrent readers during compaction

**Evening Tasks**
- [ ] Stress test with readers + writers + compaction
- [ ] Run `loom` tests
- [ ] Profile lock contention

---

## Week 4: Query Optimizations & Polish

### Learning Objectives
- Implement Bloom filters from scratch
- Build efficient block cache
- Production-ready testing

---

### Day 27: Bloom Filter — Theory & Implementation

**Morning Study**
- [ ] Bloom filter math:
  ```
  False positive rate: p ≈ (1 - e^(-kn/m))^k
  Optimal k: k = (m/n) * ln(2) ≈ 0.693 * (m/n)
  ```
  Where: m = bits, n = entries, k = hash functions
- [ ] Study double hashing technique (2 hashes → k probes)

**Afternoon Implementation**
- [ ] Implement `BloomFilter`:
  ```rust
  pub struct BloomFilter {
      bits: Vec<u64>,  // Bit vector
      num_hashes: u32,
      num_bits: usize,
  }
  
  impl BloomFilter {
      pub fn add(&mut self, key: &[u8]);
      pub fn may_contain(&self, key: &[u8]) -> bool;
  }
  ```
- [ ] Use your xxHash implementation
- [ ] Implement double hashing for k probes

**Evening Tasks**
- [ ] Test false positive rate empirically
- [ ] Verify zero false negatives
- [ ] Benchmark probe performance (<100ns)

**Bloom Filter Config:**
- 10 bits/key → ~1% false positive rate
- 2 hash functions computed, k probes derived

---

### Day 28: Bloom Filter Integration

**Morning Tasks**
- [ ] Build filter during SSTable creation:
  ```rust
  impl SstWriter {
      fn add(&mut self, key: &[u8], value: &[u8]) {
          self.filter_builder.add(extract_user_key(key));
          // ... rest
      }
  }
  ```
- [ ] Write filter block to SSTable

**Afternoon Tasks**
- [ ] Load filter on SSTable open
- [ ] Integrate into read path:
  ```rust
  fn get(&self, key: &[u8]) -> Result<Option<Vec<u8>>> {
      if let Some(ref filter) = self.filter {
          if !filter.may_contain(key) {
              return Ok(None);  // Definitely not here
          }
      }
      // Continue with block lookup...
  }
  ```

**Evening Tasks**
- [ ] Measure read improvement (10-100x for missing keys)
- [ ] Test filter effectiveness
- [ ] Tune bits-per-key

---

### Day 29: Block Cache

**Morning Tasks**
- [ ] Enhance LRU cache for blocks:
  ```rust
  pub struct BlockCache {
      shards: Vec<Mutex<LruCache<CacheKey, Arc<Block>>>>,
  }
  
  #[derive(Hash, Eq, PartialEq)]
  struct CacheKey {
      file_id: u64,
      block_offset: u64,
  }
  ```
- [ ] Implement sharding (16 shards) to reduce contention

**Afternoon Tasks**
- [ ] Integrate cache into SSTable reader:
  ```rust
  fn read_block(&self, handle: BlockHandle) -> Result<Arc<Block>> {
      let key = CacheKey::new(self.file_id, handle.offset);
      
      if let Some(block) = self.cache.get(&key) {
          return Ok(block);
      }
      
      let block = self.load_block_from_disk(handle)?;
      self.cache.insert(key, block.clone());
      Ok(block)
  }
  ```
- [ ] Add cache statistics (hits, misses, evictions)

**Evening Tasks**
- [ ] Benchmark cached vs uncached reads
- [ ] Test cache under memory pressure
- [ ] Tune cache size

---

### Day 30: Comprehensive Testing

**Morning Tasks**
- [ ] Randomized correctness testing:
  ```rust
  #[test]
  fn fuzz_db_operations() {
      let db = Db::open_temp()?;
      let mut reference = BTreeMap::new();
      
      for _ in 0..100_000 {
          match random_op() {
              Op::Put(k, v) => {
                  db.put(&k, &v)?;
                  reference.insert(k, v);
              }
              Op::Delete(k) => {
                  db.delete(&k)?;
                  reference.remove(&k);
              }
              Op::Get(k) => {
                  assert_eq!(db.get(&k)?, reference.get(&k).cloned());
              }
          }
      }
  }
  ```

**Afternoon Tasks**
- [ ] Crash recovery testing with fault injection
- [ ] Run `loom` tests for all concurrent paths
- [ ] Long-running stability tests

**Evening Tasks**
- [ ] Fix any discovered bugs
- [ ] Document test coverage
- [ ] Write known limitations

---

### Day 31: Benchmarking

**Morning Tasks**
- [ ] Create benchmark suite with `criterion`:
  ```rust
  fn bench_sequential_write(c: &mut Criterion) {
      c.bench_function("seq_write_100k", |b| {
          b.iter(|| {
              let db = Db::open_temp().unwrap();
              for i in 0..100_000 {
                  db.put(&i.to_be_bytes(), &[0u8; 100]).unwrap();
              }
          });
      });
  }
  ```
- [ ] Benchmark: seq write, random write, point read, range scan

**Afternoon Tasks**
- [ ] Profile with `perf` and generate flamegraphs
- [ ] Identify hot spots
- [ ] Measure vs reference implementations (if curious)

**Evening Tasks**
- [ ] Document benchmark results
- [ ] Create performance regression tests
- [ ] Note optimization opportunities

---

### Day 32: Final Polish & Documentation

**Morning Tasks**
- [ ] Code review all modules
- [ ] Ensure consistent error handling
- [ ] Add documentation comments

**Afternoon Tasks**
- [ ] Write architecture documentation
- [ ] Document public API with examples
- [ ] Create tuning guide

**Evening Tasks**
- [ ] Final end-to-end testing
- [ ] Write retrospective: what you learned
- [ ] Plan future enhancements

---

## Components Built From Scratch

| Component | Lines (approx) | Complexity |
|-----------|----------------|------------|
| CRC32 | ~50 | Low |
| Hash functions | ~100 | Low |
| Varint encoding | ~50 | Low |
| LZ4 compression | ~300 | Medium |
| Skip list | ~500 | High |
| LRU cache | ~150 | Medium |
| Bloom filter | ~100 | Medium |
| WAL | ~400 | Medium |
| SSTable | ~800 | High |
| Compaction | ~600 | High |
| Manifest | ~300 | Medium |
| **Total** | **~3500** | |

---

## Success Criteria

By end of Week 4:

1. **Zero external runtime dependencies** (except optional crossbeam-epoch)
2. **Crash-safe**: Survives kill -9 at any point
3. **Correct**: Passes fuzz tests, `loom` tests
4. **Fast**: Meets performance targets
5. **Understood**: You can explain every line

---

## What You've Mastered

After completing this plan, you will deeply understand:

- Lock-free programming and memory ordering
- Binary file format design
- Crash consistency and recovery
- Cache algorithms and systems
- Compression algorithms
- Probabilistic data structures
- Database storage internals

This is senior/staff-level systems knowledge. Ship it. 🚀

