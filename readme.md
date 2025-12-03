# TinyLSM

> LSM-tree storage engine in pure Rust. Built from first principles for deep understanding of database internals.

---

## About

A from-scratch implementation of an LSM-tree storage engine in Rust. The focus is on understanding database internals deeply — not reinventing every algorithm.

**Built from scratch:**
- Lock-free skip list with epoch-based memory reclamation
- Bloom filters
- LRU block cache
- SSTable format and block encoding
- Write-ahead logging and crash recovery
- Leveled compaction

**Using battle-tested libraries for:**
- CRC32 checksums
- LZ4 compression
- Hash functions

This keeps the focus on what matters: the storage engine architecture.

---

## Features

- **Crash-safe**: Write-ahead log with proper fsync semantics
- **Fast writes**: In-memory skip list memtable, sequential disk I/O
- **Efficient reads**: Bloom filters, block cache, indexed SSTables
- **Space efficient**: LZ4 compression, leveled compaction
- **Concurrent**: Lock-free reads, background compaction

---

## Architecture

```
                    ┌─────────────────────────────────────┐
     Writes ───────►│           Memtable                  │
                    │        (Skip List)                  │
                    └──────────────┬──────────────────────┘
                                   │ flush
                    ┌──────────────▼──────────────────────┐
                    │            Level 0                  │
                    │    (Recently flushed SSTables)      │
                    └──────────────┬──────────────────────┘
                                   │ compaction
                    ┌──────────────▼──────────────────────┐
                    │            Level 1                  │
                    ├─────────────────────────────────────┤
                    │            Level 2                  │
                    ├─────────────────────────────────────┤
                    │            Level N                  │
                    └─────────────────────────────────────┘
```

---

## 4-Week Implementation Plan

### Week 1: Foundations — Memtable & Write-Ahead Log
- Skip list implementation (lock-free)
- Memtable with sequence numbers
- WAL record format and crash recovery
- Serialization utilities (varint encoding)

### Week 2: SSTable Format & Disk Storage
- Block-based SSTable format
- Delta-encoded keys with restart points
- Index blocks and footer
- Zero-copy reads with mmap
- Table cache

### Week 3: Compaction & Crash Recovery  
- Merge iterator (k-way merge)
- Leveled compaction
- Manifest and version control
- Full crash recovery
- Background compaction thread

### Week 4: Query Optimizations & Polish
- Bloom filters
- Block cache (sharded LRU)
- Comprehensive testing
- Benchmarking

---

## Dependencies

```toml
[dependencies]
crc32fast = "1.3"          # Checksums
lz4_flex = "0.11"          # Compression  
xxhash-rust = "0.8"        # Hashing for bloom filters

[dev-dependencies]
criterion = "0.5"          # Benchmarking
tempfile = "3.10"          # Test utilities
rand = "0.8"               # Test utilities
```

---

## Performance Targets

| Operation | Target |
|-----------|--------|
| Sequential write | 100K+ ops/sec |
| Random write | 50K+ ops/sec |
| Point read (cached) | <10μs |
| Point read (disk) | <100μs |
| Range scan | 5M+ entries/sec |

---

## What You'll Learn

- Lock-free programming and atomic operations
- Binary file format design
- Crash consistency and recovery semantics  
- Write/read/space amplification trade-offs
- Cache replacement algorithms
- Probabilistic data structures (bloom filters)
- Database storage engine internals

---

## References

- [LevelDB Implementation Notes](https://github.com/google/leveldb/blob/main/doc/impl.md)
- [RocksDB Wiki](https://github.com/facebook/rocksdb/wiki)
- [Designing Data-Intensive Applications](https://dataintensive.net/) — Chapter 3
- [Skip Lists: A Probabilistic Alternative to Balanced Trees](https://15721.courses.cs.cmu.edu/spring2018/papers/08-oltpindexes1/pugh-skiplists-cacm1990.pdf)

---

## License

MIT

