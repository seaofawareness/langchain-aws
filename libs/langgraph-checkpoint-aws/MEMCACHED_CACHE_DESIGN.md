# MemcachedCache Design Evaluation

## Overview

This document evaluates the design and implementation of a `MemcachedCache` class following the same pattern as `ValkeyCache` but using `pymemcache` for AWS ElastiCache Memcached support.

## Current Status: ValkeyCache Pattern

### Architecture
- **Base Class**: `BaseCache[ValueT]` from `langgraph.cache.base`
- **Serialization**: `SerializerProtocol` (defaults to `JsonPlusSerializer`)
- **Client**: `valkey.Valkey` (Redis-compatible)
- **Features**:
  - TTL support (per-key and default)
  - Connection pooling
  - Namespace organization using `/` separator
  - Async/sync operations
  - Batch operations (up to 100 keys)
  - Prefix-based key organization

### Key Methods
1. `__init__(client, serde, prefix, ttl)` - Initialize with client
2. `from_conn_string()` - Create from connection string with pool
3. `from_pool()` - Create from existing connection pool
4. `aget()` / `get()` - Retrieve cached values (async/sync)
5. `aset()` / `set()` - Store cached values with TTL (async/sync)
6. `aclear()` / `clear()` - Clear namespaces or all cache (async/sync)

### Key Storage Format
- **Pattern**: `{prefix}{namespace}/{key}`
- **Example**: `langgraph:cache:users/123/session_key`
- **Serialization**: `{encoding}:{binary_data}` (e.g., `json:<data>`)

## Memcached vs Redis/Valkey Comparison

| Feature | Valkey/Redis | Memcached | Impact |
|---------|--------------|-----------|--------|
| **Data Structure** | Complex (lists, sets, hashes) | Simple key-value | ✅ Both support basic K-V |
| **TTL Support** | Per-key TTL | Per-key TTL | ✅ Compatible |
| **Pipelining** | Yes (transactions) | Limited (multi get/set) | ⚠️ Different API |
| **Namespaces** | SCAN with patterns | No native support | ⚠️ Must simulate |
| **Key Length** | 512 MB | 250 bytes max | ⚠️ Must handle carefully |
| **Value Size** | 512 MB | 1 MB default | ✅ Sufficient |
| **Atomic Operations** | Many (INCR, etc.) | Limited | ✅ Not needed |
| **Connection Pooling** | Built-in | Built-in | ✅ Compatible |
| **TLS Support** | Yes | Yes (pymemcache) | ✅ Compatible |

## Pymemcache Client API

### Installation
```bash
pip install pymemcache
```

### Basic Operations
```python
from pymemcache.client.base import Client
from pymemcache.client.hash import HashClient  # For multiple servers
from pymemcache import PooledClient  # Connection pooling

# Single server
client = Client(('localhost', 11211))

# With connection pooling
client = PooledClient(
    ('localhost', 11211),
    max_pool_size=10,
    connect_timeout=5,
    timeout=2
)

# Multiple servers (consistent hashing)
client = HashClient([
    ('localhost', 11211),
    ('localhost', 11212)
])

# Operations
client.set('key', 'value', expire=3600)  # TTL in seconds
value = client.get('key')
client.delete('key')
client.get_many(['key1', 'key2', 'key3'])  # Batch get
client.set_many({'key1': 'val1', 'key2': 'val2'}, expire=3600)  # Batch set
```

### Key Differences from Valkey
1. **No SCAN operation** - Cannot efficiently list keys by pattern
2. **set_many/get_many** instead of pipeline for batch operations
3. **Key length limit** - 250 bytes (Valkey: practically unlimited)
4. **No transactions** - set_many is not atomic

## Design Considerations for MemcachedCache

### 1. Key Length Limitation ⚠️

**Problem**: Memcached keys limited to 250 bytes

**ValkeyCache Keys**:
```
langgraph:cache:users/profile/session/data/v2/key_name = ~60 chars base + namespace
```

**Solutions**:
- ✅ **Hash long keys**: Use MD5/SHA256 hash for keys > 200 bytes
- ✅ **Shorter prefix**: Use shorter default prefix (e.g., `lgc:` instead of `langgraph:cache:`)
- ⚠️ **Limit namespace depth**: Warn if namespace too deep

**Recommendation**: Hash keys if `len(prefix + namespace + key) > 200`

### 2. Namespace Clear Operation ⚠️

**Problem**: Memcached has no SCAN or pattern-based key listing

**ValkeyCache Implementation**:
```python
async def aclear(self, namespaces: Sequence[Namespace] | None = None):
    if namespaces is None:
        pattern = f"{self.prefix}*"
    else:
        patterns = [f"{self.prefix}{'/'.join(ns)}/*" for ns in namespaces]

    keys = await scan_keys(pattern)  # Uses SCAN command
    await delete_keys(keys)
```

**Solutions**:
- ❌ **Track keys separately** - Defeats purpose of cache (adds overhead)
- ⚠️ **Limited clear()** - Only support `clear(None)` to flush all cache
- ✅ **Use external key tracking** - Optional Redis/DynamoDB for key index
- ✅ **Document limitation** - Make clear that namespace clear is not supported

**Recommendation**:
1. Only support `clear(None)` → `client.flush_all()`
2. Raise `NotImplementedError` for namespace-specific clear
3. Document this limitation clearly

### 3. Serialization Format

**Keep same format as ValkeyCache**:
```python
encoding = "json"  # or "msgpack", "pickle", etc.
data = serializer.dumps_typed(value)  # Returns (encoding, binary_data)
stored_value = f"{encoding}:{data}"  # Store as "encoding:data"
```

**Recommendation**: ✅ Use identical serialization format for compatibility

### 4. Connection Pooling

**Pymemcache Options**:
```python
from pymemcache import PooledClient

client = PooledClient(
    server=('localhost', 11211),
    max_pool_size=10,
    connect_timeout=5.0,
    timeout=2.0,
    no_delay=True,
    serde=None  # We'll handle serialization ourselves
)
```

**Recommendation**: ✅ Use `PooledClient` as default

### 5. Multi-Server Support

**Pymemcache HashClient**:
```python
from pymemcache.client.hash import HashClient

client = HashClient([
    ('server1', 11211),
    ('server2', 11211),
    ('server3', 11211)
])
```

**Recommendation**: ✅ Support both single and multi-server configurations

## Proposed Implementation

### File Structure
```
langgraph_checkpoint_aws/
├── cache/
│   ├── __init__.py
│   ├── valkey/
│   │   ├── __init__.py
│   │   └── cache.py (ValkeyCache)
│   └── memcached/
│       ├── __init__.py
│       ├── cache.py (MemcachedCache)
│       └── utils.py (helper functions)
```

### Class Signature

```python
class MemcachedCache(BaseCache[ValueT]):
    """Memcached-based cache implementation with TTL support.

    Features:
    - TTL support (per-key and default)
    - Connection pooling
    - Multi-server support with consistent hashing
    - Async operations
    - Batch operations
    - Automatic key hashing for long keys

    Limitations:
    - Namespace-specific clear() not supported (only clear all)
    - Keys are hashed if > 200 bytes
    - No atomic batch operations

    Example:
        ```python
        # Single server with pooling
        with MemcachedCache.from_server(
            "localhost:11211",
            ttl_seconds=3600.0,
            pool_size=10
        ) as cache:
            # Use cache...

        # Multiple servers (consistent hashing)
        with MemcachedCache.from_servers(
            ["server1:11211", "server2:11211", "server3:11211"],
            ttl_seconds=3600.0
        ) as cache:
            # Use cache with multi-server...

        # Or direct initialization
        from pymemcache import PooledClient
        client = PooledClient(('localhost', 11211))
        cache = MemcachedCache(client, prefix="lgc:")
        ```
    """

    def __init__(
        self,
        client: PooledClient | HashClient,
        *,
        serde: SerializerProtocol | None = None,
        prefix: str = "lgc:",  # Shorter than ValkeyCache
        ttl: float | None = None,
        hash_long_keys: bool = True,  # Auto-hash keys > 200 bytes
    ) -> None:
        ...

    @classmethod
    @contextmanager
    def from_server(
        cls,
        server: str,  # "host:port" format
        *,
        prefix: str = "lgc:",
        ttl_seconds: float | None = None,
        serde: SerializerProtocol | None = None,
        pool_size: int = 10,
        connect_timeout: float = 5.0,
        timeout: float = 2.0,
    ) -> Generator[MemcachedCache[ValueT], None, None]:
        """Create cache from server string with pooling."""
        ...

    @classmethod
    @contextmanager
    def from_servers(
        cls,
        servers: list[str],  # ["host1:port1", "host2:port2", ...]
        *,
        prefix: str = "lgc:",
        ttl_seconds: float | None = None,
        serde: SerializerProtocol | None = None,
    ) -> Generator[MemcachedCache[ValueT], None, None]:
        """Create cache with multi-server support (consistent hashing)."""
        ...

    # Core methods (same signature as ValkeyCache)
    def _make_key(self, ns: Namespace, key: str) -> str:
        """Create memcached key with optional hashing for long keys."""
        ...

    async def aget(self, keys: Sequence[FullKey]) -> dict[FullKey, ValueT]:
        """Get cached values (uses get_many)."""
        ...

    async def aset(self, pairs: Mapping[FullKey, tuple[ValueT, int | None]]) -> None:
        """Set cached values (uses set_many)."""
        ...

    async def aclear(self, namespaces: Sequence[Namespace] | None = None) -> None:
        """Clear cache. Only supports clearing ALL (flush_all)."""
        if namespaces is not None:
            raise NotImplementedError(
                "Memcached does not support namespace-specific clearing. "
                "Use aclear() without arguments to clear all cached data."
            )
        ...
```

### Key Hashing Implementation

```python
import hashlib

def _make_key(self, ns: Namespace, key: str) -> str:
    """Create memcached key with automatic hashing for long keys."""
    ns_str = "/".join(ns) if ns else ""
    full_key = f"{self.prefix}{ns_str}/{key}" if ns_str else f"{self.prefix}{key}"

    # Memcached key length limit is 250 bytes
    if self.hash_long_keys and len(full_key) > 200:
        # Hash the full key and use a shorter format
        key_hash = hashlib.sha256(full_key.encode()).hexdigest()[:32]
        # Keep some readability by including prefix
        return f"{self.prefix}h:{key_hash}"

    return full_key
```

### Batch Operations

```python
async def _set_batch(
    self, batch: list[tuple[FullKey, tuple[ValueT, int | None]]]
) -> None:
    """Set a batch of key-value pairs using set_many."""
    data_to_set = {}

    for (ns, key), (value, ttl) in batch:
        try:
            mc_key = self._make_key(ns, key)
            encoding, data = self.serde.dumps_typed(value)
            serialized_value = encoding.encode() + b":" + data

            final_ttl = ttl if ttl is not None else self.ttl
            if final_ttl is not None and final_ttl <= 0:
                logger.error(f"Invalid TTL {final_ttl} for key {key}, skipping")
                continue

            data_to_set[mc_key] = (serialized_value, int(final_ttl) if final_ttl else 0)

        except Exception as e:
            logger.error(f"Error preparing cached value for key {key}: {e}")
            continue

    if data_to_set:
        try:
            # set_many with expire parameter
            # Note: pymemcache set_many uses same expire for all keys
            # We need to call set individually for different TTLs
            same_ttl_groups = {}
            for k, (v, ttl) in data_to_set.items():
                if ttl not in same_ttl_groups:
                    same_ttl_groups[ttl] = {}
                same_ttl_groups[ttl][k] = v

            for ttl, items in same_ttl_groups.items():
                await asyncio.to_thread(
                    self.client.set_many,
                    items,
                    expire=ttl if ttl > 0 else 0
                )

        except Exception as e:
            logger.error(f"Error executing set_many: {e}")
            raise
```

## Pros and Cons

### ✅ Pros
1. **Consistent API**: Follows same pattern as ValkeyCache
2. **ElastiCache Support**: Native AWS ElastiCache Memcached support
3. **Performance**: Memcached is very fast for simple K-V operations
4. **Lower Memory**: Memcached has lower memory overhead than Redis
5. **Multi-threaded**: Memcached is multi-threaded (better CPU utilization)
6. **TLS Support**: pymemcache supports TLS for secure connections
7. **Cost**: ElastiCache Memcached can be cheaper than Redis

### ⚠️ Cons & Limitations
1. **No namespace clear**: Cannot clear specific namespaces efficiently
2. **Key hashing**: Long keys must be hashed (loses readability)
3. **No SCAN**: Cannot list keys by pattern
4. **No persistence**: Data lost on restart (Redis can persist)
5. **No replication**: Memcached clusters don't replicate (use multiple servers)
6. **Different batching**: `set_many` applies same TTL to all keys in batch

## Implementation Complexity

### Effort Estimate
- **Low Complexity** (2-3 days):
  - Core MemcachedCache class (~500 lines, similar to ValkeyCache)
  - Unit tests (~1000 lines, similar pattern)
  - Integration tests (~500 lines)
  - Documentation updates

### Files to Create/Modify

**New Files**:
1. `langgraph_checkpoint_aws/cache/memcached/__init__.py`
2. `langgraph_checkpoint_aws/cache/memcached/cache.py`
3. `langgraph_checkpoint_aws/cache/memcached/utils.py` (if needed)
4. `tests/unit_tests/cache/memcached/test_memcached_cache_unit.py`
5. `tests/integration_tests/cache/memcached/test_memcached_cache_integration.py`
6. `samples/cache/memcached_cache.ipynb`

**Modified Files**:
1. `pyproject.toml` - Add pymemcache optional dependency
2. `langgraph_checkpoint_aws/__init__.py` - Export MemcachedCache
3. `langgraph_checkpoint_aws/cache/__init__.py` - Export from memcached module
4. `README.md` - Add MemcachedCache documentation

## Recommended Approach

### Phase 1: Core Implementation
1. Add `pymemcache` to optional dependencies
2. Create `MemcachedCache` class following ValkeyCache pattern
3. Implement key hashing for long keys
4. Implement batch operations with TTL grouping
5. Add `memcached_available` flag

### Phase 2: Testing
1. Create comprehensive unit tests (mock pymemcache client)
2. Create integration tests (require memcached server)
3. Test with AWS ElastiCache Memcached

### Phase 3: Documentation
1. Add API documentation
2. Create usage examples notebook
3. Document limitations (no namespace clear, key hashing)
4. Add to main README

## Code Diff from ValkeyCache

### Key Differences

```python
# ValkeyCache
class ValkeyCache(BaseCache[ValueT]):
    client: Valkey  # Redis-compatible

    async def aclear(self, namespaces: Sequence[Namespace] | None = None):
        # Uses SCAN to find keys matching pattern
        if namespaces is None:
            pattern = f"{self.prefix}*"
        else:
            patterns = [f"{self.prefix}{'/'.join(ns)}/*" for ns in namespaces]
        # ... scan and delete

# MemcachedCache
class MemcachedCache(BaseCache[ValueT]):
    client: PooledClient | HashClient  # Memcached

    async def aclear(self, namespaces: Sequence[Namespace] | None = None):
        # Memcached doesn't support pattern-based key listing
        if namespaces is not None:
            raise NotImplementedError(
                "Memcached does not support namespace-specific clearing. "
                "Use aclear() without arguments to clear all cached data."
            )
        # Clear all cache
        await asyncio.to_thread(self.client.flush_all)
```

### Batch Operations

```python
# ValkeyCache - uses pipeline
pipe = self.client.pipeline(transaction=True)
for key, value in items:
    pipe.setex(key, ttl, value) if ttl else pipe.set(key, value)
await asyncio.to_thread(pipe.execute)

# MemcachedCache - group by TTL and use set_many
ttl_groups = defaultdict(dict)
for key, (value, ttl) in items:
    ttl_groups[ttl][key] = value

for ttl, data in ttl_groups.items():
    await asyncio.to_thread(self.client.set_many, data, expire=ttl or 0)
```

## pyproject.toml Changes

```toml
[project.optional-dependencies]
valkey = [
    "valkey>=6.1.1",
    "orjson>=3.11.3"
]
memcached = [
    "pymemcache>=4.0.0"
]
```

## AWS ElastiCache Integration

### ElastiCache for Memcached
- **Supported versions**: 1.6.x
- **Node types**: cache.t4g.micro to cache.r7g.16xlarge
- **TLS**: Supported (in-transit encryption)
- **Auto Discovery**: Supported (cluster mode)
- **Pricing**: ~30% cheaper than Redis for same instance type

### Connection String Format
```python
# Single endpoint
"memcached://my-cluster.abc123.cfg.use1.cache.amazonaws.com:11211"

# With TLS
"memcached+tls://my-cluster.abc123.cfg.use1.cache.amazonaws.com:11211"
```

## Recommendation: Implementation Strategy

### ✅ **RECOMMENDED**: Implement MemcachedCache

**Reasons**:
1. ✅ AWS customers use ElastiCache Memcached extensively
2. ✅ Cost-effective alternative to Redis/Valkey
3. ✅ Simple implementation (2-3 days effort)
4. ✅ Follows established ValkeyCache pattern
5. ✅ Limitations are acceptable for cache use case

**Acceptable Trade-offs**:
- No namespace-specific clear (flush all is fine for cache)
- Key hashing for long keys (transparent to users)
- No SCAN operation (not needed for cache get/set)

### Implementation Priorities

**P0 (Must Have)**:
- [x] Core MemcachedCache class
- [x] Serialization compatible with ValkeyCache
- [x] TTL support (per-key and default)
- [x] Batch operations (get_many, set_many with TTL grouping)
- [x] Key hashing for long keys
- [x] Unit tests

**P1 (Should Have)**:
- [x] Connection pooling support
- [x] Multi-server support (HashClient)
- [x] Integration tests
- [x] Documentation and examples

**P2 (Nice to Have)**:
- [ ] TLS connection support
- [ ] Auto-discovery for ElastiCache cluster mode
- [ ] Metrics/monitoring hooks
- [ ] Key compression for large values

## Alternative: Don't Implement

**If NOT implementing**, reasons would be:
- ❌ Limited user demand (most use Redis/Valkey)
- ❌ Maintenance burden for another cache backend
- ❌ Limitations might confuse users (namespace clear)

However, given AWS's strong ElastiCache Memcached presence and the low implementation cost, **implementing MemcachedCache is recommended**.

## Next Steps

1. ✅ Add `pymemcache` to `[project.optional-dependencies]`
2. ✅ Create `MemcachedCache` class
3. ✅ Implement key hashing utility
4. ✅ Write unit tests
5. ✅ Write integration tests
6. ✅ Add documentation
7. ✅ Create PR

**Estimated Timeline**: 2-3 days for full implementation with tests and docs
