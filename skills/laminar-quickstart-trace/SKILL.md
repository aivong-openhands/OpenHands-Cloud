---
name: laminar-quickstart-trace
description: "Create a minimal Laminar trace demo in minutes with no external LLM calls. Use when a user asks for a Laminar example, a quickstart demo, or wants to see traces appear in the Laminar UI quickly (cloud or self-hosted)."
---

# Laminar Quickstart Trace

## User Input Required

Always ask the user for:
1. **LMNR_PROJECT_API_KEY** - Their Laminar project API key
2. **Laminar API server URL** - Examples:
   - Cloud: `https://api.lmnr.ai` (default)
   - Self-hosted: `http://laminar-app-server-service:8000` or custom domain
3. **Runtime preference** - Python or Node.js

Example prompt: "To set up Laminar tracing, I need: 1) Your Laminar API key (LMNR_PROJECT_API_KEY), 2) Your Laminar server URL (e.g., http://laminar-app-server-service:8000 for self-hosted or https://api.lmnr.ai for cloud), and 3) Your preferred runtime (Python or Node.js)."

## Workflow

1. **Collect credentials** - Ask for API key and server URL as described above
2. **Test connectivity** - Check both HTTP and gRPC ports (default: 8000 and 8001)
3. **Choose approach**:
   - If gRPC available → Use standard Laminar SDK with `base_url`, `http_port`, `grpc_port`
   - If HTTP only → Use OpenTelemetry HTTP exporter directly (see `references/http-only.md`)
4. **Build demo** - Create smallest runnable demo with manual spans; tag with unique run_id
5. **Run and verify** - Execute script, flush traces, direct user to UI with filtering instructions
6. **Troubleshoot** - If traces don't appear, check `references/troubleshooting.md`

## References

- `references/quickstart-python.md` - Python demo with standard Laminar SDK (requires gRPC)
- `references/quickstart-node.md` - Node/JS demo with standard Laminar SDK (requires gRPC)
- `references/http-only.md` - HTTP-only approach for self-hosted without gRPC
- `references/troubleshooting.md` - Common failures and fast fixes
