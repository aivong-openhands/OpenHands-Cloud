# HTTP-Only Laminar Tracing (Self-Hosted)

Use this approach when gRPC ports (8001/8443) are not accessible but HTTP port (8000) is available.

## When to Use This

- Self-hosted Laminar without gRPC exposed
- Kubernetes environments with restricted port access
- HTTP-only load balancers or proxies
- When standard Laminar SDK shows "DEADLINE_EXCEEDED" or "UNAVAILABLE" errors

## Python HTTP-Only Setup

### 1. Test Connectivity First

```bash
# Test HTTP (should succeed)
curl http://YOUR_LAMINAR_URL:8000/health

# Test gRPC (might fail in HTTP-only setups)
timeout 3 bash -c "cat < /dev/null > /dev/tcp/YOUR_LAMINAR_HOST/8001" && echo "gRPC available" || echo "gRPC not available"
```

### 2. Install Dependencies

```bash
pip install opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp-proto-http
```

### 3. Quickstart Code

```python
import os
import time
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.trace import Status, StatusCode

# Configuration
API_KEY = os.getenv("LMNR_PROJECT_API_KEY", "your-api-key-here")
LAMINAR_URL = os.getenv("LAMINAR_URL", "http://laminar-app-server-service:8000")

# Setup tracing with HTTP exporter
def setup_tracing(service_name="quickstart-demo"):
    provider = TracerProvider(
        resource=Resource.create({"service.name": service_name})
    )
    
    http_exporter = OTLPSpanExporter(
        endpoint=f"{LAMINAR_URL}/v1/traces",
        headers={"Authorization": f"Bearer {API_KEY}"},
        timeout=10,
    )
    
    provider.add_span_processor(BatchSpanProcessor(http_exporter))
    trace.set_tracer_provider(provider)
    return provider

# Initialize
provider = setup_tracing()
tracer = trace.get_tracer(__name__)

# Create trace
run_id = f"quickstart-{int(time.time())}"
print(f"Run ID: {run_id}")

with tracer.start_as_current_span(
    "quickstart.root",
    attributes={"run_id": run_id, "tag": "quickstart"}
) as root_span:
    
    with tracer.start_as_current_span("quickstart.step") as step_span:
        step_span.set_attribute("answer", 42)
        step_span.set_status(Status(StatusCode.OK))
    
    root_span.set_status(Status(StatusCode.OK))

print("Flushing traces...")
provider.force_flush(timeout_millis=10000)
print(f"✓ Trace sent! Run ID: {run_id}")
```

### 4. Run

```bash
export LMNR_PROJECT_API_KEY="your-key-here"
export LAMINAR_URL="http://your-laminar-server:8000"
python quickstart_http.py
```

## Node.js HTTP-Only Setup

### 1. Install Dependencies

```bash
npm install @opentelemetry/api @opentelemetry/sdk-node @opentelemetry/exporter-trace-otlp-http @opentelemetry/resources @opentelemetry/semantic-conventions
```

### 2. Quickstart Code

```javascript
import { trace } from '@opentelemetry/api';
import { NodeTracerProvider } from '@opentelemetry/sdk-node';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';

const API_KEY = process.env.LMNR_PROJECT_API_KEY || 'your-api-key-here';
const LAMINAR_URL = process.env.LAMINAR_URL || 'http://laminar-app-server-service:8000';

// Setup tracing
const provider = new NodeTracerProvider({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'quickstart-demo',
  }),
});

const exporter = new OTLPTraceExporter({
  url: `${LAMINAR_URL}/v1/traces`,
  headers: {
    'Authorization': `Bearer ${API_KEY}`,
  },
});

provider.addSpanProcessor(new BatchSpanProcessor(exporter));
provider.register();

const tracer = trace.getTracer('quickstart');

// Create trace
const runId = `quickstart-${Date.now()}`;
console.log(`Run ID: ${runId}`);

const rootSpan = tracer.startSpan('quickstart.root', {
  attributes: { run_id: runId, tag: 'quickstart' },
});

const stepSpan = tracer.startSpan('quickstart.step', {
  attributes: { answer: 42 },
});

stepSpan.end();
rootSpan.end();

console.log('Flushing traces...');
await provider.forceFlush();
console.log(`✓ Trace sent! Run ID: ${runId}`);
```

### 3. Run

```bash
export LMNR_PROJECT_API_KEY="your-key-here"
export LAMINAR_URL="http://your-laminar-server:8000"
node quickstart_http.mjs
```

## Verification

### Query API to Check Traces

```bash
curl -X POST "$LAMINAR_URL/v1/sql/query" \
  -H "Authorization: Bearer $LMNR_PROJECT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "SELECT name, trace_id, start_time FROM spans WHERE start_time > now() - INTERVAL 10 MINUTE ORDER BY start_time DESC LIMIT 10",
    "parameters": {}
  }'
```

## Limitations

**Cannot use Laminar SDK decorators**: The `lmnr` package's `@observe` decorator (Python) and `observe()` wrapper (Node.js) require gRPC and will fail with "DEADLINE_EXCEEDED" errors.

**Workaround**: Use OpenTelemetry's tracer API directly (as shown above). You get the same functionality:
- ✅ Nested spans
- ✅ Custom attributes and metadata
- ✅ Error tracking
- ✅ Full UI visibility
- ❌ Cannot use `@observe` decorator syntax

## When to Fix gRPC Instead

If you need the `@observe` decorator or Laminar's auto-instrumentation features, expose gRPC ports:

1. Update Kubernetes service to expose port 8001 or 8443
2. Configure load balancer for gRPC traffic
3. Then use the standard SDK with `base_url`, `http_port`, and `grpc_port` parameters

## Troubleshooting

**Traces not appearing?**
1. Verify endpoint: `curl $LAMINAR_URL/health`
2. Check API key is correct
3. Ensure flush/forceFlush is called before script exits
4. Add timeout to flush: `provider.force_flush(timeout_millis=10000)`

**Empty response from query API?**
- Traces may take 1-2 seconds to appear in database
- Add `time.sleep(2)` after flush, then query again

**"Authorization failed" errors?**
- Verify API key has correct project access
- Check Authorization header format: `Bearer YOUR_KEY`
