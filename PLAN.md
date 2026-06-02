# 🏆 The 500MB Club — Go 1.26 Victory Plan (Stdlib-First)

> Deadline: **July 26, 2026, 22:00 UTC**  
> Target Go: **1.26.3** (latest stable)  
> Philosophy: **Standard library everywhere possible; minimal, justified external deps**

---

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│           Go 1.26 + net/http + Redis (custom client)       │
│                                                            │
│  HTTP:    net/http with enhanced ServeMux (Go 1.22+)      │
│  JSON:    encoding/json (stdlib)                          │
│  Logging: log/slog (stdlib, structured)                   │
│  Redis:   minimal custom RESP2 client (~200 lines)        │
│  Metrics: hand-rolled Prometheus text format              │
│                                                            │
│  External dependencies: NONE                              │
└────────────────────────────────────────────────────────────┘
```

| Component | Choice | Rationale |
|-----------|--------|-----------|
| HTTP | `net/http` | Enhanced ServeMux with method + path param routing. No chi/fasthttp. |
| JSON | `encoding/json` | Good enough at 200 RPS. `json.RawMessage` for fast validation. |
| Routing | `http.NewServeMux` (1.22+) | `mux.HandleFunc("POST /devices/{id}/telemetry", h)` |
| Redis | Custom RESP2 client | 200 lines, zero-alloc, exactly what we need. No bloat. |
| Logging | `log/slog` | Structured, zero-cost when disabled. |
| Metrics | Manual Prometheus format | `/metrics` in `text/plain; version=0.0.4` |

## Go 1.26 Modern Features (JetBrains Guidelines Compliant)

| Feature | Use Case |
|---------|----------|
| `new(val)` | `new(202)` for status codes, pointer fields in configs |
| `errors.AsType[T](err)` | Type-safe error inspection |
| `wg.Go(fn)` | Graceful shutdown goroutines |
| `for i := range n` | All index loops |
| `cmp.Or(a, b, "default")` | Query param defaults, env vars |
| `min(a,b)` / `max(a,b)` | Bounds checking |
| `clear(m)` | Reset pooled maps |
| `slices.Contains`, `slices.SortFunc` | Collection ops |
| `maps.Clone`, `maps.Keys` | Map ops |
| `strings.CutPrefix` | Header/path parsing |
| `strings.SplitSeq` | No-alloc string iteration |
| `omitzero` | JSON struct tags (correct for time.Duration) |
| `t.Context()` | Test contexts |
| `b.Loop()` | Benchmark loops |

## Scoring Strategy

### Score Formula
```
score = 100 × Σ(weight[d] × dim[d])

Where dim[d] = mean(clip(ratio_of_each_metric, clip_min[d], clip_max[d]))

Score ceiling = 304, floor = 25. "Meets target" = 100.
```

| Dimension | Weight | Our Target | Expected Score | Max Contribution |
|-----------|--------|-----------|---------------|-----------------|
| **Efficiency** | 32% | RSS ~60MB, CPU ~12% | 3.6–3.9 | 128 |
| **Capacity** | 27% | 1300+ RPS | 1.30 | 108 |
| **Tail Latency** | 20% | All ops < targets | 1.50 (saturated) | 30 |
| **Resilience** | 13% | Spike p99 < 6ms, 0% errors | 1.96–2.00 | 26 |
| **Stability** | 8% | No drift | 1.15–1.25 | 12 |
| **TOTAL** | | | **~215–235** | **304** |

### Gate Requirements (must pass)
- Sustain ~200 RPS with `http_req_failed` < 0.5%
- Aggregate p95 RSS < 500 MB
- Aggregate CPU < 200% (2 cores)

## Project Structure

```
the_500mb_club_go/
├── cmd/
│   └── api/
│       └── main.go                 # Entry point, server setup, graceful shutdown
├── internal/
│   ├── handler/
│   │   ├── handler.go              # HTTP handlers, ServeMux routing, middleware
│   │   ├── handler_test.go         # Unit tests
│   │   ├── handler_bench_test.go   # Benchmarks
│   │   ├── telemetry.go            # POST single + batch ingest
│   │   ├── telemetry_test.go       # Unit + table-driven tests
│   │   ├── query.go                # GET time-window + cursor pagination
│   │   ├── query_test.go           # Unit tests
│   │   ├── anomaly.go              # GET anomaly z-score computation
│   │   ├── anomaly_test.go         # Unit tests + edge cases
│   │   ├── health.go               # healthz, readyz, metrics
│   │   └── health_test.go          # Unit tests
│   ├── redis/
│   │   ├── client.go               # Minimal RESP2 Redis client
│   │   ├── client_test.go          # Integration tests (require Redis)
│   │   ├── pool.go                 # Connection pool
│   │   ├── pool_test.go            # Pool tests
│   │   └── pipeline.go             # Pipeline for batch operations
│   ├── telemetry/
│   │   ├── point.go                # TelemetryPoint type + validation
│   │   ├── point_test.go           # Validation tests
│   │   ├── encode.go               # Compact binary encoding (56 bytes)
│   │   ├── encode_test.go          # Round-trip encode/decode tests
│   │   ├── store.go                # Redis storage operations
│   │   └── store_test.go           # Storage integration tests
│   ├── anomaly/
│   │   ├── zscore.go               # Welford's single-pass algorithm
│   │   └── zscore_test.go          # Tests with known distributions
│   └── middleware/
│       ├── instance.go             # X-Instance-Id header injection
│       ├── logging.go              # Request logging via slog
│       └── middleware_test.go      # Middleware unit tests
├── stress/
│   ├── stress_test.go              # End-to-end stress tests
│   └── README.md                   # How to run stress tests
├── testdata/
│   └── telemetry_points.json       # Sample data for tests
├── nginx.conf                       # Round-robin load balancer
├── docker-compose.yml               # 3×API + Redis + Nginx
├── Dockerfile                       # Multi-stage: build → scratch
├── me.json                          # Team info for submission
├── go.mod                           # module + go 1.26, zero deps
├── go.sum
├── Makefile                         # Build, test, bench, stress targets
└── README.md                        # Project docs
```

## Implementation Details

### 1. Custom RESP2 Redis Client (`internal/redis/`)

~200 lines. Supports exactly what we need:

```go
// Commands we implement:
//   ZADD key score member
//   ZRANGEBYSCORE key min max [LIMIT offset count]
//   ZREVRANGE key start stop [WITHSCORES]
//   PING
//   MULTI/EXEC (pipeline)

type Client struct { pool *Pool }

func (c *Client) ZADD(ctx context.Context, key string, score int64, member []byte) error
func (c *Client) ZRANGEBYSCORE(ctx context.Context, key string, min, max int64, offset, count int) ([][]byte, error)
func (c *Client) ZREVRANGE(ctx context.Context, key string, start, stop int) ([][]byte, error)
func (c *Client) Pipeline(ctx context.Context) *Pipeline
```

RESP2 protocol is simple:
- Send: `*3\r\n$4\r\nZADD\r\n$8\r\nkey...\r\n$3\r\n...\r\n`
- Receive: `:1\r\n` (integer) or `*3\r\n$...\r\n...` (array)

### 2. Compact Point Encoding (`internal/telemetry/`)

```go
// 56 bytes per point (vs ~150 bytes JSON → saves 62% Redis memory)
// Layout: [ts:8][lat:8][lon:8][battery:8][ax:8][ay:8][az:8]
const PointSize = 56

func EncodePoint(p TelemetryPoint) []byte
func DecodePoint(b []byte) TelemetryPoint
func EncodePointToJSON(b []byte) []byte  // binary → JSON (for API response)
```

### 3. Anomaly Detection (`internal/anomaly/`)

Single-pass Welford's online algorithm. No caching (as required).

```go
func Compute(points []TelemetryPoint) Result {
    // Welford's algorithm: single pass, O(n), stable
    // Returns: z_score, samples, anomalous (|z| > 3), mean, stddev
}
```

### 4. GC Tuning (in `main.go`)

```go
debug.SetMemoryLimit(70 << 20)   // 70 MiB per API instance
debug.SetGCPercent(-1)            // disable automatic GC

// Periodic manual GC to return memory to OS
go func() {
    for range time.NewTicker(5 * time.Second).C {
        runtime.GC()
        debug.FreeOSMemory()
    }
}()
```

### 5. Pool Allocation Strategy

```go
var (
    bufPool   = sync.Pool{New: func() any { return make([]byte, 0, 4096) }}
    pointPool = sync.Pool{New: func() any { b := make([]byte, 56); return &b }}
    jsonPool  = sync.Pool{New: func() any { return &bytes.Buffer{} }}
)
```

## Testing Strategy

### Unit Tests (table-driven, modern Go)

```go
func TestTelemetryPointValidation(t *testing.T) {
    ctx := t.Context()  // Go 1.24+
    tests := []struct {
        name    string
        point   TelemetryPoint
        wantErr bool
    }{
        {"valid", validPoint(), false},
        {"missing ts", pointWithoutTS(), true},
        {"lat out of range", pointWithLat(200), true},
        {"lon out of range", pointWithLon(200), true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := tt.point.Validate()
            if (err != nil) != tt.wantErr {
                t.Errorf("Validate() error = %v, wantErr = %v", err, tt.wantErr)
            }
        })
    }
}
```

### Benchmark Tests

```go
func BenchmarkAnomalyCompute(b *testing.B) {
    points := generate256Points()
    for b.Loop() {  // Go 1.25+
        Compute(points)
    }
}

func BenchmarkEncodeDecode(b *testing.B) {
    p := validPoint()
    for b.Loop() {
        buf := EncodePoint(p)
        _ = DecodePoint(buf)
    }
}

func BenchmarkPostTelemetry(b *testing.B) {
    // HTTP handler benchmark
    handler := setupTestHandler()
    req := newPostTelemetryRequest()
    for b.Loop() {
        rr := httptest.NewRecorder()
        handler.ServeHTTP(rr, req)
    }
}
```

### Integration Tests (require Redis)

```go
func TestRedisZADDZRANGE(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    client := redis.NewClient("localhost:6379")
    key := fmt.Sprintf("test:%d", time.Now().UnixNano())
    
    // ZADD 256 points
    for i := range 256 {
        p := generatePoint(int64(i * 1000))
        err := client.ZADD(t.Context(), key, p.TS, EncodePoint(p))
        if err != nil {
            t.Fatalf("ZADD: %v", err)
        }
    }
    
    // ZRANGEBYSCORE
    points, err := client.ZRANGEBYSCORE(t.Context(), key, 0, 255000, 0, 100)
    if err != nil {
        t.Fatalf("ZRANGEBYSCORE: %v", err)
    }
    if len(points) != 100 {
        t.Errorf("got %d points, want 100", len(points))
    }
}
```

### Stress Tests (`stress/`)

```go
// stress/stress_test.go
// Run with: go test -tags=stress -count=1 -timeout=30m ./stress/

func TestStressConcurrentWrites(t *testing.T) {
    // 10 goroutines × 1000 writes
    var wg sync.WaitGroup
    for range 10 {
        wg.Go(func() {  // Go 1.25+
            for range 1000 {
                // POST /devices/{id}/telemetry
            }
        })
    }
    wg.Wait()
}

func TestStressSustainedLoad(t *testing.T) {
    // 200 RPS for 60 seconds
    start := time.Now()
    for time.Since(start) < 60*time.Second {
        // fire request
        // check latency
    }
}

func TestStressSpike(t *testing.T) {
    // Ramp 50 → 800 RPS
}

func TestStressEndurance(t *testing.T) {
    // 45 min sustained load, check no memory leak
}
```

### Makefile Targets

```makefile
.PHONY: test test-unit test-integration test-stress bench lint build docker-build smoke

test: test-unit test-integration

test-unit:
	go test -count=1 -short -race ./internal/...

test-integration:
	go test -count=1 -run Integration ./internal/...

test-stress:
	go test -tags=stress -count=1 -timeout=30m ./stress/

bench:
	go test -bench=. -benchmem -benchtime=10s ./internal/...

bench-profile:
	go test -bench=. -benchmem -cpuprofile=cpu.prof -memprofile=mem.prof ./internal/...

lint:
	go vet ./...
	staticcheck ./...

build:
	CGO_ENABLED=0 go build -ldflags="-s -w" -o bin/api ./cmd/api/

docker-build:
	docker buildx build --platform linux/arm64 -t api:latest .

smoke:
	# Requires docker-compose up first
	k6 run test/smoke.js

test-load:
	k6 run test/test.js
```

## docker-compose.yml (Budget Allocation)

```yaml
services:
  api-1: &api
    image: ghcr.io/you/api:latest
    user: "10001:10001"
    mem_limit: 85m
    cpus: 0.55
    environment:
      - INSTANCE_ID=api-1
      - REDIS_ADDR=storage:6379
      - GOMEMLIMIT=75MiB
    networks: [backend]
    depends_on:
      storage:
        condition: service_healthy

  api-2: *api
    environment:
      - INSTANCE_ID=api-2
      - REDIS_ADDR=storage:6379
      - GOMEMLIMIT=75MiB

  api-3: *api
    environment:
      - INSTANCE_ID=api-3
      - REDIS_ADDR=storage:6379
      - GOMEMLIMIT=75MiB

  lb:
    image: nginx:1.27-alpine
    ports: ["8080:80"]
    mem_limit: 20m
    cpus: 0.15
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks: [backend]
    depends_on: [api-1, api-2, api-3]

  storage:
    image: redis:7-alpine
    mem_limit: 50m
    cpus: 0.20
    networks: [backend]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 1s
    command: redis-server --maxmemory 40mb --maxmemory-policy noeviction --save ""

networks:
  backend:
    internal: true
```

**Budget: 85×3 + 20 + 50 = 325 MB max (well under 500 MB cap)**

## Performance Targets

| Metric | Target | How |
|--------|--------|-----|
| RSS p95 (aggregate) | **55–70 MB** | GC=off + GOMEMLIMIT + binary encoding |
| CPU avg (aggregate) | **10–15%** | Zero-alloc hot path, minimal JSON decode |
| Max sustained RPS | **1300–1500** | Redis pipelines + sorted sets |
| p99 POST | <3 ms | Single ZADD |
| p99 Batch | <5 ms | Single pipeline (100 ZADD, 1 RTT) |
| p99 Range | <5 ms | ZRANGEBYSCORE O(log N) |
| p99 Anomaly | <8 ms | One ZREVRANGE + single-pass Welford |
| Spike p99 | <6 ms | No GC pauses + bounded concurrency |
| Image size | <4 MB | Scratch + stripped binary |

## Execution Roadmap

### Phase 1: Foundation (Days 1–2)
- [ ] `go mod init` with `go 1.26`, zero external deps
- [ ] Custom RESP2 Redis client + pool + pipeline
- [ ] Telemetry point type + validation + binary encode/decode
- [ ] HTTP server with enhanced ServeMux + all route stubs
- [ ] X-Instance-Id middleware
- [ ] `healthz`, `readyz` (Redis ping)

### Phase 2: Core Features (Days 3–5)
- [ ] POST single ingest (ZADD)
- [ ] POST batch ingest (pipeline ZADD)
- [ ] GET time-window query (ZRANGEBYSCORE + cursor pagination)
- [ ] GET anomaly (ZREVRANGE + Welford)
- [ ] GET /metrics (Prometheus exposition)
- [ ] Table-driven unit tests for all packages

### Phase 3: Optimization (Days 6–8)
- [ ] `sync.Pool` for all hot-path buffers
- [ ] GC tuning (GOGC=off, GOMEMLIMIT, manual GC)
- [ ] Profiling with pprof — eliminate every allocation
- [ ] Benchmark suite for all critical paths
- [ ] Integration tests against real Redis

### Phase 4: Hardening (Days 9–10)
- [ ] nginx.conf round-robin
- [ ] docker-compose.yml with tight limits
- [ ] Multi-stage Dockerfile (scratch base)
- [ ] Local smoke test (`k6 run smoke.js`)
- [ ] Local load test (`k6 run test.js`)
- [ ] Stress tests (concurrent, sustained, spike, endurance)

### Phase 5: Submission (Before July 26)
- [ ] Public GitHub repo with MIT license
- [ ] `main` branch: full API code
- [ ] `implementation` branch: compose + nginx.conf + me.json
- [ ] Publish arm64 Docker image
- [ ] Fork challenge repo, add `submissions/<username>.json`
- [ ] Open PR
