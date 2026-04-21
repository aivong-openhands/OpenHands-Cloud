# gRPC vs HTTP for Laminar Tracing

## Quick Answer

**Both work perfectly fine.** For most applications, the difference is negligible. Use HTTP-only if gRPC isn't available - you get all the same features.

## When to Use Each

### Use HTTP-only when:
- ✅ gRPC ports not exposed (common in K8s/cloud environments)
- ✅ Behind HTTP-only proxies or load balancers
- ✅ Simpler infrastructure is preferred (one port)
- ✅ Tracing volume is moderate (<1000 spans/second)

### Use gRPC when:
- ✅ You want the `@observe` decorator (cleaner code)
- ✅ High-frequency tracing (>1000 spans/second)
- ✅ Infrastructure supports HTTP/2
- ✅ Multiple services using Laminar SDK

## Feature Comparison

| Feature | gRPC | HTTP-only |
|---------|------|-----------|
| Traces visible in UI | ✅ Yes | ✅ Yes |
| Nested spans | ✅ Yes | ✅ Yes |
| Custom attributes | ✅ Yes | ✅ Yes |
| Error tracking | ✅ Yes | ✅ Yes |
| Query API | ✅ Yes | ✅ Yes |
| `@observe` decorator | ✅ Yes | ❌ No (manual spans) |
| Code simplicity | ✅ Cleaner | ⚠️ More verbose |
| Latency per trace | ~5-10ms | ~10-20ms |
| Infrastructure | Needs HTTP/2 | Works everywhere |

## Code Comparison

### With gRPC (Cleaner)

```python
from lmnr import Laminar, observe

Laminar.initialize(
    project_api_key=API_KEY,
    base_url="http://laminar-server",
    http_port=8000,
    grpc_port=8001,
)

@observe(name="process_data", tags=["production"])
def process_data(user_id: str):
    """Clean decorator syntax"""
    @observe(name="fetch_from_db")
    def fetch():
        return get_user(user_id)
    
    @observe(name="transform")
    def transform(data):
        return enrich(data)
    
    data = fetch()
    return transform(data)
```

### With HTTP-only (More Manual)

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

# Setup (once at startup)
provider = TracerProvider()
exporter = OTLPSpanExporter(
    endpoint="http://laminar-server:8000/v1/traces",
    headers={"Authorization": f"Bearer {API_KEY}"}
)
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

def process_data(user_id: str):
    """Manual span creation"""
    with tracer.start_as_current_span(
        "process_data",
        attributes={"tag": "production"}
    ) as root:
        
        with tracer.start_as_current_span("fetch_from_db") as fetch_span:
            data = get_user(user_id)
        
        with tracer.start_as_current_span("transform") as transform_span:
            result = enrich(data)
        
        return result
```

**Result**: Both produce identical traces in the UI. HTTP-only just requires more explicit span management.

## Performance Reality

For typical applications:

| Traces/Second | gRPC Overhead | HTTP Overhead | Real Impact |
|---------------|---------------|---------------|-------------|
| 10-100 | 0.05ms CPU | 0.1ms CPU | Negligible |
| 100-1,000 | 0.5ms CPU | 1ms CPU | Barely noticeable |
| 1,000-10,000 | 5ms CPU | 10ms CPU | Noticeable at scale |
| >10,000 | Significant | Significant | gRPC recommended |

**Bottom line**: Unless you're tracing >1000 spans/second per service, HTTP-only is perfectly fine.

## When gRPC Actually Matters

### High-Volume Tracing
If you're tracing:
- Every database query in a high-traffic API
- Every microservice hop in a complex system
- ML inference calls (100s per second)

Then gRPC's binary protocol and lower overhead help.

### Real-Time Streaming
gRPC supports bidirectional streaming, which enables:
- Live trace updates in UI as spans complete
- Real-time alerting on trace anomalies
- Interactive debugging sessions

HTTP-only uses batch export, so traces appear every few seconds.

## Making HTTP-only Work Great

Even without gRPC, optimize with:

```python
# 1. Batch size tuning
from opentelemetry.sdk.trace.export import BatchSpanProcessor

processor = BatchSpanProcessor(
    exporter,
    max_queue_size=2048,       # More buffering
    schedule_delay_millis=5000, # Export every 5s
    max_export_batch_size=512,  # Bigger batches
)

# 2. Sampling for high volume
from opentelemetry.sdk.trace.sampling import TraceIdRatioBased

# Sample 10% of traces
sampler = TraceIdRatioBased(0.1)
provider = TracerProvider(sampler=sampler)

# 3. Async export (doesn't block your app)
# BatchSpanProcessor already does this!
```

## Enabling gRPC Later

If you start with HTTP-only and want to add gRPC later:

### Step 1: Expose gRPC port in your service

```yaml
# Kubernetes Service
ports:
  - name: http
    port: 8000
  - name: grpc
    port: 8001  # Add this
```

### Step 2: Update your code

```python
# Change from:
exporter = OTLPSpanExporter(
    endpoint="http://laminar-server:8000/v1/traces",
)

# To:
from lmnr import Laminar, observe

Laminar.initialize(
    project_api_key=API_KEY,
    base_url="http://laminar-server",
    http_port=8000,
    grpc_port=8001,
)

# Now use @observe decorator
```

### Step 3: Refactor to decorators gradually

```python
# Old HTTP-only code still works alongside new gRPC code
# Refactor function by function as needed
```

## Recommendation by Use Case

### Prototyping / MVP
→ **HTTP-only** - Simpler, works everywhere

### Production with <1000 spans/sec
→ **HTTP-only** - Performance difference is negligible

### Production with >1000 spans/sec
→ **gRPC** - Lower overhead matters at scale

### Multiple services needing tracing
→ **gRPC** - SDK auto-instrumentation is easier to manage

### Self-hosted K8s without gRPC exposed
→ **HTTP-only** - Don't complicate your infrastructure for minimal gain

### Enterprise with existing HTTP/2 infrastructure
→ **gRPC** - You already have the infrastructure

## Common Misconceptions

**Myth**: "HTTP-only traces are delayed or incomplete"
- ✅ **False**: Same data, appears within seconds

**Myth**: "HTTP-only can't handle production workloads"
- ✅ **False**: OpenTelemetry's HTTP exporter is production-grade

**Myth**: "gRPC is always faster"
- ⚠️ **Nuanced**: Yes it's faster, but the difference only matters at high volume

**Myth**: "You need gRPC for nested spans"
- ✅ **False**: HTTP-only supports full span hierarchy

## Troubleshooting Performance

If you notice high tracing overhead:

### HTTP-only
```python
# 1. Check batch settings
# 2. Add sampling
# 3. Profile your instrumentation code (not transport)
# 4. Consider gRPC if >1000 spans/sec
```

### gRPC
```python
# 1. Check network latency to server
# 2. Verify HTTP/2 is actually being used
# 3. Profile your instrumentation code
# 4. May still need sampling at very high volume
```

**Remember**: 90% of tracing overhead is from instrumentation (creating spans, serializing data), not transport protocol.

## Summary

| Question | Answer |
|----------|--------|
| Which is better? | **Both work great** |
| When does it matter? | **Only at >1000 spans/second** |
| Should I enable gRPC? | **If easy, yes. If hard, no need.** |
| Can I switch later? | **Yes, easily** |
| HTTP-only limitations? | **No @observe decorator (manual spans)** |
| Visible difference in UI? | **None** |

Choose based on your infrastructure constraints, not performance. HTTP-only is perfectly suitable for production use.
