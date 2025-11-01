# MemcachedSaver Design Evaluation

## Overview

This document evaluates the design and implementation of `MemcachedSaver` and `AsyncMemcachedSaver` classes following the same pattern as `ValkeySaver` and `AsyncValkeySaver` but using `pymemcache` for AWS ElastiCache Memcached support.

## Executive Summary

**⚠️ NOT RECOMMENDED** for implementation due to critical architectural limitations that fundamentally conflict with checkpoint saver requirements.

### Critical Blocker Issues

1. **❌ No LIST Data Structure**: Memcached doesn't support Redis LIST operations (`lpush`, `lrange`) needed for thread checkpoint ordering
2. **❌ No KEYS Command**: Cannot enumerate keys by pattern, making `delete_thread()` impossible
3. **❌ No Atomic Transactions**: Cannot ensure checkpoint consistency without pipeline/transactions
4. **❌ Weak Durability**: Memcached is designed for volatile caching, not persistent storage

**Recommendation**: Use ValkeySaver/Redis for checkpoint persistence. Memcached is appropriate for MemcachedCache but NOT for checkpoint savers.

---

## Current Status: ValkeySaver Pattern

### Architecture

**Base Class Hierarchy**:
```
BaseCheckpointSaver[V]  (from langgraph.checkpoint.base)
    ↓
BaseValkeySaver  (shared logic for sync/async)
    ↓
ValkeySaver (sync)  |  AsyncValkeySaver (async)
```

### Key Data Structures in Valkey/Redis

ValkeySaver relies heavily on Redis-specific data structures:

#### 1. **Thread Checkpoint List** (Redis LIST)
```python
# Key: thread:{thread_id}:{checkpoint_ns}
# Type: LIST
# Purpose: Ordered list of checkpoint IDs for a thread (newest first)

thread:abc123:main = ["ckpt-003", "ckpt-002", "ckpt-001"]
                      ↑ Most recent (lpush adds to front)
```

**Operations**:
- `lpush(thread_key, checkpoint_id)` - Add new checkpoint to front
- `lrange(thread_key, 0, 0)` - Get latest checkpoint
- `lrange(thread_key, 0, -1)` - Get all checkpoints (for list/delete)
- `lrange(thread_key, start, end)` - Paginated listing

#### 2. **Checkpoint Data** (Redis STRING with JSON)
```python
# Key: checkpoint:{thread_id}:{checkpoint_ns}:{checkpoint_id}
# Type: STRING (JSON serialized)
# Contains: checkpoint dict, metadata, parent_config

checkpoint:abc123:main:ckpt-001 = {
    "checkpoint": {...},  # Serialized checkpoint state
    "metadata": {...},    # Checkpoint metadata
    "parent_config": {...},  # Parent checkpoint reference
    "version": "2024-01-01-...",
}
```

#### 3. **Writes Data** (Redis STRING with JSON array)
```python
# Key: writes:{thread_id}:{checkpoint_ns}:{checkpoint_id}
# Type: STRING (JSON array)
# Contains: Intermediate writes associated with checkpoint

writes:abc123:main:ckpt-001 = [
    {"channel": "messages", "value": "...", "task_id": "task-1"},
    {"channel": "state", "value": "...", "task_id": "task-1"},
]
```

### Core Operations in ValkeySaver

#### 1. **get_tuple()** - Retrieve Latest or Specific Checkpoint
```python
def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None:
    if checkpoint_id:
        # Get specific checkpoint directly
        checkpoint_data = client.get(checkpoint_key)
        writes_data = client.get(writes_key)
    else:
        # Get LATEST checkpoint
        checkpoint_ids = client.lrange(thread_key, 0, 0)  # ← LIST operation
        checkpoint_id = checkpoint_ids[0]
        checkpoint_data = client.get(checkpoint_key)
        writes_data = client.get(writes_key)
```

**Memcached Impact**: ❌ **Cannot get "latest"** - No LIST to track ordering

#### 2. **list()** - Iterate Through Checkpoints (Newest First)
```python
def list(self, config, filter=None, before=None, limit=None):
    # Get ordered list of checkpoint IDs
    all_ids = client.lrange(thread_key, 0, -1)  # ← LIST operation

    # Apply 'before' filter
    if before_id:
        before_idx = find_index(all_ids, before_id)  # ← Requires ordering
        start_idx = before_idx + 1

    # Batch fetch checkpoints in order
    checkpoint_ids = client.lrange(thread_key, start_idx, end_idx)  # ← Pagination
    for checkpoint_id in checkpoint_ids:
        checkpoint_data = client.get(checkpoint_key)
        yield deserialize(checkpoint_data)
```

**Memcached Impact**: ❌ **Cannot maintain order** - No LIST for ordered iteration

#### 3. **put()** - Save Checkpoint Atomically
```python
def put(self, config, checkpoint, metadata, new_versions):
    pipe = client.pipeline()  # ← Atomic transaction
    pipe.set(checkpoint_key, serialize(checkpoint))
    pipe.lpush(thread_key, checkpoint_id)  # ← LIST operation
    if ttl:
        pipe.expire(checkpoint_key, ttl)
        pipe.expire(thread_key, ttl)
    pipe.execute()  # ← Atomic execution
```

**Memcached Impact**: ❌ **No atomicity** - Cannot guarantee consistency

#### 4. **delete_thread()** - Delete All Thread Data
```python
def delete_thread(self, thread_id: str):
    # Find all thread namespaces
    thread_keys = client.keys(f"thread:{thread_id}:*")  # ← Pattern matching

    for thread_key in thread_keys:
        # Get all checkpoint IDs
        checkpoint_ids = client.lrange(thread_key, 0, -1)  # ← LIST operation

        # Collect all keys to delete
        for checkpoint_id in checkpoint_ids:
            keys_to_delete.extend([
                checkpoint_key,
                writes_key
            ])

    # Batch delete
    client.delete(*keys_to_delete)
```

**Memcached Impact**: ❌ **Cannot enumerate keys** - No KEYS command

#### 5. **put_writes()** - Append Writes to Checkpoint
```python
def put_writes(self, config, writes, task_id):
    pipe = client.pipeline()
    pipe.get(writes_key)
    results = pipe.execute()

    existing_writes = deserialize(results[0]) or []
    existing_writes.extend(new_writes)

    pipe = client.pipeline()
    pipe.set(writes_key, serialize(existing_writes))
    pipe.expire(writes_key, ttl)
    pipe.execute()  # ← Atomic read-modify-write
```

**Memcached Impact**: ⚠️ **Race condition** - No atomic read-modify-write

---

## Memcached Limitations Analysis

### Missing Features Comparison

| Feature | Valkey/Redis | Memcached | Impact on Checkpoint Saver |
|---------|--------------|-----------|----------------------------|
| **LIST data type** | ✅ lpush, lrange, lpop | ❌ None | 🔴 **CRITICAL** - Cannot track checkpoint order |
| **KEYS pattern** | ✅ `keys thread:*` | ❌ None | 🔴 **CRITICAL** - Cannot enumerate for delete |
| **Pipelining** | ✅ Multi-cmd transactions | ❌ Limited | 🔴 **CRITICAL** - No atomicity guarantees |
| **TTL per key** | ✅ Individual TTL | ✅ Individual TTL | ✅ Compatible |
| **Simple GET/SET** | ✅ O(1) | ✅ O(1) | ✅ Compatible |
| **Persistence** | ✅ RDB/AOF | ❌ Volatile only | 🟡 **WARNING** - Data loss on restart |
| **Replication** | ✅ Primary-replica | ❌ No replication | 🟡 **WARNING** - No HA |

### Why These Limitations Are Critical

#### 1. **No LIST → Cannot Track Checkpoint Order** 🔴

**Problem**: Checkpoints MUST be retrievable in chronological order (newest first)

**ValkeySaver approach**:
```python
# Checkpoints are ordered by insertion time
lpush(thread_key, "ckpt-003")  # [ckpt-003]
lpush(thread_key, "ckpt-002")  # [ckpt-002, ckpt-003]
lpush(thread_key, "ckpt-001")  # [ckpt-001, ckpt-002, ckpt-003]

# Get latest
latest = lrange(thread_key, 0, 0)[0]  # Returns "ckpt-001"
```

**Memcached alternatives**:

**❌ Alternative 1: Store as JSON array in single key**
```python
# Problem: Race conditions on concurrent updates
thread_data = get("thread:abc123")  # Thread 1 reads
thread_data = get("thread:abc123")  # Thread 2 reads
thread_data.append("ckpt-A")        # Thread 1 modifies
thread_data.append("ckpt-B")        # Thread 2 modifies
set("thread:abc123", thread_data)   # Thread 1 writes (loses ckpt-B)
set("thread:abc123", thread_data)   # Thread 2 writes (loses ckpt-A)
```

**❌ Alternative 2: Separate keys with timestamps**
```python
# Problem: Cannot efficiently query latest without scanning all keys
set("checkpoint:abc123:2024-01-01T10:00:00:ckpt-1", data)
set("checkpoint:abc123:2024-01-01T10:01:00:ckpt-2", data)
set("checkpoint:abc123:2024-01-01T10:02:00:ckpt-3", data)

# To get latest: would need to scan ALL checkpoint keys (impossible in Memcached)
```

**❌ Alternative 3: Maintain "latest" pointer**
```python
set("thread:abc123:latest", "ckpt-001")

# Problem: Doesn't support list() operation or before= filtering
# Problem: Race condition when updating latest pointer
# Problem: No way to iterate through historical checkpoints
```

**Conclusion**: No practical solution for ordered checkpoint retrieval

#### 2. **No KEYS → Cannot Enumerate for Deletion** 🔴

**Problem**: `delete_thread()` needs to find ALL keys associated with a thread

**ValkeySaver approach**:
```python
# Find all thread keys
thread_keys = keys("thread:abc123:*")  # Returns all namespace variations

# For each thread, get all checkpoint IDs
for thread_key in thread_keys:
    checkpoint_ids = lrange(thread_key, 0, -1)
    for checkpoint_id in checkpoint_ids:
        delete(f"checkpoint:{thread_id}:{ns}:{checkpoint_id}")
        delete(f"writes:{thread_id}:{ns}:{checkpoint_id}")
```

**Memcached alternatives**:

**❌ Alternative 1: Maintain separate key index**
```python
# Store list of all keys in a separate Memcached entry
key_index = get("index:thread:abc123")
key_index.append("checkpoint:abc123:main:ckpt-001")
key_index.append("writes:abc123:main:ckpt-001")
set("index:thread:abc123", key_index)

# Problem: Race conditions updating index
# Problem: Index must be updated on EVERY checkpoint operation
# Problem: Defeats the purpose of Memcached (extra overhead)
```

**❌ Alternative 2: External key tracking (DynamoDB/Redis)**
```python
# Use external service to track Memcached keys
dynamodb.put_item("keys", {"thread_id": "abc123", "key": "checkpoint:..."})

# Problem: Requires additional infrastructure
# Problem: Two systems to maintain (Memcached + DynamoDB)
# Problem: Consistency issues between systems
# Problem: If you need Redis/DynamoDB anyway, why use Memcached?
```

**❌ Alternative 3: Don't support delete_thread()**
```python
def delete_thread(self, thread_id: str):
    raise NotImplementedError(
        "delete_thread not supported in MemcachedSaver"
    )

# Problem: Core functionality expected by users
# Problem: Violates BaseCheckpointSaver interface expectations
# Problem: Memory leaks - old checkpoints never cleaned up
```

**Conclusion**: No practical solution for thread deletion

#### 3. **No Atomic Transactions → Race Conditions** 🔴

**Problem**: Checkpoint operations must be atomic to ensure consistency

**ValkeySaver approach**:
```python
pipe = client.pipeline()
pipe.set(checkpoint_key, data)
pipe.lpush(thread_key, checkpoint_id)
pipe.expire(checkpoint_key, ttl)
pipe.expire(thread_key, ttl)
pipe.execute()  # All-or-nothing execution
```

**Memcached alternatives**:

**❌ No transaction support**
```python
client.set(checkpoint_key, data)       # ← Succeeds
client.set(thread_key, checkpoint_id)  # ← Fails (network error)
# Result: Checkpoint data stored but not in thread list
# Result: Checkpoint unreachable (orphaned data)
```

**❌ Alternative: CAS (Compare-And-Swap)**
```python
# Memcached supports CAS for single-key operations
value, cas = client.gets(key)
success = client.cas(key, new_value, cas)

# Problem: Only works for single key
# Problem: Cannot atomically update multiple keys (checkpoint + thread list)
# Problem: Complex retry logic needed
```

**Conclusion**: Cannot guarantee checkpoint consistency

#### 4. **No Persistence → Data Loss** 🟡

**Problem**: Checkpoints should survive restarts for agent resumption

**Memcached**: All data is volatile
- ElastiCache Memcached: No persistence, all data lost on restart
- No RDB/AOF like Redis
- Designed for temporary caching, not durable storage

**Impact**:
- Agent state lost on server restart
- Cannot resume long-running workflows after infrastructure changes
- Violates user expectations for checkpoint "persistence"

**ValkeySaver with Redis**: Data persists across restarts
- RDB snapshots
- AOF (Append-Only File) for durability
- Designed for both caching AND persistence

---

## Architectural Workarounds (All Inadequate)

### Workaround 1: Maintain Shadow Index in Redis/DynamoDB

**Approach**: Use Memcached for data storage, Redis/DynamoDB for metadata

```python
class MemcachedSaver:
    def __init__(self, memcached_client, redis_client):
        self.cache = memcached_client  # Store checkpoint data
        self.index = redis_client      # Store thread→checkpoint mapping

    def put(self, config, checkpoint, metadata, new_versions):
        checkpoint_id = checkpoint["id"]

        # Store data in Memcached
        self.cache.set(checkpoint_key, serialize(checkpoint))

        # Store ordering in Redis
        self.index.lpush(thread_key, checkpoint_id)

    def list(self, config, filter=None, before=None, limit=None):
        # Get ordered IDs from Redis
        checkpoint_ids = self.index.lrange(thread_key, 0, -1)

        # Fetch data from Memcached
        for checkpoint_id in checkpoint_ids:
            data = self.cache.get(checkpoint_key)
            yield deserialize(data)
```

**Problems**:
- ❌ **Complexity**: Managing two systems
- ❌ **Cost**: Need both Memcached AND Redis
- ❌ **Consistency**: Data can be out of sync (cache miss but index has key)
- ❌ **Performance**: Extra network calls
- ⚠️ **Defeats Purpose**: If you need Redis anyway, just use ValkeySaver!

### Workaround 2: Single-Key JSON with Optimistic Locking

**Approach**: Store all thread data in one Memcached key with CAS

```python
class MemcachedSaver:
    def put(self, config, checkpoint, metadata, new_versions):
        max_retries = 10
        for attempt in range(max_retries):
            # Get current thread data with CAS token
            thread_data, cas = self.client.gets(thread_key)

            if thread_data is None:
                thread_data = {"checkpoints": [], "data": {}}

            # Add new checkpoint
            checkpoint_id = checkpoint["id"]
            thread_data["checkpoints"].insert(0, checkpoint_id)
            thread_data["data"][checkpoint_id] = {
                "checkpoint": serialize(checkpoint),
                "metadata": serialize(metadata),
            }

            # Try to update with CAS
            if self.client.cas(thread_key, thread_data, cas):
                return config  # Success!

            # CAS failed, retry
            time.sleep(0.1 * attempt)

        raise Exception("Failed to save checkpoint after retries")
```

**Problems**:
- ❌ **Scalability**: All thread data in one key (size limit: 1 MB default)
- ❌ **Performance**: Must read/write entire thread data for each checkpoint
- ❌ **Contention**: High retry rate with concurrent checkpoints
- ❌ **Complexity**: Complex retry logic
- ⚠️ **Memory**: Large threads quickly exceed Memcached value size limit

### Workaround 3: Timestamp-Based Keys Without Ordering

**Approach**: Use timestamp in key, accept lack of ordering guarantees

```python
class MemcachedSaver:
    def put(self, config, checkpoint, metadata, new_versions):
        checkpoint_id = checkpoint["id"]
        timestamp = time.time()

        # Include timestamp in key
        checkpoint_key = f"ckpt:{thread_id}:{timestamp}:{checkpoint_id}"
        self.client.set(checkpoint_key, serialize(checkpoint))

        # Store "latest" pointer
        latest_key = f"latest:{thread_id}"
        self.client.set(latest_key, checkpoint_key)

    def get_tuple(self, config):
        if checkpoint_id:
            # Must know full key including timestamp (impossible!)
            pass
        else:
            # Get latest via pointer
            latest_key = f"latest:{thread_id}"
            checkpoint_key = self.client.get(latest_key)
            return self.client.get(checkpoint_key)

    def list(self, config, filter=None, before=None, limit=None):
        # IMPOSSIBLE: Cannot enumerate keys by pattern
        raise NotImplementedError("list() not supported")

    def delete_thread(self, thread_id: str):
        # IMPOSSIBLE: Cannot find all keys for thread
        raise NotImplementedError("delete_thread() not supported")
```

**Problems**:
- ❌ **Core Features Missing**: `list()` and `delete_thread()` impossible
- ❌ **Race Conditions**: "latest" pointer can point to deleted checkpoint
- ❌ **Fragmentation**: Old checkpoints accumulate (memory leak)
- ⚠️ **Violates Interface**: BaseCheckpointSaver expects these methods to work

---

## Comparison Matrix: ValkeySaver vs Hypothetical MemcachedSaver

| Feature | ValkeySaver | MemcachedSaver | Feasibility |
|---------|-------------|----------------|-------------|
| **get_tuple() - latest** | ✅ `lrange(0, 0)` | ❌ No ordering | 🔴 **Not feasible** |
| **get_tuple() - specific** | ✅ Direct GET | ✅ Direct GET (if key known) | 🟡 Partial |
| **list() - ordered** | ✅ `lrange` + batch GET | ❌ No ordering | 🔴 **Not feasible** |
| **list() - with before=** | ✅ Find index in LIST | ❌ No ordering | 🔴 **Not feasible** |
| **list() - with filter=** | ✅ Filter after fetch | ⚠️ Must fetch all first | 🟡 Inefficient |
| **put() - atomic** | ✅ Pipeline transaction | ❌ No transactions | 🔴 **Race conditions** |
| **put_writes() - atomic** | ✅ Pipeline transaction | ❌ No transactions | 🔴 **Race conditions** |
| **delete_thread()** | ✅ KEYS + DELETE | ❌ Cannot enumerate | 🔴 **Not feasible** |
| **TTL support** | ✅ Per-key EXPIRE | ✅ Per-key TTL | ✅ Compatible |
| **Persistence** | ✅ RDB/AOF | ❌ Volatile | 🟡 **Data loss risk** |
| **Multi-namespace** | ✅ Separate LISTs | ❌ Cannot enumerate | 🔴 **Not feasible** |

**Overall Feasibility**: 🔴 **NOT RECOMMENDED**

---

## Why MemcachedCache Works But MemcachedSaver Doesn't

### MemcachedCache Requirements ✅

**Simple operations that Memcached handles well**:
- `get(keys)` → `get_many(keys)` ✅
- `set(key, value, ttl)` → `set_many(...)` ✅
- `clear()` → `flush_all()` ✅ (acceptable limitation: no namespace-specific clear)
- No ordering requirements ✅
- No key enumeration needed (except for clear) ✅
- No complex transactions needed ✅

**Why it works**:
- Keys are explicitly provided by caller
- No need to discover keys
- No ordering requirements
- Cache misses are acceptable
- Temporary storage is the design goal

### MemcachedSaver Requirements ❌

**Complex operations that Memcached cannot handle**:
- `get_tuple()` → Need latest checkpoint **with ordering** ❌
- `list()` → Need **ordered iteration** through checkpoints ❌
- `list(before=...)` → Need **relative positioning** in ordered list ❌
- `delete_thread()` → Need **key enumeration** by pattern ❌
- `put()` → Need **atomic multi-key updates** ❌
- Persistence expected for resumable agents ❌

**Why it fails**:
- Checkpoint IDs are not known in advance (need discovery)
- Temporal ordering is fundamental requirement
- Atomicity is critical for consistency
- Durability is expected (checkpoint = persistence)
- Missing core Memcached features (LIST, KEYS, transactions)

---

## Alternative: Use ValkeySaver for Checkpoints, MemcachedCache for Caching

### Recommended Architecture

```python
from langgraph_checkpoint_aws import ValkeySaver, MemcachedCache

# Checkpoint persistence (requires ordering, atomicity, durability)
checkpoint_saver = ValkeySaver(
    valkey_client,  # ElastiCache for Redis/Valkey
    ttl_seconds=86400  # 24 hour checkpoint retention
)

# Application caching (simple K-V, no ordering needed)
app_cache = MemcachedCache(
    memcached_client,  # ElastiCache for Memcached
    ttl_seconds=3600  # 1 hour cache retention
)

# Use both appropriately
graph = StateGraph(...)
graph = graph.compile(
    checkpointer=checkpoint_saver,  # Persistent state
    cache=app_cache,                # Volatile caching
)
```

### Why This Works

| Component | Technology | Reason |
|-----------|-----------|--------|
| **Checkpoints** | Valkey/Redis | Needs LIST, KEYS, transactions, persistence |
| **Cache** | Memcached | Simple K-V, cheaper, faster for pure caching |

**Cost Optimization**:
- Use cheaper Memcached where ordering/persistence not needed
- Use Redis/Valkey only where advanced features required
- Best of both worlds

---

## Implementation Complexity If Attempted

### Minimal Viable MemcachedSaver (With Major Limitations)

```python
class MemcachedSaver(BaseCheckpointSaver):
    """
    EXPERIMENTAL: MemcachedSaver with severe limitations.

    ⚠️ LIMITATIONS:
    - list() not supported (no ordering)
    - delete_thread() not supported (cannot enumerate keys)
    - get_tuple() only works with explicit checkpoint_id
    - No atomic operations (race conditions possible)
    - No persistence (data lost on restart)

    NOT RECOMMENDED FOR PRODUCTION USE.
    Use ValkeySaver instead.
    """

    def __init__(self, client, ttl=None, serde=None):
        super().__init__(serde=serde)
        self.client = client
        self.ttl = ttl

    def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None:
        """Only works if checkpoint_id is explicitly provided."""
        checkpoint_id = get_checkpoint_id(config)
        if not checkpoint_id:
            raise NotImplementedError(
                "MemcachedSaver cannot retrieve latest checkpoint. "
                "You must provide explicit checkpoint_id in config."
            )

        # Can only get specific checkpoint
        checkpoint_data = self.client.get(checkpoint_key)
        writes_data = self.client.get(writes_key)
        return self._deserialize(checkpoint_data, writes_data)

    def list(
        self,
        config: RunnableConfig | None,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
    ) -> Iterator[CheckpointTuple]:
        """NOT SUPPORTED: Memcached cannot enumerate or order checkpoints."""
        raise NotImplementedError(
            "list() not supported in MemcachedSaver. "
            "Memcached does not support key enumeration or ordering. "
            "Use ValkeySaver for checkpoint listing."
        )

    def put(
        self,
        config: RunnableConfig,
        checkpoint: Checkpoint,
        metadata: CheckpointMetadata,
        new_versions: ChannelVersions,
    ) -> RunnableConfig:
        """⚠️ NOT ATOMIC: Race conditions possible with concurrent puts."""
        checkpoint_id = checkpoint["id"]

        # Store checkpoint (not atomic!)
        self.client.set(checkpoint_key, serialize(checkpoint), expire=self.ttl)

        # ⚠️ Problem: If this fails, checkpoint is orphaned
        self.client.set(writes_key, serialize([]), expire=self.ttl)

        return config

    def put_writes(
        self,
        config: RunnableConfig,
        writes: Sequence[tuple[str, Any]],
        task_id: str,
        task_path: str = "",
    ) -> None:
        """⚠️ RACE CONDITION: Not atomic read-modify-write."""
        # Read existing writes
        existing = self.client.get(writes_key)
        existing_writes = deserialize(existing) if existing else []

        # ⚠️ Problem: Another process could update writes here

        # Append new writes
        existing_writes.extend(serialize_writes(writes))

        # Write back (might overwrite concurrent updates!)
        self.client.set(writes_key, serialize(existing_writes), expire=self.ttl)

    def delete_thread(self, thread_id: str) -> None:
        """NOT SUPPORTED: Memcached cannot enumerate keys by pattern."""
        raise NotImplementedError(
            "delete_thread() not supported in MemcachedSaver. "
            "Memcached does not support key pattern matching. "
            "Use ValkeySaver for thread deletion."
        )
```

### What This Implementation Cannot Do

1. ❌ Get latest checkpoint (no ordering)
2. ❌ List checkpoints (no enumeration)
3. ❌ Paginate through history (no ordering)
4. ❌ Filter checkpoints with `before=` (no ordering)
5. ❌ Delete thread (no key enumeration)
6. ❌ Guarantee consistency (no transactions)
7. ❌ Survive restarts (no persistence)

### What Users Expect (From BaseCheckpointSaver Interface)

1. ✅ `get_tuple()` - with and without checkpoint_id
2. ✅ `list()` - ordered iteration with filtering
3. ✅ `put()` - atomic checkpoint storage
4. ✅ `delete_thread()` - cleanup all thread data
5. ✅ Data persistence across restarts
6. ✅ Consistency guarantees

**Gap**: 85% of expected functionality is missing or broken

---

## Effort Estimate If Implementing (Not Recommended)

### Optimistic Estimate: 5-7 days

**Phase 1: Core Implementation (2 days)**
- [ ] Create `MemcachedSaver` class skeleton
- [ ] Implement minimal `get_tuple()` (checkpoint_id required)
- [ ] Implement limited `put()` (no atomicity)
- [ ] Add comprehensive warning documentation

**Phase 2: Testing (2 days)**
- [ ] Unit tests (mock pymemcache)
- [ ] Document all limitations in tests
- [ ] Integration tests (require memcached server)
- [ ] Test race condition scenarios

**Phase 3: Documentation (1 day)**
- [ ] API documentation with limitations
- [ ] Warning notices in docstrings
- [ ] Comparison guide (when to use Valkey vs Memcached)
- [ ] Migration guide (Memcached → Valkey)

**Phase 4: PR Review and Feedback (2 days)**
- [ ] Address review comments
- [ ] Justify design decisions
- [ ] Defend against "why would anyone use this?"

### Realistic Estimate: Don't Implement

**Reasons**:
1. Users will be confused by limitations
2. Support burden for explaining why things don't work
3. Potential data loss/corruption from race conditions
4. No clear use case (if you need checkpointing, use Redis)
5. Implementation effort better spent elsewhere

---

## Final Recommendation

### ❌ **DO NOT IMPLEMENT MemcachedSaver**

**Reasons**:

#### 1. **Architectural Mismatch**
- Memcached designed for **volatile caching**
- Checkpoint savers need **persistent state management**
- Missing critical features (LIST, KEYS, transactions)

#### 2. **Severe Limitations**
- Cannot get latest checkpoint
- Cannot list checkpoints in order
- Cannot delete thread
- No consistency guarantees
- Data lost on restart

#### 3. **User Experience**
- Confusing limitations
- Unexpected behavior (race conditions)
- Data loss risks
- "Works sometimes" is worse than "doesn't work"

#### 4. **Better Alternatives Exist**
- **ValkeySaver**: Full-featured, reliable, proven
- **MemcachedCache**: Memcached for what it's good at
- Use both together for optimal cost/performance

#### 5. **Implementation Cost vs Value**
- **Cost**: 5-7 days development + ongoing support burden
- **Value**: Minimal (no users would benefit from this)
- **Risk**: Data loss, race conditions, user confusion

### ✅ **RECOMMENDED APPROACH**

#### Use ValkeySaver for Checkpoints

```python
from langgraph_checkpoint_aws import ValkeySaver

# ElastiCache for Redis (Valkey-compatible)
checkpoint_saver = ValkeySaver.from_conn_string(
    "valkey://my-cluster.cache.amazonaws.com:6379",
    ttl_seconds=86400  # 24 hours
)

graph = StateGraph(...).compile(checkpointer=checkpoint_saver)
```

**Why**:
- ✅ Full checkpoint saver features
- ✅ Ordering, enumeration, transactions
- ✅ Persistence and durability
- ✅ Proven and tested
- ✅ AWS ElastiCache for Redis supports this

#### Use MemcachedCache for Application Caching

```python
from langgraph_checkpoint_aws import MemcachedCache

# ElastiCache for Memcached (for caching)
app_cache = MemcachedCache.from_server(
    "my-memcached.cache.amazonaws.com:11211",
    ttl_seconds=3600  # 1 hour
)

# Use cache in your application logic
def expensive_computation():
    if cached := app_cache.get(key):
        return cached

    result = compute()
    app_cache.set(key, result, ttl=3600)
    return result
```

**Why**:
- ✅ Memcached perfect for this use case
- ✅ Simple K-V operations
- ✅ Cost-effective
- ✅ No ordering/persistence needed

### Cost Optimization Strategy

**Hybrid Approach**:
```yaml
# ElastiCache Configuration

# Redis/Valkey for Checkpoints (state persistence)
Cache1:
  Type: cache.t4g.small  # $0.034/hr
  Purpose: Checkpoint persistence
  Nodes: 2 (primary + replica)
  Cost: ~$50/month

# Memcached for Application Cache (volatile)
Cache2:
  Type: cache.t4g.small  # $0.024/hr (30% cheaper)
  Purpose: Application caching
  Nodes: 2
  Cost: ~$35/month

Total: ~$85/month for full caching + persistence
```

**vs Single Redis Approach**:
```yaml
# Single larger Redis instance
Cache1:
  Type: cache.t4g.medium  # $0.068/hr
  Purpose: Both checkpoints and caching
  Nodes: 2
  Cost: ~$100/month
```

**Savings**: $15/month ($180/year) + better architecture separation

---

## Appendix A: Feature-by-Feature Analysis

### Feature 1: get_tuple() - Get Latest Checkpoint

**ValkeySaver**:
```python
# Get latest checkpoint ID from ordered list
checkpoint_ids = client.lrange(f"thread:{thread_id}", 0, 0)
latest_id = checkpoint_ids[0]

# Fetch checkpoint data
checkpoint_data = client.get(f"checkpoint:{thread_id}:{latest_id}")
```

**Memcached Attempt**:
```python
# Option 1: Maintain "latest" pointer
latest_id = client.get(f"latest:{thread_id}")
checkpoint_data = client.get(f"checkpoint:{thread_id}:{latest_id}")

# Problems:
# - Race condition updating "latest" pointer
# - "latest" might point to deleted checkpoint
# - No support for list() operations
```

**Verdict**: ⚠️ Fragile workaround, breaks list()

### Feature 2: list() - Ordered Iteration

**ValkeySaver**:
```python
# Get ordered list of all checkpoint IDs
all_ids = client.lrange(f"thread:{thread_id}", 0, -1)

# Paginate and filter
for i in range(0, len(all_ids), batch_size):
    batch_ids = all_ids[i:i+batch_size]
    # Batch fetch checkpoint data
    for checkpoint_id in batch_ids:
        data = client.get(f"checkpoint:{thread_id}:{checkpoint_id}")
        yield deserialize(data)
```

**Memcached Attempt**:
```python
# IMPOSSIBLE: No way to get ordered list of checkpoint IDs
# Cannot use LRANGE (no LIST type)
# Cannot use KEYS (no pattern matching)
# Cannot enumerate keys

def list(self, config, filter=None, before=None, limit=None):
    raise NotImplementedError("Memcached cannot list keys")
```

**Verdict**: ❌ Not feasible

### Feature 3: delete_thread() - Cleanup

**ValkeySaver**:
```python
# Find all thread-related keys
thread_keys = client.keys(f"thread:{thread_id}:*")

# For each namespace, get all checkpoint IDs
for thread_key in thread_keys:
    checkpoint_ids = client.lrange(thread_key, 0, -1)

    # Delete all checkpoint and writes keys
    for checkpoint_id in checkpoint_ids:
        client.delete(f"checkpoint:{thread_id}:*:{checkpoint_id}")
        client.delete(f"writes:{thread_id}:*:{checkpoint_id}")

# Delete thread index keys
client.delete(*thread_keys)
```

**Memcached Attempt**:
```python
# Option 1: Track keys in external system (defeats purpose)
key_list = dynamodb.query("keys", thread_id=thread_id)
for key in key_list:
    client.delete(key)
    dynamodb.delete("keys", key=key)

# Option 2: Don't support deletion (memory leak)
def delete_thread(self, thread_id: str):
    raise NotImplementedError("Memcached cannot enumerate keys")

# Option 3: Rely on TTL expiration (slow, incomplete)
# Just wait for TTL to expire... (but what if TTL is long?)
```

**Verdict**: ❌ Not feasible without external system

### Feature 4: Atomic put() - Consistency

**ValkeySaver**:
```python
pipe = client.pipeline()
pipe.set(checkpoint_key, checkpoint_data)
pipe.set(writes_key, writes_data)
pipe.lpush(thread_key, checkpoint_id)
pipe.expire(checkpoint_key, ttl)
pipe.expire(writes_key, ttl)
pipe.expire(thread_key, ttl)
results = pipe.execute()  # All-or-nothing

# If any operation fails, ALL operations are rolled back
```

**Memcached Attempt**:
```python
# No transactions - must do operations individually
client.set(checkpoint_key, checkpoint_data, expire=ttl)  # Succeeds
client.set(writes_key, writes_data, expire=ttl)          # Succeeds
# Network error here...
# Checkpoint stored but not in any index!
# Data is orphaned and unreachable

# Option: Use CAS for single key
value, cas_id = client.gets(key)
success = client.cas(key, new_value, cas_id)
# But CAS only works for ONE key, not multiple
```

**Verdict**: ❌ No atomicity across multiple keys

---

## Appendix B: User Impact Analysis

### Scenario 1: Agent Developer Using Checkpoints

**User Expectation**:
```python
# Save agent state
config = {"configurable": {"thread_id": "user-123"}}
for step in agent.stream(input, config=config):
    print(step)  # Each step automatically checkpointed

# Later: Resume from latest checkpoint
state = agent.get_state(config)  # ← Expects latest checkpoint
agent.update_state(state, new_values)
history = agent.get_state_history(config)  # ← Expects ordered history
```

**With MemcachedSaver**:
```python
# Save agent state
config = {"configurable": {"thread_id": "user-123"}}
for step in agent.stream(input, config=config):
    print(step)  # Checkpoints saved (no ordering info)

# Later: Resume from latest checkpoint
state = agent.get_state(config)  # ← ERROR: Cannot get latest!
# Error: NotImplementedError: Must provide checkpoint_id

# Try to get history
history = agent.get_state_history(config)  # ← ERROR: list() not supported
# Error: NotImplementedError: Memcached cannot list keys
```

**Result**: ❌ Core functionality broken

### Scenario 2: Cleaning Up Old Threads

**User Expectation**:
```python
# Cleanup old user threads
for user_id in inactive_users:
    thread_id = f"user-{user_id}"
    checkpointer.delete_thread(thread_id)
```

**With MemcachedSaver**:
```python
# Cleanup old user threads
for user_id in inactive_users:
    thread_id = f"user-{user_id}"
    checkpointer.delete_thread(thread_id)  # ← ERROR!
    # Error: NotImplementedError: Memcached cannot enumerate keys

# Old checkpoints accumulate in Memcached
# Memory fills up
# Performance degrades
# Manual TTL expiration only option (slow, incomplete)
```

**Result**: ❌ Memory leak, no cleanup

### Scenario 3: Debugging Agent Execution

**User Expectation**:
```python
# Debug agent by examining checkpoint history
config = {"configurable": {"thread_id": "debug-session"}}
checkpoints = list(checkpointer.list(config, limit=10))

for i, checkpoint in enumerate(checkpoints):
    print(f"Step {i}: {checkpoint.metadata}")
    print(f"State: {checkpoint.checkpoint}")
```

**With MemcachedSaver**:
```python
# Debug agent by examining checkpoint history
config = {"configurable": {"thread_id": "debug-session"}}
checkpoints = list(checkpointer.list(config, limit=10))  # ← ERROR!
# Error: NotImplementedError: Memcached cannot list keys

# Alternative: Must know checkpoint IDs somehow?
# But how do you get checkpoint IDs without list()?
# Impossible!
```

**Result**: ❌ Cannot debug, cannot inspect history

---

## Conclusion

### Summary

| Aspect | Assessment |
|--------|-----------|
| **Technical Feasibility** | 🔴 Not feasible without severe limitations |
| **User Experience** | 🔴 Breaks expected functionality |
| **Data Safety** | 🔴 Race conditions, no persistence |
| **Development Cost** | 🟡 Medium (5-7 days) |
| **Maintenance Cost** | 🔴 High (support burden) |
| **Value Proposition** | 🔴 None (better alternatives exist) |

### Recommendation: **DO NOT IMPLEMENT**

**Instead**:
1. ✅ Use **ValkeySaver** for checkpoint persistence
2. ✅ Use **MemcachedCache** for application caching
3. ✅ Document this architectural decision
4. ✅ Provide cost optimization guide for hybrid approach

**If users ask for MemcachedSaver**:
> "Memcached is designed for simple key-value caching and lacks features essential for checkpoint persistence:
> - No ordered data structures (LIST) for checkpoint sequencing
> - No key enumeration (KEYS) for thread cleanup
> - No atomic transactions for consistency
> - No persistence for durability
>
> Use ValkeySaver (Redis/Valkey) for checkpoints and MemcachedCache for application caching. This hybrid approach provides the best cost/performance balance."

---

## References

- [LangGraph Checkpoint Saver Interface](https://python.langchain.com/docs/langgraph/checkpoint)
- [Pymemcache Documentation](https://pymemcache.readthedocs.io/)
- [AWS ElastiCache for Memcached](https://docs.aws.amazon.com/AmazonElastiCache/latest/mem-ug/WhatIs.html)
- [AWS ElastiCache for Redis](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/WhatIs.html)
- [Redis vs Memcached](https://aws.amazon.com/elasticache/redis-vs-memcached/)
- ValkeySaver Implementation: `langgraph_checkpoint_aws/checkpoint/valkey/`
- MemcachedCache Design: `MEMCACHED_CACHE_DESIGN.md`
