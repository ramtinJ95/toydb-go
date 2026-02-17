# Building a Distributed Database in Go: Complete 6-Month Curriculum

## ToyDB Feature Parity Reference Implementation

This curriculum produces a distributed SQL database with exact feature parity to [erikgrinaker/toydb](https://github.com/erikgrinaker/toydb). Every section maps directly to ToyDB's actual implementation, corrected for accuracy based on source code analysis.

---

## Final Validation Summary

✅ **Architecture Match Confirmed**: Bottom-up order (Storage → Encoding → MVCC → Raft → SQL) matches ToyDB's actual dependency graph  
✅ **All Erik's References Included**: Every source from ToyDB's `docs/references.md` is incorporated  
✅ **Feature Parity Verified**: All SQL features (including secondary indexes, foreign keys, referential integrity) covered  
✅ **Intentional Omissions Documented**: No snapshots, no SSI, no GC — matches ToyDB's educational simplifications  
✅ **Go-Specific Adaptations**: Idiomatic patterns and libraries recommended for each component

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                             │
│                 (TCP + Bincode, No Framing)                      │
└─────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                         Server/Session                           │
│            (Session State, Transaction Control)                  │
└─────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                          SQL Engine                              │
│  ┌──────────┐   ┌──────────┐   ┌───────────┐   ┌────────────┐  │
│  │  Lexer   │ → │  Parser  │ → │  Planner  │ → │  Executor  │  │
│  │          │   │(Prec.    │   │(Heuristic │   │ (Iterator  │  │
│  │          │   │ Climbing)│   │ Optimizer)│   │   Model)   │  │
│  └──────────┘   └──────────┘   └───────────┘   └────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Raft Consensus Layer                          │
│     (Leader Election, Log Replication, Quorum Reads)             │
│     [No Snapshots, No Membership Changes - Intentionally]        │
└─────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                MVCC Transaction Layer                            │
│          (Snapshot Isolation, Lowest-Txn-ID Wins)                │
│          [No Write Skew Detection - Intentionally]               │
└─────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Encoding Layer                                │
│         (Keycode: Order-Preserving Binary Encoding)              │
└─────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                     Storage Engine                               │
│        ┌─────────────┐              ┌─────────────┐             │
│        │   BitCask   │              │   Memory    │             │
│        │ (Log + Hash │              │  (BTreeMap) │             │
│        │   Index)    │              │             │             │
│        └─────────────┘              └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Dependency Graph (Implementation Order)

```
Week 1-2: Storage Engine Trait + Memory Engine
    ↓                          ↓
    ↓                   Week 3-4: BitCask Persistent Engine
    ↓                          ↓
    ├──────────────────────────┘
    ↓
Week 5-6: Keycode Order-Preserving Encoding
    ↓
Week 7-11: MVCC Transaction Layer (Snapshot Isolation)
    ↓
    ├─────────────────────────────────────────────────┐
    ↓                                                 ↓
Week 12-17: Raft Consensus                   Week 18-20: SQL Lexer + Parser
(Election, Replication,                      (Precedence Climbing)
 Linearizable Reads)                                  ↓
    ↓                                        Week 21-23: SQL Planner +
    ↓                                        Heuristic Optimizer
    ↓                                                 ↓
    ├─────────────────────────────────────────────────┘
    ↓
Week 24-25: SQL Executor (Iterator Model, Joins, Aggregates)
    ↓
Week 26: Server/Client Integration + Testing

Note: BitCask and Keycode are independent of each other (both feed into MVCC).
Raft and SQL Parser are also independent (both need MVCC; integration happens
in Week 24-26). Teams of 2+ developers can parallelize these tracks.
```

---

# PHASE 1: STORAGE FOUNDATIONS (Weeks 1-6)

## Module 1.1: Storage Engine Abstraction (Week 1-2)

### Learning Objectives
- Design a pluggable storage engine interface
- Implement an in-memory engine using Go's standard library
- Understand iterator patterns for range scans (including reverse scans)
- Expose explicit durability via a flush operation

### What ToyDB Implements
```go
// Go equivalent of ToyDB's Engine trait
type Engine interface {
    // Delete removes a key, or does nothing if it does not exist
    Delete(key []byte) error

    // Flush persists buffered data to disk
    Flush() error

    // Get retrieves a value by key
    Get(key []byte) (value []byte, ok bool, err error)

    // Scan returns an iterator over a key range (inclusive/exclusive bounds)
    Scan(range Range) Iterator

    // ScanPrefix returns an iterator over keys with a given prefix
    ScanPrefix(prefix []byte) Iterator

    // Set stores a key-value pair
    Set(key []byte, value []byte) error

    // Status returns engine statistics
    Status() (Status, error)
}

type Iterator interface {
    // Next advances the iterator and returns the next key-value pair
    // Returns nil, nil when exhausted
    Next() (key []byte, value []byte, err error)

    // Prev moves the iterator backwards (reverse scan)
    // Returns nil, nil when exhausted
    Prev() (key []byte, value []byte, err error)

    // Close releases iterator resources
    Close() error
}

type Range struct {
    Start          []byte
    End            []byte
    StartInclusive bool
    EndInclusive   bool
}

type Status struct {
    Name         string
    Keys         uint64
    Size         uint64
    DiskSize     uint64
    LiveDiskSize uint64
}

ToyDB's storage engine is single-user (methods take mutable access). In Go, serialize access with a mutex or ensure a single goroutine owns the engine.
```

### Required Readings

**Core Concepts:**
| Reading | Why It Matters |
|---------|----------------|
| [Go Blog: Errors are Values](https://go.dev/blog/errors-are-values) | Go error handling patterns you'll use throughout |
| [Effective Go: Interfaces](https://go.dev/doc/effective_go#interfaces) | Interface design philosophy for the Engine trait |

**Reference Implementations to Study:**
| Project | What to Learn |
|---------|---------------|
| [bbolt](https://github.com/etcd-io/bbolt) | Study the `DB` and `Bucket` interfaces for inspiration |
| [pebble Iterator](https://github.com/cockroachdb/pebble/blob/master/iterator.go) | Production iterator patterns |

### Implementation Assignment

**Week 1: Engine Interface + Memory Engine**
1. Define the `Engine` interface in `storage/engine.go` (including `Flush`, `ScanPrefix`, and reverse scans)
2. Implement `MemoryEngine` using `sync.RWMutex` + Go's `btree` package (or sorted keys over a map)
3. Implement the `Iterator` interface with forward and reverse scans

**Week 2: Testing + Edge Cases**
1. Write table-driven tests for all Engine operations
2. Handle edge cases: empty ranges, non-existent keys, nil values vs deleted keys
3. Implement `Status()` returning key count, logical size, disk size, and live disk size

### Deliverable
Working `MemoryEngine` passing all interface tests, with reverse scans and explicit `Flush()` durability.

---

## Module 1.2: BitCask Persistent Storage (Week 3-4)

### Learning Objectives
- Understand log-structured storage and append-only writes
- Implement an in-memory hash index (key directory)
- Build log compaction and crash recovery
- Add exclusive file locking to prevent concurrent opens

### What ToyDB Implements

BitCask is a log-structured hash table (single log file in ToyDB):
- **Writes**: Append key-value record to log file, update in-memory index
- **Reads**: Look up file offset in index, read single record from disk
- **Deletes**: Append tombstone record, remove from index
- **Compaction**: Rewrite entire log keeping only live entries
- **Simplifications**: Single log file, no hint files, compaction runs on open

```
Log File Format (big-endian):
┌──────────┬──────────┬─────────┬───────┬───────────┐
│ key_len  │ val_len  │   key   │ value │  (repeat) │
│ (u32 BE) │ (i32 BE) │(variable│       │           │
│          │ -1=tomb) │         │       │           │
└──────────┴──────────┴─────────┴───────┴───────────┘

In-Memory Index (KeyDir):
map[string]struct {
    offset int64  // Position in log file
    size   int64  // Record size for reading
}
```

### Required Readings

**Essential Papers:**
| Paper | Authors | Why It Matters |
|-------|---------|----------------|
| [Bitcask: A Log-Structured Hash Table](https://riak.com/assets/bitcask-intro.pdf) | Basho Technologies | Original design document - read this first |

**Blog Posts:**
| Post | Why It Matters |
|------|----------------|
| [Build Your Own Database (Log-Structured Storage)](https://build-your-own.org/database/04_btree_code_1) | Step-by-step implementation guidance |

**Go-Specific:**
| Resource | Why It Matters |
|----------|----------------|
| [os.File documentation](https://pkg.go.dev/os#File) | File I/O, Sync, Seek operations |
| [encoding/binary package](https://pkg.go.dev/encoding/binary) | Binary serialization for log records |

### Implementation Assignment

**Week 3: Core BitCask**
1. Implement log file format with length-prefixed records (u32 key, i32 value)
2. Build in-memory KeyDir index loaded on startup
3. Implement `Get`, `Set`, `Delete` (durability via explicit `Flush()`, not fsync per write)
4. Add exclusive file locking and crash recovery by truncating incomplete tail entries

**Week 4: Compaction + Scan**
1. Implement log compaction: rewrite log with only live entries (single file rewrite)
2. Add optional auto-compaction on open based on garbage fraction/size
3. Add `Scan` support via KeyDir range (forward and reverse)
4. Write durability tests: crash mid-write, verify recovery

### Deliverable
Persistent `BitCaskEngine` that survives restarts, with compaction reducing log size.

---

## Module 1.3: Keycode Order-Preserving Encoding (Week 5-6)

### Learning Objectives
- Understand why lexicographic ordering matters for range scans
- Implement order-preserving binary encoding for all SQL types
- Handle edge cases: negative numbers, floats, variable-length strings

### What ToyDB Implements

ToyDB's custom "Keycode" encoding ensures that `bytes.Compare(encode(a), encode(b))` matches the logical comparison of `a` and `b`. This is **critical** for MVCC keys where you need to scan versions in order. Keycode is not self-describing: decoding requires the expected type. Values are encoded separately using Bincode.

**Encoding Rules:**
| Type | Encoding Strategy |
|------|-------------------|
| `bool` | `0x00` = false, `0x01` = true |
| `u64` | Big-endian binary |
| `i64` | Flip sign bit, big-endian (so -1 < 0 < 1) |
| `f64` | IEEE 754 big-endian, flip sign bit; for negatives flip all bits |
| `String` | Escape `0x00` as `0x00 0xff`, terminate with `0x00 0x00` |
| `Vec<u8>` | Same as String (byte strings) |
| `Vec<T>` / tuples | Concatenate encoded elements |
| `enum` | Variant index as `u8`, then encoded fields |

**Why standard encoding fails:**
- Little-endian integers: `1` encodes as `01 00 00 00`, `256` as `00 01 00 00` → wrong order
- Standard float encoding: negative floats sort incorrectly
- Variable-length strings: need delimiter that doesn't appear in data

### Required Readings

**Essential:**
| Reading | Why It Matters |
|---------|----------------|
| [CockroachDB Key Encoding](https://github.com/cockroachdb/cockroach/blob/master/docs/tech-notes/encoding.md) | Production-grade order-preserving encoding |
| [FoundationDB Tuple Layer](https://apple.github.io/foundationdb/data-modeling.html#tuples) | Similar encoding scheme with excellent docs |
| [IEEE 754 Float Representation](https://en.wikipedia.org/wiki/IEEE_754) | Understanding float bit patterns |

**Go Implementation Reference:**
| Resource | Why It Matters |
|----------|----------------|
| [encoding/binary BigEndian](https://pkg.go.dev/encoding/binary#BigEndian) | Big-endian helpers |
| [math.Float64bits](https://pkg.go.dev/math#Float64bits) | Converting floats to sortable integers |

### Implementation Assignment

**Week 5: Basic Types**
1. Implement `Encode`/`Decode` for bool, u64, i64 with sign-bit flip
2. Implement f64 encoding handling positive/negative/NaN/Inf
3. Write exhaustive tests: random values, edge cases, ordering verification

**Week 6: Composite Keys**
1. Implement string/byte encoding with `0x00 -> 0x00 0xff` escaping and `0x00 0x00` terminator
2. Build tuple/sequence encoding: `EncodeTuple(values ...any) []byte`
3. Implement enum variant index encoding (u8 + fields)
4. Implement `prefix_range(prefix []byte)` helper for prefix scans
5. Implement MVCC key encoding structure (you will use this in Phase 2 — for now, just build the encode/decode logic):
    ```go
    type MVCCKey struct {
        Type    KeyType  // NextVersion, TxnActive, TxnActiveSnapshot, TxnWrite, Version, Unversioned
        Key     []byte   // User key (for Version/TxnWrite types)
        Version uint64   // Transaction ID
    }
    ```
    These key types will be explained in depth in Module 2.1 (MVCC Fundamentals). At this stage, focus on ensuring the encoding produces correct lexicographic ordering for each variant.

### Deliverable
`keycode` package with `Encode`/`Decode` for all types, passing ordering tests for 10,000+ random values.

---

# PHASE 2: MVCC TRANSACTIONS (Weeks 7-11)

## Module 2.1: MVCC Fundamentals (Week 7-8)

### Learning Objectives
- Understand Snapshot Isolation semantics and its limitations
- Implement version visibility rules
- Design the MVCC key-value schema
- Capture active-set snapshots for time-travel reads

### What ToyDB Implements

**Isolation Level: Snapshot Isolation (SI)**
- Each transaction sees a consistent snapshot as of its start time
- **Prevents**: dirty writes, dirty reads, lost updates, read skew, phantom reads
- **Allows**: write skew (two transactions read overlapping data, make independent decisions, both commit)

**MVCC Key Schema:**
```go
// MVCC key enum (Keycode-encoded; variant index is the prefix)
type KeyKind uint8

const (
    KeyNextVersion KeyKind = iota
    KeyTxnActive
    KeyTxnActiveSnapshot
    KeyTxnWrite
    KeyVersion
    KeyUnversioned
)

type Key struct {
    Kind    KeyKind
    Version uint64
    Key     []byte
}

// Examples:
// NextVersion:            [variant index]
// TxnActive(5):           [variant index, 0x00..0x05]
// TxnActiveSnapshot(5):   [variant index, 0x00..0x05]
// TxnWrite(5,key):        [variant index, 0x00..0x05, ...key...]
// Version(key,5):         [variant index, ...key..., 0x00..0x05]
// Unversioned(key):       [variant index, ...key...]
```

### Required Readings

**Essential Papers:**
| Paper | Authors | Year | Why It Matters |
|-------|---------|------|----------------|
| [A Critique of ANSI SQL Isolation Levels](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf) | Berenson et al. | 1995 | Defines SI, explains anomalies |
| [Generalized Isolation Level Definitions](http://pmg.csail.mit.edu/papers/icde00.pdf) | Adya, Liskov, O'Neil | 2000 | Formal SI definition |

**Blog Posts:**
| Post | Why It Matters |
|------|----------------|
| [How Postgres Makes Transactions Atomic](https://brandur.org/postgres-atomicity) | Practical MVCC implementation insights |
| [Jepsen: Consistency Models](https://jepsen.io/consistency) | Visual guide to isolation levels |
| [CockroachDB Transaction Layer](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html) | Production MVCC design |

**ToyDB Source Study:**
| File | What to Learn |
|------|---------------|
| `src/storage/mvcc.rs` | Full MVCC implementation |
| `src/storage/testscripts/mvcc/` | Test cases showing expected behavior |

### Implementation Assignment

**Week 7: Version Storage**
1. Implement MVCC key encoding using your Keycode package
2. Build version storage: `Set(key, version, value)`, `Get(key, version)`
3. Implement `NextVersion()` to allocate transaction IDs atomically
4. Implement unversioned metadata keys (`GetUnversioned`/`SetUnversioned`)

**Week 8: Active Transaction Tracking**
1. Implement `Begin()`: allocate version, write TxnActive marker, persist active-set snapshot if non-empty
2. Implement `GetActiveTransactions()`: scan TxnActive prefix
3. Build visibility check: a version V is visible to transaction T if:
   - V is not in T.active (active set snapshot)
   - For read-write: V <= T.version
   - For read-only: V < T.version

### Deliverable
MVCC storage layer supporting versioned reads/writes with active transaction tracking.

---

## Module 2.2: Transaction Lifecycle (Week 9-10)

### Learning Objectives
- Implement complete transaction lifecycle (begin, read, write, commit, rollback)
- Build lowest-transaction-id-wins conflict detection
- Support read-only and read-write transaction modes
- Support read-only as-of (time-travel) transactions

### What ToyDB Implements

**Transaction Modes:**
```go
type Mode int
const (
    ModeReadWrite Mode = iota  // Can read and write
    ModeReadOnly               // Read-only at latest version
    ModeReadOnlyAsOf           // Read-only at specific historical version
)
```

**Write Conflict Detection:**
- On write, scan the latest version for the key in the visible range
- If the latest version is not visible (newer or in active set), abort with serialization error
- Lowest transaction ID wins: earlier versions remain, later writers retry

**Rollback Support:**
- Every write records its key in `TxnWrite(version, key)`
- On rollback, scan our TxnWrite entries and delete corresponding Version entries

### Required Readings

**Papers:**
| Paper | Why It Matters |
|-------|----------------|
| [Serializable Snapshot Isolation in PostgreSQL](https://drkp.net/papers/ssi-vldb12.pdf) | Shows what ToyDB intentionally omits (SSI) |

**Blog Posts:**
| Post | Why It Matters |
|------|----------------|
| [Write Skew Explained](https://www.cockroachlabs.com/blog/what-write-skew-looks-like/) | Understanding SI's limitation |

**Books:**
| Book | Chapters | Why It Matters |
|------|----------|----------------|
| *Designing Data-Intensive Applications* (Kleppmann) | Ch. 7: Transactions | Best explanation of isolation levels |
| *Database Internals* (Petrov) | Ch. 5: Transaction Processing | MVCC implementation patterns |

### Implementation Assignment

**Week 9: Write Path**
1. Implement `Transaction.Set(key, value)`:
    - Check we're in ReadWrite mode
    - Write `Version(key, txn_version) = value`
    - Write `TxnWrite(txn_version, key)` for rollback tracking
2. Implement `Transaction.Delete(key)`:
    - Same as Set but with tombstone value
3. Implement `BeginReadOnly()` and `BeginAsOf(version)` without incrementing NextVersion

**Week 10: Commit, Rollback, and Resume**
1. Implement `Transaction.Commit()`:
    - Delete TxnWrite records and remove TxnActive marker
    - Do not flush here; rely on Raft log for durability
2. Implement `Transaction.Rollback()`:
    - Scan our TxnWrite entries
    - Delete corresponding Version entries and TxnWrite records
    - Delete TxnActive marker (leave TxnActiveSnapshot for time-travel)
3. Implement `MVCC.Resume(state TransactionState)`:
    - Reconstruct a `Transaction` from a serialized `TransactionState` (version, read_only flag, active set)
    - Verify the TxnActive marker still exists for read-write transactions (error if not)
    - This is critical for Raft integration: when the SQL engine receives a replicated command, it resumes the transaction that was started on the leader node

### Deliverable
Full transaction support with concurrent transaction tests demonstrating conflict detection.

---

## Module 2.3: MVCC Read Path + Time Travel (Week 11)

### Learning Objectives
- Implement point-in-time reads (scan for correct version)
- Build time-travel query support
- Understand why ToyDB skips garbage collection

### What ToyDB Implements

**Version Visibility Algorithm:**
```go
func (t *Transaction) Get(key []byte) ([]byte, error) {
    // Scan versions of this key from newest to oldest
    // Find first version V where:
    //   V is visible per snapshot rules
    //   - V not in active set
    //   - read-write: V <= t.version
    //   - read-only: V < t.version
    
    // Start scan at Version(key, t.version)
    // End scan at Version(key, 0)
    // Return first visible version's value
}
```

**Time Travel Queries:**
ToyDB supports `BEGIN TRANSACTION READ ONLY AS OF SYSTEM TIME <version>` to read historical snapshots. Each read-write transaction stores its active-set snapshot at that version, which time-travel queries restore for correct visibility.

**No Garbage Collection:**
ToyDB intentionally keeps all versions forever. This is an educational simplification—production databases would need GC to reclaim space from old versions invisible to all transactions.

### Required Readings

**Papers:**
| Paper | Why It Matters |
|-------|----------------|
| [An Empirical Evaluation of In-Memory MVCC](https://db.cs.cmu.edu/papers/2017/p781-wu.pdf) | Wu et al., CMU - MVCC implementation tradeoffs |

**Reference:**
| Resource | Why It Matters |
|----------|----------------|
| [PostgreSQL MVCC docs](https://www.postgresql.org/docs/current/mvcc.html) | Production visibility rules |

### Implementation Assignment

**Week 11:**
1. Implement `Transaction.Get(key)` with version visibility scan
2. Implement `Transaction.Scan(start, end)` returning latest visible versions only
3. Add read-only `AS OF` support using stored active-set snapshots
4. Write tests: concurrent readers see consistent snapshots, time-travel returns historical data

### Deliverable
Complete MVCC layer with snapshot reads, time-travel support, passing isolation tests.

---

# PHASE 3: RAFT CONSENSUS (Weeks 12-17)

## Module 3.1: Raft Fundamentals + Leader Election (Week 12-13)

### Learning Objectives
- Understand distributed consensus and why it's hard
- Implement Raft leader election with randomized timeouts
- Build the state machine abstraction

### What ToyDB Implements

**Raft Features (Implemented):**
- Leader election with term-based voting
- Log replication with consistency checks
- Quorum-based linearizable reads

**Raft Features (NOT Implemented):**
- Log compaction/snapshots (new nodes replay entire log)
- Log truncation (entire log retained)
- Dynamic membership changes (requires cluster restart)
- Leader leases (every read hits quorum)
- Pre-vote / check-quorum extensions
- Request retries and reject hints (simplified protocol)

**State Machine Interface:**
```go
type State interface {
    // GetAppliedIndex returns last applied log index
    GetAppliedIndex() uint64

    // Apply executes a write command entry, returns result
    Apply(entry Entry) ([]byte, error)

    // Read executes a read-only query
    Read(query []byte) ([]byte, error)
}
```

### Required Readings

**Essential Papers:**
| Paper | Why It Matters |
|-------|----------------|
| [In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf) | The Raft paper - READ MULTIPLE TIMES |
| [Raft Refloated](https://www.cl.cam.ac.uk/~ms705/pub/papers/2015-osr-raft.pdf) | Critical analysis of Raft subtleties |

**Supplementary Papers:**
| Paper | Why It Matters |
|-------|----------------|
| [Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) | Lamport's original—helps appreciate Raft's clarity |
| [FLP Impossibility](https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf) | Why consensus is fundamentally hard |

**Essential Guides:**
| Resource | Why It Matters |
|----------|----------------|
| [Students' Guide to Raft](https://thesquareplanet.com/blog/students-guide-to-raft/) | Common implementation pitfalls |
| [Raft Visualization](https://raft.github.io/) | Interactive demo of election/replication |

**Go Reference Implementations:**
| Project | What to Study |
|---------|---------------|
| [etcd/raft](https://github.com/etcd-io/raft) | Production Raft—study the `Ready` pattern |
| [hashicorp/raft](https://github.com/hashicorp/raft) | Alternative design with different tradeoffs |

### Implementation Assignment

**Week 12: State Machine + Node Structure**
1. Define `State` interface and message types:
    ```go
    type Envelope struct {
        Term uint64
        From NodeID
        To   NodeID
        Msg  Message
    }

    type Message enum {
        Campaign { lastIndex, lastTerm uint64 }
        CampaignResponse { vote bool }
        Heartbeat { lastIndex, commitIndex, readSeq uint64 }
        HeartbeatResponse { matchIndex, readSeq uint64 }
        Append { baseIndex, baseTerm uint64, entries []Entry }
        AppendResponse { matchIndex, rejectIndex uint64 }
        Read { seq uint64 }
        ReadResponse { seq uint64 }
        ClientRequest { id UUID, request Request }
        ClientResponse { id UUID, response Result<Response> }
    }
    ```
2. Implement node state: `Follower`, `Candidate`, `Leader`
3. Build persistent state: `currentTerm`, `votedFor`, `log[]`

**Week 13: Leader Election**
1. Implement election timeout (randomized 10-20 ticks at 100ms/tick)
2. Implement `Campaign` RPC: request votes from all peers
3. Implement `CampaignResponse`: vote if candidate's log is up-to-date
4. Implement leader transition on majority votes
5. Send heartbeats every 4 ticks and append a no-op entry on leadership change
6. Write election tests: single leader elected, elections after failure

### Deliverable
Raft node that reliably elects a single leader, handles leader failures.

---

## Module 3.2: Log Replication (Week 14-15)

### Learning Objectives
- Implement Append RPC with consistency checks and probing
- Build log matching and conflict resolution
- Understand commit index advancement

### What ToyDB Implements

**Log Entry Structure:**
```go
type Entry struct {
    Index   uint64  // Position in log (1-indexed)
    Term    uint64  // Term when entry was created
    Command []byte  // State machine command
}
```

**Replication Protocol:**
1. Leader appends entry to local log
2. Leader sends `Append` to all followers
3. Followers check consistency: `baseIndex` and `baseTerm` must match
4. If match: append entries, respond with `AppendResponse{matchIndex}`
5. If mismatch: respond with `AppendResponse{rejectIndex}`, leader probes earlier base indexes
6. When majority acknowledge: advance `commitIndex`, apply to state machine

### Required Readings

**Essential:**
| Resource | Why It Matters |
|----------|----------------|
| Raft Paper §5.3-5.4 | Log replication details |
| [TLA+ Raft Specification](https://github.com/ongardie/raft.tla) | Formal specification for edge cases |

**Debugging Help:**
| Resource | Why It Matters |
|----------|----------------|
| [Debugging Raft](https://blog.josejg.com/debugging-pretty/) | Structured logging for Raft debugging |

### Implementation Assignment

**Week 14: Basic Replication**
1. Implement `Append` message with entries, baseIndex, baseTerm
2. Implement follower log consistency check
3. Implement leader's `nextIndex` and `matchIndex` tracking
4. Build log conflict resolution (truncate and overwrite on mismatch)
5. Cap Append batches (ToyDB uses 100 entries per message)

**Week 15: Commitment + Application**
1. Implement `commitIndex` advancement when majority matches
2. Apply committed entries to state machine
3. Implement leader forwarding: followers proxy client requests to leader, abort on candidate/term change
4. Write replication tests: entries replicate, survive minority failures

### Deliverable
Log replication working across 3+ nodes, surviving single node failures.

---

## Module 3.3: Persistence + Linearizable Reads (Week 16-17)

### Learning Objectives
- Implement durable state for crash recovery
- Build quorum-based linearizable reads
- Integrate Raft with MVCC state machine

### What ToyDB Implements

**Persistent State (must survive restarts):**
- `currentTerm`: latest term seen
- `votedFor`: candidate voted for in current term
- `log[]`: all log entries
- `commitIndex`: last committed entry

**Linearizable Reads (Quorum-Based):**
ToyDB does NOT use leader leases. Instead:
1. Client sends read request to leader
2. Leader assigns read a sequence number
3. Leader sends `Read{seq}` to all followers (and piggybacks `read_seq` on heartbeats)
4. Leader waits for majority `ReadResponse{seq}` (confirming it's still leader)
5. Leader executes read against state machine
6. Leader returns result

This ensures reads see all committed writes, even if a new leader was elected.

### Required Readings

**Essential:**
| Resource | Why It Matters |
|----------|----------------|
| Raft Paper §8 | Client interaction and linearizable reads |
| [Raft Q&A on Read-Only](https://groups.google.com/g/raft-dev/c/9hWiMZDf6iE) | Discussion of read approaches |

**Go Patterns:**
| Resource | Why It Matters |
|----------|----------------|
| [etcd/raft ReadIndex](https://github.com/etcd-io/raft/blob/main/doc.go) | Production linearizable read implementation |

### Implementation Assignment

**Week 16: Persistence**
1. Implement persistent log storage using your BitCask engine
2. Save term/votedFor on every update
3. Implement crash recovery: reload state, rejoin cluster
4. Write crash tests: kill node mid-operation, verify recovery

**Week 17: Linearizable Reads + Integration**
1. Implement read index protocol:
    ```go
    func (n *Node) Read(query []byte) ([]byte, error) {
        // 1. Increment read sequence number
        // 2. Send Read{seq} to followers, wait for quorum
        // 3. Execute read on state machine
    }
    ```
2. Build Raft client with automatic leader discovery
3. Integrate MVCC as the Raft state machine
4. Write linearizability tests using [Porcupine](https://github.com/anishathalye/porcupine)

### Deliverable
Complete Raft implementation with linearizable reads, integrated with MVCC, passing Porcupine tests.

---

# PHASE 4: SQL ENGINE (Weeks 18-25)

## Module 4.1: Lexer + Parser (Week 18-20)

### Learning Objectives
- Build a hand-written lexer for SQL tokens
- Implement recursive descent parsing for statements
- Master precedence climbing for expressions

### What ToyDB Implements

**Supported Tokens:**
- Keywords: AS, ASC, AND, BEGIN, BOOL, BOOLEAN, BY, COMMIT, CREATE, CROSS, DEFAULT, DELETE, DESC, DOUBLE, DROP, EXISTS, EXPLAIN, FALSE, FLOAT, FROM, GROUP, HAVING, IF, INDEX, INFINITY, INNER, INSERT, INT, INTEGER, INTO, IS, JOIN, KEY, LEFT, LIKE, LIMIT, NAN, NOT, NULL, OF, OFFSET, ON, ONLY, OR, ORDER, OUTER, PRIMARY, READ, REFERENCES, RIGHT, ROLLBACK, SELECT, SET, STRING, SYSTEM, TABLE, TEXT, TIME, TRANSACTION, TRUE, UNIQUE, UPDATE, VALUES, VARCHAR, WHERE, WRITE
- Operators: +, -, *, /, %, ^ (power), ! (factorial), =, !=, <, >, <=, >=, LIKE, IS (NULL/NOT NULL/NAN/NOT NAN)
- Punctuation: (, ), ,, ;

**Expression Precedence (lowest to highest):**
1. OR
2. AND
3. NOT
4. =, !=, LIKE, IS
5. >, >=, <, <=
6. +, -
7. *, /, %
8. ^ (power, right-associative)
9. ! (factorial, postfix)
10. Unary +, - (prefix)

### Required Readings

**Parsing Techniques:**
| Resource | Why It Matters |
|----------|----------------|
| [Precedence Climbing](https://eli.thegreenplace.net/2012/08/02/parsing-expressions-by-precedence-climbing) | Eli Bendersky - exact algorithm ToyDB uses |
| [Crafting Interpreters: Parsing](https://craftinginterpreters.com/parsing-expressions.html) | Alternative Pratt parsing explanation |
| [Simple but Powerful Pratt Parsing](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html) | Rust perspective, applicable to Go |

**Go Lexer/Parser Patterns:**
| Resource | Why It Matters |
|----------|----------------|
| [Go text/scanner](https://pkg.go.dev/text/scanner) | Standard library scanner for inspiration |
| [InfluxDB SQL Parser](https://github.com/influxdata/influxql) | Production SQL parser in Go (parser.go, scanner.go at repo root) |

### Implementation Assignment

**Week 18: Lexer**
1. Implement `Lexer` struct with `NextToken() Token` method
2. Handle all SQL keywords, operators, identifiers, literals
3. Handle string literals with doubled single quotes only (no backslash escapes)
4. Handle numeric literals (integers, floats, scientific notation, NaN, Infinity)
5. Lowercase unquoted identifiers and preserve quoted identifiers

**Week 19: Statement Parser**
1. Implement recursive descent parser structure
2. Parse DDL: `CREATE TABLE name (col type, ...)`, `DROP TABLE [IF EXISTS] name`
3. Parse DML: `INSERT INTO ... VALUES ...`, `UPDATE ... SET ...`, `DELETE FROM ...`
4. Parse `SELECT ... FROM ... WHERE ...` (basic, no joins yet)

**Week 20: Expression Parser + Complex Statements**
1. Implement precedence climbing for expressions
2. Add JOIN parsing: `... JOIN table ON condition`
3. Add GROUP BY, HAVING, ORDER BY, LIMIT, OFFSET
4. Parse `IS NULL` / `IS NOT NULL` / `IS NAN` / `IS NOT NAN`
5. Parse transaction statements: BEGIN [TRANSACTION] [READ ONLY|READ WRITE] [AS OF SYSTEM TIME <txn_id>], COMMIT, ROLLBACK
6. Parse EXPLAIN prefix

### Deliverable
Complete SQL parser producing AST for all ToyDB-supported statements.

---

## Module 4.2: Query Planner + Optimizer (Week 21-23)

### Learning Objectives
- Convert AST to logical plan tree
- Implement heuristic optimization rules
- Build the schema catalog

### What ToyDB Implements

**Logical Plan Nodes:**
```go
type Plan interface {
    // CreateTable, DropTable, Insert, Update, Delete, Select
}

type Node interface {
    // All plan nodes implement this
}

type Scan struct {
    Table  string
    Alias  string
    Filter Expression  // Pushed-down predicate
}

type Filter struct {
    Source    Node
    Predicate Expression
}

type Projection struct {
    Source  Node
    Columns []Expression
}

type HashJoin struct {
    Left       Node
    Right      Node
    LeftColumn int
    RightColumn int
    Outer      bool
}

type NestedLoopJoin struct {
    Left     Node
    Right    Node
    Predicate Expression  // Join condition (optional)
    Outer    bool
}

type Aggregate struct {
    Source     Node
    GroupBy    []Expression
    Aggregates []AggregateExpr
}

type Order struct {
    Source Node
    Keys   []OrderKey  // (expr, ASC/DESC)
}

type Limit struct {
    Source Node
    Count  uint64
}

type Offset struct {
    Source Node
    Count  uint64
}

type IndexLookup struct {
    Table  Table
    Column int
    Values []Value
}

type KeyLookup struct {
    Table Table
    Keys  []Value
}

type Remap struct {
    Source  Node
    Targets []int // source -> target mapping, -1 to drop
}

type Values struct {
    Rows [][]Expression
}

type Nothing struct {
    Columns []Label
}
```

Note: Index lookups treat NULL and NaN values as equal to support `IS NULL` and `IS NAN` predicates.

**Optimization Rules (Heuristic, No Cost Model):**
1. **Constant folding**: `1 + 2` → `3`
2. **Predicate pushdown**: Move filters closer to scans (CNF decomposition)
3. **Index lookup**: Convert scans to KeyLookup/IndexLookup for equality predicates
4. **Hash join selection**: Use HashJoin for equi-joins, NestedLoop otherwise
5. **Short-circuit**: Remove nodes that can never emit rows (Nothing)

### Required Readings

**Query Processing:**
| Resource | Why It Matters |
|----------|----------------|
| [CMU 15-445 Query Processing](https://15445.courses.cs.cmu.edu/fall2022/notes/12-queryexecution1.pdf) | Andy Pavlo's lecture notes |
| [How Query Engines Work](https://howqueryengineswork.com/) | Free online book |

**Optimization:**
| Paper | Why It Matters |
|-------|----------------|
| [Access Path Selection in System R](https://courses.cs.duke.edu/compsci516/cps216/spring03/papers/selinger-etal-1979.pdf) | Classic cost-based optimizer |

**Note:** ToyDB uses heuristic optimization, not cost-based. The System R paper helps understand what a full optimizer would look like.

### Implementation Assignment

**Week 21: Catalog + Basic Planning + Schema Validation**
1. Implement schema catalog:
   ```go
   type Catalog interface {
       CreateTable(name string, columns []Column) error
       GetTable(name string) (*Table, error)
       DropTable(name string) error
       ListTables() []string
   }
   ```
2. Implement schema-level validation in `Table`:
    - `Table.Validate()`: verify primary key exists, column types valid, foreign key references exist, referenced columns are primary keys
    - `Table.ValidateRow(row)`: type-check values, enforce NOT NULL, enforce UNIQUE constraints, enforce foreign key referential integrity
3. Implement type checking during plan construction (verify operand types for expressions, column types for comparisons)
4. Convert SELECT AST to Scan → Filter → Projection plan
5. Implement name resolution with a Scope (bind columns to indexes, handle aliases)
6. Build Values and Remap nodes for INSERT column mapping

**Week 22: Join Planning**
1. Implement join order from AST (ToyDB uses AST order, no reordering)
2. Build NestedLoopJoin by default; optimizer may replace with HashJoin
3. Handle LEFT/RIGHT/INNER/CROSS join types
4. Implement ON clause as join predicate

**Week 23: Optimization Rules**
1. Implement constant folding in expressions
2. Implement predicate pushdown through joins
3. Implement KeyLookup/IndexLookup conversion for equality predicates
4. Implement short-circuit to Nothing where possible
5. Add EXPLAIN output: print plan tree with node types

### Deliverable
Query planner producing optimized logical plans, EXPLAIN showing plan structure.

---

## Module 4.3: Query Executor (Week 24-25)

### Learning Objectives
- Implement the Volcano/iterator execution model
- Build hash join and nested loop join executors
- Implement aggregation with GROUP BY/HAVING

### What ToyDB Implements

**Iterator Model:**
```go
type Executor interface {
    // Open initializes the executor
    Open() error
    
    // Next returns the next row, or nil when exhausted
    Next() (Row, error)
    
    // Close releases resources
    Close() error
}
```

ToyDB buffers all result rows in the session before sending them over the network (no streaming).

**Session Behavior:**
- Explicit transactions via BEGIN/COMMIT/ROLLBACK
- Implicit transactions per statement when no explicit txn is open
- Commit on success, rollback on error
- Roll back any open transaction on session close

**Join Implementations:**
- **HashJoin**: Build hash table from right side, probe with left side
- **NestedLoopJoin**: For each left row, scan all right rows

**Additional Nodes:**
- **IndexLookup**: Use secondary index to get primary keys, then fetch rows
- **KeyLookup**: Fetch rows by primary key
- **Offset**: Skip N rows
- **Values**: Emit constant rows
- **Remap**: Reorder/drop columns
- **Nothing**: Emit zero rows

**Aggregate Functions:**
- COUNT(*), COUNT(col)
- SUM, AVG, MIN, MAX

### Required Readings

**Execution Models:**
| Paper | Why It Matters |
|-------|----------------|
| [Volcano: Query Evaluation System](https://cs.uwaterloo.ca/~david/cs848/background/volcano.pdf) | Graefe - iterator model origin |

**Join Algorithms:**
| Resource | Why It Matters |
|----------|----------------|
| [CMU Join Algorithms](https://15445.courses.cs.cmu.edu/fall2022/notes/11-joins.pdf) | Comprehensive join algorithm overview |

### Implementation Assignment

**Week 24: Basic Executors**
1. Implement `ScanExecutor`: iterate over table rows
2. Implement `FilterExecutor`: evaluate predicate, treat NULL as non-matching
3. Implement `ProjectionExecutor`: evaluate column expressions
4. Implement `LimitExecutor`: count rows, stop at limit
5. Implement `OffsetExecutor`: skip rows

**Week 25: Joins + Aggregates**
1. Implement `HashJoinExecutor`:
    - Build phase: hash right side into `map[hashKey][]Row`
    - Probe phase: for each left row, lookup and emit matches
2. Implement `NestedLoopJoinExecutor` for non-equi joins
3. Implement `AggregateExecutor`:
    - No GROUP BY: single accumulator
    - With GROUP BY: `map[groupKey]accumulators`
4. Implement `OrderExecutor`: collect all rows, sort, iterate
5. Implement `IndexLookupExecutor` and `KeyLookupExecutor`
6. Implement `ValuesExecutor`, `RemapExecutor`, `NothingExecutor`
7. Wire executor to MVCC transactions for actual data access
8. Implement secondary index maintenance during write operations:
    - On INSERT: create index entries for all indexed columns
    - On UPDATE: delete old index entries and create new ones for changed indexed columns
    - On DELETE: remove index entries for all indexed columns
    - Enforce foreign key referential integrity on DELETE (error if other tables reference the row)
9. Implement `<>` as alternate not-equal operator (alias for `!=`)

### Deliverable
Complete query executor running all ToyDB SQL features against MVCC storage.

---

# PHASE 5: INTEGRATION (Week 26)

## Module 5.1: Server + Client Protocol

### Learning Objectives
- Implement TCP server with session management
- Implement TCP protocol with Bincode-style messages (no extra framing)
- Handle transaction state across requests
- Mirror ToyDB's Request/Response message flow

### What ToyDB Implements

**Protocol:**
- TCP connection with Bincode-encoded Request/Response messages (no extra framing)
- Binary serialization (Bincode in Rust; use encoding/gob or msgpack in Go)
- Request types:
    - `Execute(string)`: execute a SQL statement
    - `GetTable(string)`: retrieve table schema by name
    - `ListTables`: list all table names
    - `Status`: get server/cluster status
- Response types:
    - `Execute(StatementResult)`: result of SQL execution
    - `Row(Row)`: streaming row (used internally)
    - `GetTable(Table)`: table schema
    - `ListTables([]string)`: table name list
    - `Status(Status)`: server status containing node ID, Raft status, and MVCC status
- `StatementResult` variants: `Begin{txn_state}`, `Commit{version}`, `Rollback{version}`, `Explain{plan}`, `CreateTable{name}`, `DropTable{name, existed}`, `Delete{count}`, `Insert{count}`, `Update{count}`, `Select{columns, rows}`
- Session tracks: active transaction ID, transaction mode (client updates its `txn` field based on Begin/Commit/Rollback responses)

**Server Features:**
- Per-connection session handling with transaction state
- Leader forwarding for Raft writes
- OS threads + channels (no async runtime)
- Separate TCP listeners for SQL and Raft traffic

### Required Readings

| Resource | Why It Matters |
|----------|----------------|
| [Go net package](https://pkg.go.dev/net) | TCP server implementation |
| [Go encoding/gob](https://pkg.go.dev/encoding/gob) | Binary serialization |
| [MessagePack for Go](https://github.com/vmihailenco/msgpack) | Alternative serialization |

### Implementation Assignment

1. Implement TCP server accepting connections
2. Implement session state tracking active transaction
3. Build request/response message types
4. Implement SQL execution pipeline: parse → plan → optimize → execute
5. Return serialization errors and document client-side retry behavior
6. Build simple CLI client for testing

### Deliverable
Working client-server database accepting SQL queries over TCP.

---

## Module 5.2: Testing + Verification

### Learning Objectives
- Build comprehensive test infrastructure
- Verify correctness with linearizability testing
- Create golden test scripts

### What ToyDB Implements

**Goldenscript Testing:**
ToyDB uses a custom golden master framework where test files contain commands and expected outputs. Tests fail if actual output differs.

**Test Categories:**
- Unit tests per module
- MVCC isolation tests (concurrent transactions)
- Raft consensus tests (leader election, log replication)
- SQL correctness tests (query results)
- Cluster integration tests (multi-node)

### Required Readings

| Resource | Why It Matters |
|----------|----------------|
| [Porcupine](https://github.com/anishathalye/porcupine) | Linearizability testing for Go |
| [Testing Distributed Systems](https://asatarin.github.io/testing-distributed-systems/) | Curated resource list |
| [Jepsen](https://jepsen.io/) | Inspiration for chaos testing |

### Implementation Assignment

1. Set up table-driven unit tests for all modules
2. Implement integration tests for multi-node cluster
3. Write MVCC isolation tests:
   - Concurrent readers see consistent snapshots
   - Write-write conflicts detected
   - Committed writes visible to new transactions
4. Run Porcupine linearizability tests against Raft layer
5. Create SQL golden tests for all supported queries

### Deliverable
Comprehensive test suite with >80% code coverage, passing linearizability tests.

---

# APPENDIX A: Complete ToyDB Feature Checklist

Use this checklist to verify feature parity:

## Storage
- [ ] Engine interface with Get/Set/Delete/Scan/ScanPrefix/Flush
- [ ] Reverse scans (double-ended iterator)
- [ ] Memory engine (BTreeMap-based)
- [ ] BitCask engine (log + hash index)
- [ ] Log compaction
- [ ] Keycode order-preserving encoding

## MVCC
- [ ] Snapshot Isolation
- [ ] Transaction modes: ReadWrite, ReadOnly, ReadOnlyAsOf
- [ ] Version visibility rules
- [ ] Write conflict detection (lowest-txn-id wins)
- [ ] Rollback support
- [ ] Transaction resume from serialized state (for Raft integration)
- [ ] Time-travel reads (AS OF SYSTEM TIME) with active-set snapshots
- [ ] TxnActiveSnapshot and Unversioned keys
- [ ] NO garbage collection (intentional)

## Raft
- [ ] Leader election with randomized timeout
- [ ] Log replication with consistency checks
- [ ] Commit index advancement
- [ ] Persistent term/votedFor/log
- [ ] Crash recovery
- [ ] Linearizable reads (quorum-based)
- [ ] NO snapshots (intentional)
- [ ] NO membership changes (intentional)

## SQL Parser
- [ ] Hand-written lexer
- [ ] Recursive descent for statements
- [ ] Precedence climbing for expressions
- [ ] All operators: +, -, *, /, %, ^, !, AND, OR, NOT, comparisons, LIKE, IS
- [ ] CREATE TABLE / DROP TABLE (with IF EXISTS)
- [ ] INSERT / UPDATE / DELETE
- [ ] SELECT with all clauses
- [ ] BEGIN [TRANSACTION] [READ ONLY|READ WRITE] [AS OF SYSTEM TIME], COMMIT, ROLLBACK
- [ ] EXPLAIN
- [ ] Column constraints: PRIMARY KEY, NOT NULL, DEFAULT, UNIQUE, INDEX, REFERENCES
- [ ] IS NULL / IS NOT NULL / IS NAN / IS NOT NAN

## SQL Planner
- [ ] Schema catalog
- [ ] Name resolution
- [ ] Type checking
- [ ] Plan roots: CreateTable, DropTable, Insert, Update, Delete, Select
- [ ] Logical plan nodes (Scan, Filter, Projection, Aggregate, Order, Limit, Offset)
- [ ] Join nodes (NestedLoopJoin, HashJoin)
- [ ] Lookup nodes (KeyLookup, IndexLookup)
- [ ] Plan helpers (Values, Remap, Nothing)
- [ ] HashJoin for equi-joins
- [ ] NestedLoopJoin for non-equi joins

## SQL Optimizer
- [ ] Constant folding
- [ ] Predicate pushdown (CNF)
- [ ] KeyLookup/IndexLookup for equality predicates
- [ ] HashJoin selection for equi-joins
- [ ] Short-circuit to Nothing
- [ ] NO cost-based optimization (intentional)
- [ ] NO join reordering (intentional)

## SQL Executor
- [ ] Iterator/Volcano model
- [ ] All plan node executors (Scan, Filter, Projection, Aggregate, Order, Limit, Offset)
- [ ] Join executors (NestedLoopJoin, HashJoin)
- [ ] Lookup executors (KeyLookup, IndexLookup)
- [ ] Values/Remap/Nothing executors
- [ ] Aggregates: COUNT, SUM, AVG, MIN, MAX
- [ ] Data types: BOOLEAN, INTEGER, FLOAT, STRING
- [ ] NULL handling
- [ ] NaN/Infinity handling
- [ ] sqrt() function (only built-in function)
- [ ] Secondary index lookups
- [ ] Secondary index maintenance (create/update/delete entries during DML)
- [ ] Foreign key / referential integrity enforcement (schema validation + delete checks)
- [ ] Type checking during plan construction
- [ ] SELECT results buffered for client responses

## Server
- [ ] TCP with Bincode (no framing)
- [ ] Binary serialization
- [ ] Request types: Execute, GetTable, ListTables, Status
- [ ] Response types: Execute, Row, GetTable, ListTables, Status
- [ ] StatementResult variants (Begin, Commit, Rollback, Explain, CreateTable, DropTable, Delete, Insert, Update, Select)
- [ ] Session transaction state (client-side tracking from Begin/Commit/Rollback responses)
- [ ] Leader forwarding
- [ ] Separate Raft and SQL listeners
- [ ] Threaded routing (goroutines + channels in Go)

---

# APPENDIX B: Recommended Go Project Structure

```
toydb-go/
├── cmd/
│   ├── toydb/          # Server binary
│   └── toysql/         # CLI client
├── internal/
│   ├── encoding/
│   │   └── keycode/    # Order-preserving encoding
│   ├── storage/
│   │   ├── engine.go   # Engine interface
│   │   ├── memory.go   # Memory engine
│   │   ├── bitcask.go  # BitCask engine
│   │   └── mvcc.go     # MVCC transaction layer
│   ├── raft/
│   │   ├── node.go     # Raft node
│   │   ├── log.go      # Log storage
│   │   ├── state.go    # State machine interface
│   │   └── transport.go # RPC transport
│   ├── sql/
│   │   ├── lexer/      # Tokenizer
│   │   ├── parser/     # AST construction
│   │   ├── planner/    # Logical planning
│   │   ├── optimizer/  # Heuristic rules
│   │   ├── executor/   # Iterator execution
│   │   └── catalog/    # Schema management
│   └── server/
│       ├── server.go   # TCP server
│       ├── session.go  # Client sessions
│       └── client.go   # Client library
├── testdata/           # Golden test files
├── go.mod
└── README.md
```

---

# APPENDIX C: Essential Go Libraries

| Category | Library | Purpose |
|----------|---------|---------|
| Testing | `github.com/stretchr/testify` | Assertions and mocking |
| Testing | `github.com/anishathalye/porcupine` | Linearizability checking |
| Serialization | `github.com/vmihailenco/msgpack/v5` | Binary encoding |
| Sorted Map | `github.com/google/btree` | In-memory B-tree (archived Oct 2025 — still functional, consider alternatives) |
| Logging | `log/slog` (stdlib) | Structured logging |
| CLI | `github.com/spf13/cobra` | Command-line interface |
| Networking | `google.golang.org/grpc` | RPC (optional, can use net/rpc) |

---

# APPENDIX D: Week-by-Week Schedule Summary

| Week | Module | Deliverable |
|------|--------|-------------|
| 1-2 | Storage Abstraction | Engine interface + Memory engine |
| 3-4 | BitCask | Persistent log-structured storage |
| 5-6 | Keycode | Order-preserving encoding |
| 7-8 | MVCC Fundamentals | Version storage + active transactions |
| 9-10 | Transaction Lifecycle | Commit/rollback with conflict detection |
| 11 | MVCC Reads | Snapshot reads + time travel |
| 12-13 | Raft Election | Leader election working |
| 14-15 | Raft Replication | Log replication across nodes |
| 16-17 | Raft Persistence | Crash recovery + linearizable reads |
| 18-20 | SQL Parser | Complete SQL AST |
| 21-23 | SQL Planner | Logical plans + optimization |
| 24-25 | SQL Executor | Full query execution |
| 26 | Integration | Server + testing + documentation |

**Total: 26 weeks / 6 months**

---

# APPENDIX E: Complete ToyDB References (from Erik Grinaker)

These are the exact references from [ToyDB's docs/references.md](https://github.com/erikgrinaker/toydb/blob/main/docs/references.md), which Erik Grinaker used while building ToyDB. All are incorporated into this curriculum.

## Foundational Resources

**Video Courses:**
| Resource | Author | Notes |
|----------|--------|-------|
| 🎥 [CMU 15-445 Intro to Database Systems](https://15445.courses.cs.cmu.edu/) | Andy Pavlo (2019) | "Absolutely fantastic introduction to database internals" |
| 🎥 [CMU 15-721 Advanced Database Systems](https://15721.courses.cs.cmu.edu/) | Andy Pavlo (2020) | Query optimization, execution engines |

**Books:**
| Resource | Author | Notes |
|----------|--------|-------|
| 📖 [Designing Data-Intensive Applications](https://dataintensive.net/) | Martin Kleppmann (2017) | Excellent overview of database technologies and concepts |
| 📖 [Database Internals](https://www.databass.dev/) | Alex Petrov (2019) | In-depth on storage engines and distributed systems algorithms |

## Raft Consensus

**Primary Sources:**
| Resource | Author | Notes |
|----------|--------|-------|
| 📄 [In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf) | Diego Ongaro, John Ousterhout (2014) | The Raft paper — READ MULTIPLE TIMES |
| 🎥 [Designing for Understandability: The Raft Consensus Algorithm](https://www.youtube.com/watch?v=vYp4LYbnnW8) | John Ousterhout (2016) | Raft talk by Ongaro's advisor |

**Implementation Guides:**
| Resource | Author | Notes |
|----------|--------|-------|
| 💬 [Students' Guide to Raft](https://thesquareplanet.com/blog/students-guide-to-raft/) | Jon Gjengset | "Very helpful in drawing attention to subtle pitfalls" |

## Transactions and Isolation

**Consistency Models:**
| Resource | Author | Notes |
|----------|--------|-------|
| 💬 [Jepsen Consistency Models](https://jepsen.io/consistency) | Kyle Kingsbury | "Excellent overview... very helpful in making sense of the jungle of overlapping and ill-defined terms" |

**Classic Papers:**
| Resource | Author | Notes |
|----------|--------|-------|
| 📄 [A Critique of ANSI SQL Isolation Levels](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf) | Berenson et al. (1995) | Defines Snapshot Isolation, explains anomalies |
| 📄 [Generalized Isolation Level Definitions](http://pmg.csail.mit.edu/papers/icde00.pdf) | Adya, Liskov, O'Neil (2000) | Formal SI definition |

**MVCC Implementation:**
| Resource | Author | Notes |
|----------|--------|-------|
| 💬 [Implementing Your Own Transactions with MVCC](https://elliotchance.medium.com/implementing-your-own-transactions-with-mvcc-bba11cab8e70) | Elliot Chance (2015) | "Found blog posts to be the most helpful" |
| 💬 [How Postgres Makes Transactions Atomic](https://brandur.org/postgres-atomicity) | Brandur Leach (2017) | Practical MVCC insights |

## SQL Parsing

| Resource | Author | Notes |
|----------|--------|-------|
| 💬 [Parsing Expressions by Precedence Climbing](https://eli.thegreenplace.net/2012/08/02/parsing-expressions-by-precedence-climbing) | Eli Bendersky (2012) | "The algorithm I found the most elegant" — ToyDB's parser is inspired by this |

## Additional References from Erik's Reading List

Erik maintains an extensive [readings repository](https://github.com/erikgrinaker/readings) with additional materials organized by topic. Key additions for this curriculum:

**Storage Engines:**
| Resource | Notes |
|----------|-------|
| 📄 [Bitcask: A Log-Structured Hash Table](https://riak.com/assets/bitcask-intro.pdf) | Original BitCask design document |
| 📄 [The Log-Structured Merge-Tree](https://www.cs.umb.edu/~poneil/lsmtree.pdf) | O'Neil et al. (1996) — for comparison |

**Transactions:**
| Resource | Notes |
|----------|-------|
| 📄 [An Empirical Evaluation of In-Memory MVCC](https://db.cs.cmu.edu/papers/2017/p781-wu.pdf) | Wu et al., CMU — MVCC implementation tradeoffs |
| 📄 [Serializable Snapshot Isolation in PostgreSQL](https://drkp.net/papers/ssi-vldb12.pdf) | What ToyDB intentionally omits |

**Distributed Systems:**
| Resource | Notes |
|----------|-------|
| 📄 [Spanner: Google's Globally Distributed Database](https://research.google/pubs/pub39966/) | Production distributed SQL reference |
| 📄 [Calvin: Fast Distributed Transactions](https://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf) | Alternative transaction coordination |

---

*This curriculum achieves exact feature parity with erikgrinaker/toydb while adapting the implementation for Go idioms. Following this progression builds understanding layer by layer, culminating in a working distributed SQL database.*
