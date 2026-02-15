# Rate Limiter - Code Snippets

## Token Bucket Implementation

### Python

```python
"""
Token Bucket Rate Limiter

Algorithm: Tokens are added at a fixed rate. Each request consumes one token.
If no tokens available, request is rejected.

Use case: APIs that want to allow bursts up to bucket capacity.
"""

import time
import threading
from dataclasses import dataclass
from typing import Dict


@dataclass
class Bucket:
    tokens: float
    last_refill: float
    capacity: int
    refill_rate: float  # tokens per second


class TokenBucketRateLimiter:
    def __init__(self, capacity: int = 10, refill_rate: float = 1.0):
        """
        Initialize token bucket rate limiter.

        Args:
            capacity: Maximum tokens (burst size)
            refill_rate: Tokens added per second
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.buckets: Dict[str, Bucket] = {}
        self.lock = threading.Lock()

    def allow_request(self, client_id: str) -> bool:
        """
        Check if request is allowed for given client.

        Returns:
            True if allowed, False if rate limited
        """
        with self.lock:
            now = time.time()

            # Get or create bucket
            if client_id not in self.buckets:
                self.buckets[client_id] = Bucket(
                    tokens=self.capacity,
                    last_refill=now,
                    capacity=self.capacity,
                    refill_rate=self.refill_rate
                )

            bucket = self.buckets[client_id]

            # Refill tokens based on elapsed time
            elapsed = now - bucket.last_refill
            tokens_to_add = elapsed * bucket.refill_rate
            bucket.tokens = min(bucket.capacity, bucket.tokens + tokens_to_add)
            bucket.last_refill = now

            # Check if we have tokens
            if bucket.tokens >= 1:
                bucket.tokens -= 1
                return True

            return False

    def get_remaining(self, client_id: str) -> int:
        """Get remaining tokens for client."""
        if client_id in self.buckets:
            return int(self.buckets[client_id].tokens)
        return self.capacity


# Usage
limiter = TokenBucketRateLimiter(capacity=10, refill_rate=2)

for i in range(15):
    allowed = limiter.allow_request("user123")
    print(f"Request {i+1}: {'Allowed' if allowed else 'Rejected'}")
```

### Go

```go
package ratelimiter

import (
    "sync"
    "time"
)

// TokenBucket implements token bucket rate limiting
type TokenBucket struct {
    capacity   float64
    refillRate float64 // tokens per second
    buckets    map[string]*bucket
    mu         sync.Mutex
}

type bucket struct {
    tokens     float64
    lastRefill time.Time
}

// NewTokenBucket creates a new token bucket rate limiter
func NewTokenBucket(capacity int, refillRate float64) *TokenBucket {
    return &TokenBucket{
        capacity:   float64(capacity),
        refillRate: refillRate,
        buckets:    make(map[string]*bucket),
    }
}

// AllowRequest checks if request is allowed for client
func (tb *TokenBucket) AllowRequest(clientID string) bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    now := time.Now()

    // Get or create bucket
    b, exists := tb.buckets[clientID]
    if !exists {
        tb.buckets[clientID] = &bucket{
            tokens:     tb.capacity,
            lastRefill: now,
        }
        b = tb.buckets[clientID]
    }

    // Refill tokens
    elapsed := now.Sub(b.lastRefill).Seconds()
    tokensToAdd := elapsed * tb.refillRate
    b.tokens = min(tb.capacity, b.tokens+tokensToAdd)
    b.lastRefill = now

    // Check and consume token
    if b.tokens >= 1 {
        b.tokens--
        return true
    }

    return false
}

func min(a, b float64) float64 {
    if a < b {
        return a
    }
    return b
}
```

---

## Sliding Window Counter Implementation

### Python with Redis

```python
"""
Sliding Window Counter Rate Limiter

Algorithm: Combines fixed window efficiency with sliding window accuracy.
Uses weighted average of current and previous window counts.

Use case: General-purpose rate limiting with good accuracy and low memory.
"""

import time
import redis
from typing import Tuple


class SlidingWindowCounter:
    def __init__(
        self,
        redis_client: redis.Redis,
        limit: int = 100,
        window_seconds: int = 60
    ):
        """
        Initialize sliding window counter.

        Args:
            redis_client: Redis connection
            limit: Max requests per window
            window_seconds: Window size in seconds
        """
        self.redis = redis_client
        self.limit = limit
        self.window = window_seconds

    def allow_request(self, client_id: str) -> Tuple[bool, int]:
        """
        Check if request is allowed.

        Returns:
            Tuple of (is_allowed, remaining_requests)
        """
        now = time.time()

        # Calculate current and previous window keys
        current_window = int(now // self.window)
        previous_window = current_window - 1

        current_key = f"rate:{client_id}:{current_window}"
        previous_key = f"rate:{client_id}:{previous_window}"

        # Get counts (using pipeline for efficiency)
        pipe = self.redis.pipeline()
        pipe.get(current_key)
        pipe.get(previous_key)
        results = pipe.execute()

        current_count = int(results[0] or 0)
        previous_count = int(results[1] or 0)

        # Calculate position in current window (0.0 to 1.0)
        window_position = (now % self.window) / self.window

        # Weighted count
        weighted_count = (
            previous_count * (1 - window_position) +
            current_count
        )

        if weighted_count < self.limit:
            # Allow and increment
            pipe = self.redis.pipeline()
            pipe.incr(current_key)
            pipe.expire(current_key, self.window * 2)
            pipe.execute()

            remaining = int(self.limit - weighted_count - 1)
            return True, max(0, remaining)

        return False, 0

    def get_retry_after(self, client_id: str) -> int:
        """Get seconds until client can retry."""
        now = time.time()
        current_window = int(now // self.window)
        window_start = current_window * self.window
        window_end = window_start + self.window
        return int(window_end - now)


# Usage with Redis
redis_client = redis.Redis(host='localhost', port=6379, db=0)
limiter = SlidingWindowCounter(redis_client, limit=100, window_seconds=60)

allowed, remaining = limiter.allow_request("user123")
print(f"Allowed: {allowed}, Remaining: {remaining}")
```

### Lua Script for Atomic Operations

```lua
-- sliding_window.lua
-- Atomic sliding window counter in Redis
-- KEYS[1] = rate limit key prefix
-- ARGV[1] = limit
-- ARGV[2] = window size in seconds
-- ARGV[3] = current timestamp

local key_prefix = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local current_window = math.floor(now / window)
local previous_window = current_window - 1

local current_key = key_prefix .. ":" .. current_window
local previous_key = key_prefix .. ":" .. previous_window

-- Get counts
local current_count = tonumber(redis.call('GET', current_key) or 0)
local previous_count = tonumber(redis.call('GET', previous_key) or 0)

-- Calculate weighted count
local window_position = (now % window) / window
local weighted_count = (previous_count * (1 - window_position)) + current_count

if weighted_count < limit then
    -- Allow: increment and set expiry
    redis.call('INCR', current_key)
    redis.call('EXPIRE', current_key, window * 2)
    return {1, limit - weighted_count - 1}  -- allowed, remaining
else
    return {0, 0}  -- rejected, remaining
end
```

```python
# Using the Lua script in Python
import redis

class AtomicSlidingWindow:
    LUA_SCRIPT = """
    local key_prefix = KEYS[1]
    local limit = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])

    local current_window = math.floor(now / window)
    local previous_window = current_window - 1

    local current_key = key_prefix .. ":" .. current_window
    local previous_key = key_prefix .. ":" .. previous_window

    local current_count = tonumber(redis.call('GET', current_key) or 0)
    local previous_count = tonumber(redis.call('GET', previous_key) or 0)

    local window_position = (now % window) / window
    local weighted_count = (previous_count * (1 - window_position)) + current_count

    if weighted_count < limit then
        redis.call('INCR', current_key)
        redis.call('EXPIRE', current_key, window * 2)
        return {1, math.floor(limit - weighted_count - 1)}
    else
        return {0, 0}
    end
    """

    def __init__(self, redis_client: redis.Redis, limit: int, window: int):
        self.redis = redis_client
        self.limit = limit
        self.window = window
        self.script = self.redis.register_script(self.LUA_SCRIPT)

    def allow_request(self, client_id: str) -> tuple:
        import time
        result = self.script(
            keys=[f"rate:{client_id}"],
            args=[self.limit, self.window, time.time()]
        )
        return bool(result[0]), int(result[1])
```

---

## Rate Limiter Middleware

### FastAPI Middleware

```python
"""
Rate Limiting Middleware for FastAPI

Features:
- Multiple rate limiting strategies
- Per-client and per-endpoint limits
- Response headers with rate limit info
"""

from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware
import redis
import time
from typing import Optional, Callable


class RateLimitMiddleware(BaseHTTPMiddleware):
    def __init__(
        self,
        app: FastAPI,
        redis_url: str = "redis://localhost:6379",
        default_limit: int = 100,
        window_seconds: int = 60,
        key_func: Optional[Callable] = None
    ):
        super().__init__(app)
        self.redis = redis.from_url(redis_url)
        self.default_limit = default_limit
        self.window = window_seconds
        self.key_func = key_func or self._default_key_func

    def _default_key_func(self, request: Request) -> str:
        """Extract client identifier from request."""
        # Try API key first
        api_key = request.headers.get("X-API-Key")
        if api_key:
            return f"api:{api_key}"

        # Fall back to IP
        forwarded = request.headers.get("X-Forwarded-For")
        if forwarded:
            return f"ip:{forwarded.split(',')[0].strip()}"

        return f"ip:{request.client.host}"

    async def dispatch(self, request: Request, call_next):
        # Skip rate limiting for certain paths
        if request.url.path in ["/health", "/metrics"]:
            return await call_next(request)

        client_id = self.key_func(request)

        # Check rate limit
        allowed, remaining, reset_time = self._check_limit(client_id)

        if not allowed:
            return JSONResponse(
                status_code=429,
                content={"error": "Too Many Requests"},
                headers={
                    "X-RateLimit-Limit": str(self.default_limit),
                    "X-RateLimit-Remaining": "0",
                    "X-RateLimit-Reset": str(reset_time),
                    "Retry-After": str(reset_time - int(time.time()))
                }
            )

        # Process request
        response = await call_next(request)

        # Add rate limit headers
        response.headers["X-RateLimit-Limit"] = str(self.default_limit)
        response.headers["X-RateLimit-Remaining"] = str(remaining)
        response.headers["X-RateLimit-Reset"] = str(reset_time)

        return response

    def _check_limit(self, client_id: str) -> tuple:
        """Check if request is within rate limit."""
        now = time.time()
        window_start = int(now // self.window) * self.window
        reset_time = int(window_start + self.window)

        key = f"ratelimit:{client_id}:{window_start}"

        # Atomic increment
        pipe = self.redis.pipeline()
        pipe.incr(key)
        pipe.expire(key, self.window + 1)
        count, _ = pipe.execute()

        remaining = max(0, self.default_limit - count)
        allowed = count <= self.default_limit

        return allowed, remaining, reset_time


# Usage
app = FastAPI()
app.add_middleware(
    RateLimitMiddleware,
    redis_url="redis://localhost:6379",
    default_limit=100,
    window_seconds=60
)

@app.get("/api/resource")
async def get_resource():
    return {"data": "Hello World"}
```

### Express.js Middleware

```javascript
/**
 * Rate Limiting Middleware for Express
 *
 * Uses sliding window counter with Redis
 */

const Redis = require('ioredis');

class RateLimiter {
  constructor(options = {}) {
    this.redis = new Redis(options.redisUrl || 'redis://localhost:6379');
    this.limit = options.limit || 100;
    this.window = options.windowSeconds || 60;
  }

  /**
   * Check rate limit for client
   * @param {string} clientId
   * @returns {Promise<{allowed: boolean, remaining: number, resetTime: number}>}
   */
  async checkLimit(clientId) {
    const now = Date.now() / 1000;
    const currentWindow = Math.floor(now / this.window);
    const previousWindow = currentWindow - 1;

    const currentKey = `rate:${clientId}:${currentWindow}`;
    const previousKey = `rate:${clientId}:${previousWindow}`;

    // Get counts
    const [currentCount, previousCount] = await this.redis.mget(
      currentKey,
      previousKey
    );

    const current = parseInt(currentCount) || 0;
    const previous = parseInt(previousCount) || 0;

    // Calculate weighted count
    const windowPosition = (now % this.window) / this.window;
    const weightedCount = previous * (1 - windowPosition) + current;

    const resetTime = Math.ceil((currentWindow + 1) * this.window);

    if (weightedCount < this.limit) {
      // Allow and increment
      await this.redis
        .multi()
        .incr(currentKey)
        .expire(currentKey, this.window * 2)
        .exec();

      return {
        allowed: true,
        remaining: Math.floor(this.limit - weightedCount - 1),
        resetTime,
      };
    }

    return {
      allowed: false,
      remaining: 0,
      resetTime,
    };
  }

  /**
   * Express middleware
   */
  middleware() {
    return async (req, res, next) => {
      // Extract client ID
      const clientId =
        req.headers['x-api-key'] ||
        req.headers['x-forwarded-for']?.split(',')[0] ||
        req.ip;

      try {
        const { allowed, remaining, resetTime } = await this.checkLimit(
          clientId
        );

        // Set headers
        res.set({
          'X-RateLimit-Limit': this.limit,
          'X-RateLimit-Remaining': remaining,
          'X-RateLimit-Reset': resetTime,
        });

        if (!allowed) {
          res.set('Retry-After', Math.ceil(resetTime - Date.now() / 1000));
          return res.status(429).json({ error: 'Too Many Requests' });
        }

        next();
      } catch (error) {
        // Fail open on Redis errors
        console.error('Rate limiter error:', error);
        next();
      }
    };
  }
}

// Usage
const express = require('express');
const app = express();

const rateLimiter = new RateLimiter({
  redisUrl: 'redis://localhost:6379',
  limit: 100,
  windowSeconds: 60,
});

app.use(rateLimiter.middleware());

app.get('/api/resource', (req, res) => {
  res.json({ data: 'Hello World' });
});

app.listen(3000);
```

---

## Distributed Rate Limiter

### Python with Redis Cluster

```python
"""
Distributed Rate Limiter

Works across multiple servers using Redis Cluster.
Handles network partitions with fail-open strategy.
"""

import time
import redis
from redis.cluster import RedisCluster
from typing import Optional
import logging

logger = logging.getLogger(__name__)


class DistributedRateLimiter:
    def __init__(
        self,
        redis_nodes: list,
        limit: int = 100,
        window: int = 60,
        fail_open: bool = True
    ):
        """
        Initialize distributed rate limiter.

        Args:
            redis_nodes: List of Redis cluster nodes
            limit: Requests per window
            window: Window size in seconds
            fail_open: Allow requests if Redis unavailable
        """
        self.redis = RedisCluster(
            startup_nodes=redis_nodes,
            decode_responses=True
        )
        self.limit = limit
        self.window = window
        self.fail_open = fail_open

        # Register Lua script
        self.script = self._register_script()

    def _register_script(self):
        lua_script = """
        local key = KEYS[1]
        local limit = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])

        local current_window = math.floor(now / window)
        local current_key = key .. ":" .. current_window
        local previous_key = key .. ":" .. (current_window - 1)

        local current = tonumber(redis.call('GET', current_key) or 0)
        local previous = tonumber(redis.call('GET', previous_key) or 0)

        local position = (now % window) / window
        local weighted = (previous * (1 - position)) + current

        if weighted < limit then
            redis.call('INCR', current_key)
            redis.call('EXPIRE', current_key, window * 2)
            return {1, math.floor(limit - weighted - 1)}
        end
        return {0, 0}
        """
        return self.redis.register_script(lua_script)

    def allow_request(
        self,
        client_id: str,
        cost: int = 1
    ) -> tuple[bool, int, int]:
        """
        Check if request is allowed.

        Args:
            client_id: Unique client identifier
            cost: Number of tokens to consume (for weighted limiting)

        Returns:
            Tuple of (allowed, remaining, retry_after_seconds)
        """
        try:
            now = time.time()
            result = self.script(
                keys=[f"ratelimit:{{{client_id}}}"],  # Hash tag for cluster
                args=[self.limit, self.window, now]
            )

            allowed = bool(result[0])
            remaining = int(result[1])

            # Calculate retry time
            window_end = (int(now // self.window) + 1) * self.window
            retry_after = int(window_end - now)

            return allowed, remaining, retry_after

        except redis.RedisError as e:
            logger.error(f"Redis error in rate limiter: {e}")

            if self.fail_open:
                # Allow request on Redis failure
                return True, self.limit, 0
            else:
                # Reject request on Redis failure
                return False, 0, self.window


# Usage
nodes = [
    {"host": "redis-1", "port": 6379},
    {"host": "redis-2", "port": 6379},
    {"host": "redis-3", "port": 6379},
]

limiter = DistributedRateLimiter(
    redis_nodes=nodes,
    limit=1000,
    window=60,
    fail_open=True
)

allowed, remaining, retry_after = limiter.allow_request("client-123")
```

---

## Testing Rate Limiter

```python
"""
Unit tests for rate limiter implementations
"""

import pytest
import time
from unittest.mock import Mock, patch
import fakeredis


class TestTokenBucket:
    def test_allows_requests_within_capacity(self):
        limiter = TokenBucketRateLimiter(capacity=5, refill_rate=1)

        for i in range(5):
            assert limiter.allow_request("client1") is True

    def test_rejects_when_empty(self):
        limiter = TokenBucketRateLimiter(capacity=2, refill_rate=0.1)

        assert limiter.allow_request("client1") is True
        assert limiter.allow_request("client1") is True
        assert limiter.allow_request("client1") is False

    def test_refills_over_time(self):
        limiter = TokenBucketRateLimiter(capacity=2, refill_rate=2)

        # Empty the bucket
        limiter.allow_request("client1")
        limiter.allow_request("client1")
        assert limiter.allow_request("client1") is False

        # Wait for refill
        time.sleep(0.6)  # Should add ~1 token

        assert limiter.allow_request("client1") is True


class TestSlidingWindowCounter:
    @pytest.fixture
    def redis_client(self):
        return fakeredis.FakeRedis()

    def test_allows_under_limit(self, redis_client):
        limiter = SlidingWindowCounter(redis_client, limit=10, window_seconds=60)

        for i in range(10):
            allowed, remaining = limiter.allow_request("client1")
            assert allowed is True
            assert remaining == 10 - i - 1

    def test_rejects_over_limit(self, redis_client):
        limiter = SlidingWindowCounter(redis_client, limit=5, window_seconds=60)

        for i in range(5):
            limiter.allow_request("client1")

        allowed, remaining = limiter.allow_request("client1")
        assert allowed is False
        assert remaining == 0

    def test_separate_clients(self, redis_client):
        limiter = SlidingWindowCounter(redis_client, limit=2, window_seconds=60)

        limiter.allow_request("client1")
        limiter.allow_request("client1")

        # Client 2 should have its own limit
        allowed, _ = limiter.allow_request("client2")
        assert allowed is True


# Load test
def test_concurrent_requests():
    """Test rate limiter under concurrent load."""
    import threading
    import redis

    client = redis.Redis()
    limiter = SlidingWindowCounter(client, limit=100, window_seconds=1)

    results = {"allowed": 0, "rejected": 0}
    lock = threading.Lock()

    def make_request():
        allowed, _ = limiter.allow_request("load_test_client")
        with lock:
            if allowed:
                results["allowed"] += 1
            else:
                results["rejected"] += 1

    threads = [threading.Thread(target=make_request) for _ in range(200)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

    # Should allow ~100 requests
    assert 95 <= results["allowed"] <= 105
```

---

## Docker Compose Setup

```yaml
services:
  rate-limiter:
    build: .
    ports:
      - '8080:8080'
    environment:
      - REDIS_URL=redis://redis:6379
      - RATE_LIMIT=100
      - RATE_WINDOW=60
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

  # For testing with Redis Cluster
  redis-node-1:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf
    ports:
      - '7001:6379'

  redis-node-2:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf
    ports:
      - '7002:6379'

  redis-node-3:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf
    ports:
      - '7003:6379'

volumes:
  redis_data:
```
