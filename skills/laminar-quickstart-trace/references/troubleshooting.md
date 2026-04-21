# Quickstart troubleshooting

## Common Issues

### No traces appearing

1. **Verify API key**: Check `LMNR_PROJECT_API_KEY` is correct
2. **Check connectivity**: 
   ```bash
   # HTTP endpoint (should return 200)
   curl -I http://your-laminar-server:8000/health
   
   # gRPC endpoint (may not be available)
   timeout 3 bash -c "cat < /dev/null > /dev/tcp/your-host/8001"
   ```
3. **Test with API query**:
   ```bash
   curl -X POST "http://your-laminar-server:8000/v1/sql/query" \
     -H "Authorization: Bearer $LMNR_PROJECT_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"query":"SELECT COUNT(*) FROM spans WHERE start_time > now() - INTERVAL 1 HOUR","parameters":{}}'
   ```

### "DEADLINE_EXCEEDED" or "UNAVAILABLE" errors

**Symptom**: Script runs but shows errors like:
```
Failed to export traces to laminar-server:8443, error code: StatusCode.DEADLINE_EXCEEDED
```

**Cause**: Laminar SDK is trying to use gRPC but port 8001/8443 is not accessible

**Solution**: Use HTTP-only approach (see `http-only.md`)

### Trace delayed

- Set `disable_batch=True`/`disableBatch: true` in Laminar.initialize()
- Call `Laminar.flush()` or `await Laminar.flush()` before script exits
- Keep process alive for 1-2 seconds after flush
- For OpenTelemetry: Use `provider.force_flush(timeout_millis=10000)`

### Hard to find the trace

- Filter by tag: `quickstart`
- Filter by metadata key: `run_id`
- Search attributes for your specific run_id value (e.g., `quickstart-1776801524`)
- Check service.name if you set it in Resource

### Self-hosted UI missing data

1. Confirm backend services running:
   ```bash
   curl http://your-laminar-server:8000/health
   ```
2. Check ports are exposed:
   - HTTP: 8000 (required)
   - gRPC: 8001 or 8443 (optional, needed for @observe decorator)
3. Verify database is accessible to backend
4. Check logs of laminar-app-server-service for errors

### Traces in database but not in UI

1. Clear browser cache and refresh
2. Check time filters in UI (expand to wider range)
3. Verify project ID in URL matches your API key's project
4. Query database directly to confirm data:
   ```bash
   curl -X POST "$LAMINAR_URL/v1/sql/query" \
     -H "Authorization: Bearer $LMNR_PROJECT_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"query":"SELECT * FROM traces ORDER BY start_time DESC LIMIT 1","parameters":{}}'
   ```

## Diagnostic Script

```python
import os
import subprocess
import requests

API_KEY = os.getenv("LMNR_PROJECT_API_KEY")
LAMINAR_URL = os.getenv("LAMINAR_URL", "http://laminar-app-server-service:8000")

print("🔍 Laminar Diagnostics\n")

# 1. Check HTTP
print("1. Testing HTTP endpoint...")
try:
    r = requests.get(f"{LAMINAR_URL}/health", timeout=5)
    print(f"   ✓ HTTP {r.status_code} - Endpoint accessible")
except Exception as e:
    print(f"   ✗ HTTP failed: {e}")

# 2. Check gRPC
print("2. Testing gRPC endpoint...")
host = LAMINAR_URL.split("://")[1].split(":")[0]
result = subprocess.run(
    ["timeout", "3", "bash", "-c", f"cat < /dev/null > /dev/tcp/{host}/8001"],
    capture_output=True
)
if result.returncode == 0:
    print(f"   ✓ gRPC port 8001 accessible")
else:
    print(f"   ⚠ gRPC port 8001 not accessible - use HTTP-only mode")

# 3. Check API key
print("3. Testing API key...")
try:
    r = requests.post(
        f"{LAMINAR_URL}/v1/sql/query",
        headers={"Authorization": f"Bearer {API_KEY}"},
        json={"query": "SELECT 1", "parameters": {}},
        timeout=5
    )
    if r.status_code == 200:
        print(f"   ✓ API key valid")
    else:
        print(f"   ✗ API key invalid: {r.status_code} {r.text}")
except Exception as e:
    print(f"   ✗ API test failed: {e}")

# 4. Check recent traces
print("4. Checking recent traces...")
try:
    r = requests.post(
        f"{LAMINAR_URL}/v1/sql/query",
        headers={"Authorization": f"Bearer {API_KEY}"},
        json={
            "query": "SELECT COUNT(*) as count FROM spans WHERE start_time > now() - INTERVAL 1 HOUR",
            "parameters": {}
        },
        timeout=5
    )
    data = r.json()
    count = data["data"][0]["count"]
    print(f"   ℹ Found {count} spans in last hour")
except Exception as e:
    print(f"   ✗ Query failed: {e}")

print("\n✓ Diagnostics complete")
```

Save as `diagnose.py` and run with:
```bash
export LMNR_PROJECT_API_KEY="your-key"
export LAMINAR_URL="http://your-server:8000"
python diagnose.py
```
