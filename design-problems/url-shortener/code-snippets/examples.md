# URL Shortener - Code Snippets

## Base62 Encoding

### Python Implementation

```python
import string

class Base62Encoder:
    CHARSET = string.digits + string.ascii_letters  # 0-9, A-Z, a-z

    @classmethod
    def encode(cls, num: int) -> str:
        """Encode a number to base62 string."""
        if num == 0:
            return cls.CHARSET[0]

        result = []
        base = len(cls.CHARSET)

        while num > 0:
            result.append(cls.CHARSET[num % base])
            num //= base

        return ''.join(reversed(result))

    @classmethod
    def decode(cls, encoded: str) -> int:
        """Decode a base62 string to number."""
        num = 0
        base = len(cls.CHARSET)

        for char in encoded:
            num = num * base + cls.CHARSET.index(char)

        return num


# Usage
encoder = Base62Encoder()
print(encoder.encode(1000000))  # "4c92"
print(encoder.decode("4c92"))   # 1000000
```

### Go Implementation

```go
package shortener

import (
    "strings"
)

const charset = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"

func Encode(num uint64) string {
    if num == 0 {
        return string(charset[0])
    }

    var result strings.Builder
    base := uint64(len(charset))

    for num > 0 {
        result.WriteByte(charset[num%base])
        num /= base
    }

    // Reverse the string
    runes := []rune(result.String())
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }

    return string(runes)
}

func Decode(encoded string) uint64 {
    var num uint64
    base := uint64(len(charset))

    for _, char := range encoded {
        num = num*base + uint64(strings.IndexRune(charset, char))
    }

    return num
}
```

---

## URL Shortener Service

### Python (FastAPI)

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, HttpUrl
import redis
import hashlib
from typing import Optional
import os

app = FastAPI()
redis_client = redis.Redis(host='localhost', port=6379, db=0)

class URLRequest(BaseModel):
    long_url: HttpUrl
    custom_alias: Optional[str] = None

class URLResponse(BaseModel):
    short_url: str
    long_url: str

class Counter:
    """Distributed counter using Redis."""

    @staticmethod
    def get_next_id() -> int:
        return redis_client.incr("url_counter")

def generate_short_code(url: str, custom: Optional[str] = None) -> str:
    if custom:
        # Validate custom alias
        if redis_client.exists(f"url:{custom}"):
            raise HTTPException(status_code=409, detail="Alias already exists")
        return custom

    # Use counter-based generation
    counter = Counter.get_next_id()
    return Base62Encoder.encode(counter)

@app.post("/api/v1/urls", response_model=URLResponse)
async def create_short_url(request: URLRequest):
    long_url = str(request.long_url)

    # Check if URL already shortened
    existing = redis_client.get(f"reverse:{long_url}")
    if existing:
        short_code = existing.decode()
        return URLResponse(
            short_url=f"https://short.ly/{short_code}",
            long_url=long_url
        )

    # Generate new short code
    short_code = generate_short_code(long_url, request.custom_alias)

    # Store mappings
    redis_client.set(f"url:{short_code}", long_url)
    redis_client.set(f"reverse:{long_url}", short_code)

    return URLResponse(
        short_url=f"https://short.ly/{short_code}",
        long_url=long_url
    )

@app.get("/{short_code}")
async def redirect(short_code: str):
    long_url = redis_client.get(f"url:{short_code}")

    if not long_url:
        raise HTTPException(status_code=404, detail="URL not found")

    # Log analytics asynchronously (in production)
    # analytics_queue.put({"short_code": short_code, "timestamp": time.time()})

    from fastapi.responses import RedirectResponse
    return RedirectResponse(url=long_url.decode(), status_code=301)
```

### Go (Gin)

```go
package main

import (
    "context"
    "net/http"

    "github.com/gin-gonic/gin"
    "github.com/go-redis/redis/v8"
)

var (
    ctx = context.Background()
    rdb *redis.Client
)

type URLRequest struct {
    LongURL     string `json:"long_url" binding:"required,url"`
    CustomAlias string `json:"custom_alias,omitempty"`
}

type URLResponse struct {
    ShortURL string `json:"short_url"`
    LongURL  string `json:"long_url"`
}

func init() {
    rdb = redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
        DB:   0,
    })
}

func generateShortCode(customAlias string) (string, error) {
    if customAlias != "" {
        exists, _ := rdb.Exists(ctx, "url:"+customAlias).Result()
        if exists > 0 {
            return "", fmt.Errorf("alias already exists")
        }
        return customAlias, nil
    }

    counter, err := rdb.Incr(ctx, "url_counter").Result()
    if err != nil {
        return "", err
    }

    return Encode(uint64(counter)), nil
}

func createShortURL(c *gin.Context) {
    var req URLRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Check if already exists
    existing, _ := rdb.Get(ctx, "reverse:"+req.LongURL).Result()
    if existing != "" {
        c.JSON(http.StatusOK, URLResponse{
            ShortURL: "https://short.ly/" + existing,
            LongURL:  req.LongURL,
        })
        return
    }

    shortCode, err := generateShortCode(req.CustomAlias)
    if err != nil {
        c.JSON(http.StatusConflict, gin.H{"error": err.Error()})
        return
    }

    // Store mappings
    rdb.Set(ctx, "url:"+shortCode, req.LongURL, 0)
    rdb.Set(ctx, "reverse:"+req.LongURL, shortCode, 0)

    c.JSON(http.StatusCreated, URLResponse{
        ShortURL: "https://short.ly/" + shortCode,
        LongURL:  req.LongURL,
    })
}

func redirect(c *gin.Context) {
    shortCode := c.Param("code")

    longURL, err := rdb.Get(ctx, "url:"+shortCode).Result()
    if err == redis.Nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "URL not found"})
        return
    }

    c.Redirect(http.StatusMovedPermanently, longURL)
}

func main() {
    r := gin.Default()

    r.POST("/api/v1/urls", createShortURL)
    r.GET("/:code", redirect)

    r.Run(":8080")
}
```

---

## ID Generation Service

### Snowflake-like ID Generator

```python
import time
import threading

class SnowflakeIDGenerator:
    """
    Generates unique IDs inspired by Twitter's Snowflake.

    Structure (64 bits):
    - 1 bit: sign (always 0)
    - 41 bits: timestamp (ms since epoch)
    - 10 bits: machine ID (1024 machines)
    - 12 bits: sequence (4096 per ms per machine)
    """

    EPOCH = 1704067200000  # Jan 1, 2024

    def __init__(self, machine_id: int):
        if machine_id < 0 or machine_id > 1023:
            raise ValueError("Machine ID must be 0-1023")

        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = -1
        self.lock = threading.Lock()

    def _current_timestamp(self) -> int:
        return int(time.time() * 1000)

    def generate(self) -> int:
        with self.lock:
            timestamp = self._current_timestamp()

            if timestamp == self.last_timestamp:
                self.sequence = (self.sequence + 1) & 0xFFF  # 12 bits
                if self.sequence == 0:
                    # Wait for next millisecond
                    while timestamp <= self.last_timestamp:
                        timestamp = self._current_timestamp()
            else:
                self.sequence = 0

            self.last_timestamp = timestamp

            # Compose ID
            id = ((timestamp - self.EPOCH) << 22) | \
                 (self.machine_id << 12) | \
                 self.sequence

            return id


# Usage
generator = SnowflakeIDGenerator(machine_id=1)
for _ in range(5):
    id = generator.generate()
    short_code = Base62Encoder.encode(id)
    print(f"ID: {id}, Short Code: {short_code}")
```

---

## Database Schema (PostgreSQL)

```sql
-- Main URL table
CREATE TABLE urls (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE,
    user_id UUID,
    is_active BOOLEAN DEFAULT TRUE
);

-- Indexes
CREATE INDEX idx_urls_short_code ON urls(short_code);
CREATE INDEX idx_urls_expires_at ON urls(expires_at)
    WHERE expires_at IS NOT NULL;
CREATE INDEX idx_urls_long_url ON urls USING hash(long_url);

-- Analytics table (separate for write performance)
CREATE TABLE url_clicks (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) NOT NULL,
    clicked_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    ip_address INET,
    user_agent TEXT,
    referrer TEXT,
    country_code CHAR(2)
);

-- Partitioned by month for easier management
CREATE TABLE url_clicks_2024_01 PARTITION OF url_clicks
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Function to get or create short URL
CREATE OR REPLACE FUNCTION get_or_create_short_url(
    p_long_url TEXT,
    p_short_code VARCHAR(10)
) RETURNS VARCHAR(10) AS $$
DECLARE
    v_existing VARCHAR(10);
BEGIN
    -- Check if exists
    SELECT short_code INTO v_existing
    FROM urls
    WHERE long_url = p_long_url AND is_active = TRUE;

    IF v_existing IS NOT NULL THEN
        RETURN v_existing;
    END IF;

    -- Insert new
    INSERT INTO urls (short_code, long_url)
    VALUES (p_short_code, p_long_url)
    ON CONFLICT (short_code) DO NOTHING;

    RETURN p_short_code;
END;
$$ LANGUAGE plpgsql;
```

---

## Redis Caching Layer

```python
import redis
from typing import Optional
import json

class URLCache:
    def __init__(self, host='localhost', port=6379):
        self.client = redis.Redis(host=host, port=port, decode_responses=True)
        self.default_ttl = 86400  # 24 hours

    def get_long_url(self, short_code: str) -> Optional[str]:
        """Get long URL from cache."""
        return self.client.get(f"url:{short_code}")

    def set_url_mapping(self, short_code: str, long_url: str, ttl: int = None):
        """Cache URL mapping."""
        ttl = ttl or self.default_ttl
        pipe = self.client.pipeline()
        pipe.setex(f"url:{short_code}", ttl, long_url)
        pipe.setex(f"reverse:{long_url}", ttl, short_code)
        pipe.execute()

    def increment_clicks(self, short_code: str):
        """Increment click counter (for real-time stats)."""
        self.client.incr(f"clicks:{short_code}")

    def get_click_count(self, short_code: str) -> int:
        """Get click count."""
        count = self.client.get(f"clicks:{short_code}")
        return int(count) if count else 0

    def check_rate_limit(self, ip: str, limit: int = 100) -> bool:
        """Check if IP is rate limited (100 requests/hour)."""
        key = f"ratelimit:{ip}"
        current = self.client.incr(key)
        if current == 1:
            self.client.expire(key, 3600)  # 1 hour window
        return current <= limit
```

---

## Load Testing Script

```python
import asyncio
import aiohttp
import time
import random
import string

async def create_url(session, base_url):
    """Create a short URL."""
    long_url = f"https://example.com/{''.join(random.choices(string.ascii_letters, k=20))}"
    async with session.post(
        f"{base_url}/api/v1/urls",
        json={"long_url": long_url}
    ) as response:
        return await response.json()

async def redirect_url(session, base_url, short_code):
    """Test redirect."""
    async with session.get(
        f"{base_url}/{short_code}",
        allow_redirects=False
    ) as response:
        return response.status

async def load_test(base_url, num_creates=100, num_redirects=1000):
    """Run load test."""
    async with aiohttp.ClientSession() as session:
        # Create URLs
        print(f"Creating {num_creates} URLs...")
        start = time.time()
        create_tasks = [create_url(session, base_url) for _ in range(num_creates)]
        results = await asyncio.gather(*create_tasks)
        create_time = time.time() - start
        print(f"Created {num_creates} URLs in {create_time:.2f}s ({num_creates/create_time:.0f} req/s)")

        # Extract short codes
        short_codes = [r['short_url'].split('/')[-1] for r in results if 'short_url' in r]

        # Test redirects
        print(f"Testing {num_redirects} redirects...")
        start = time.time()
        redirect_tasks = [
            redirect_url(session, base_url, random.choice(short_codes))
            for _ in range(num_redirects)
        ]
        await asyncio.gather(*redirect_tasks)
        redirect_time = time.time() - start
        print(f"Completed {num_redirects} redirects in {redirect_time:.2f}s ({num_redirects/redirect_time:.0f} req/s)")

if __name__ == "__main__":
    asyncio.run(load_test("http://localhost:8080"))
```

---

## Docker Compose Setup

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - REDIS_HOST=redis
      - POSTGRES_HOST=postgres
    depends_on:
      - redis
      - postgres

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: urlshortener
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - api

volumes:
  redis_data:
  postgres_data:
```
