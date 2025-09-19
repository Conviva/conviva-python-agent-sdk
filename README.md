# Conviva Python Agent SDK

This guide shows how to enable telemetry for your AI agent using the Conviva Agent SDK for Python. 

## Overview
- The SDK sends telemetry to Conviva using the OTLP/HTTP protocol.
- You must provide your Conviva customer key to activate the SDK.
- Endpoints are computed automatically by the SDK; you normally don’t need to configure them.

## Prerequisites
- Your Conviva Customer Key (provided by Conviva).
- Internet access to Conviva’s telemetry endpoints.

## Environment Variables

The SDK supports various environment variables for configuration. Here are the key ones:

### Required
- `CONVIVA_CUSTOMER_KEY`: Your Conviva customer key (required for production endpoints)
- `CONVIVA_SERVICE_NAME`: Override service name (defaults to "unknown_service:python")
- `CONVIVA_SERVICE_VERSION`: Set service version

### Optional Configuration
- `CONVIVA_RESOURCE_ATTRIBUTES`: CSV format key=value pairs for resource attributes

### Endpoint Configuration
- `CONVIVA_AGW_DOMAIN`: Override default AGW domain (defaults to "agw.conviva.com")

### SDK Behavior
- `CONVIVA_SDK_DISABLED`: Set to "true" to disable the SDK entirely
- `CONVIVA_AUTO_INCLUDE`: CSV list of instruments to include
- `CONVIVA_AUTO_EXCLUDE`: CSV list of instruments to exclude
- `CONVIVA_TIMEOUT_MS`: Request timeout in milliseconds (defaults to 10000)

## Integration Guidelines
1) Install the wheel
   - Download the wheel file from https://github.com/Conviva/conviva-python-agent-sdk/releases/download/v1.0.4/conviva_agent_sdk-1.0.4-py3-none-any.whl and do pip install
```bash
pip install ./conviva-agent-sdk-<version>-py3-none-any.whl
```

2) Initialize early in your application
```python
from conviva_agent_sdk import ConvivaAgentSDK
import os

# Initialize the SDK
ConvivaAgentSDK.init(
    customer_key=os.environ.get("CONVIVA_CUSTOMER_KEY"),
    service_name="my-service",
    service_version="1.0.0",
)

# Flush and shutdown when needed,may be in on service shutdown
ConvivaAgentSDK.flush(5000)
ConvivaAgentSDK.shutdown(5000)
```

3) Environment variable examples
```bash
# Required
export CONVIVA_CUSTOMER_KEY=<your_customer_key>
export CONVIVA_SERVICE_NAME="my-ai-service"
export CONVIVA_SERVICE_VERSION="2.1.0"

# Optional - Custom headers
export CONVIVA_RESOURCE_ATTRIBUTES="environment=production,team=ai-platform"

# Optional - Logging
export CONVIVA_LOG_LEVEL="INFO"  # DEBUG, INFO, WARNING, ERROR, CRITICAL, OFF

# Optional - Tracing Context
export CONVIVA_TRACING_CONTEXT_KEYS="convID,client_id,user_id,session_id"
export CONVIVA_TRACING_CONTEXT_MAX_VALUE_LEN="2048"

```

Notes
- By default, telemetry is sent to: `https://<CONVIVA_CUSTOMER_KEY>.agw.conviva.com/v1/{traces|logs}`.
- No additional endpoint configuration is needed in normal operation.
- The SDK uses a singleton pattern - only one instance can exist per process.

<details>
<summary><h3>Fast API Integration Example</h3></summary>

```python
from fastapi import FastAPI, Request
from conviva_agent_sdk import ConvivaAgentSDK
import os

app = FastAPI()

# ----------------------------
# Startup / Shutdown lifecycle
# ----------------------------

@app.on_event("startup")
async def startup_event():
    ConvivaAgentSDK.init(
        customer_key=os.environ["CONVIVA_CUSTOMER_KEY"],
        service_name=os.environ.get("CONVIVA_SERVICE_NAME", "my-ai-service"),
        service_version=os.environ.get("CONVIVA_SERVICE_VERSION", "1.0.0"),
    )
    print("✅ ConvivaAgentSDK initialized")

@app.on_event("shutdown")
async def shutdown_event():
    # Flush pending events and shutdown gracefully
    print("🛑 Shutting down ConvivaAgentSDK...")
    ConvivaAgentSDK.flush(5000)
    ConvivaAgentSDK.shutdown(5000)

# ----------------------------
# Middleware for tracing context
# ----------------------------

@app.middleware("http")
async def conviva_middleware(request: Request, call_next):
    tracing_context = {
        "convID": request.headers.get("X-Conviva-ConvID"),
        "client_id": request.headers.get("X-Client-ID"),
        "user_id": request.headers.get("X-User-ID"),
        "session_id": request.headers.get("X-Session-ID"),
    }

    with ConvivaAgentSDK.tracing_context(tracing_context):
        response = await call_next(request)
    return response

# ----------------------------
# Routes
# ----------------------------

@app.get("/health")
async def health():
    return {"status": "ok"}
```
</details>

## Tracing Context Configuration

The SDK automatically copies tracing context to span attributes for better observability. You can control which keys are copied and how long values can be.

- `CONVIVA_TRACING_CONTEXT_KEYS`: Comma-separated list of context keys to copy to span attributes (defaults to "convID,client_id")
- `CONVIVA_TRACING_CONTEXT_MAX_VALUE_LEN`: Maximum length for tracing context values before truncation (defaults to "256")

### Environment Variables

```bash
# Control which baggage keys are copied to span attributes
export CONVIVA_TRACING_CONTEXT_KEYS="convID,client_id,user_id,session_id,request_id"

# Set maximum length for values before truncation
export CONVIVA_TRACING_CONTEXT_MAX_VALUE_LEN="2048"
```

### How It Works

1. **Key Filtering**: Only baggage keys listed in `CONVIVA_TRACING_CONTEXT_KEYS` are copied to span attributes
2. **Case Insensitive**: Key matching is case-insensitive (e.g., "convID" matches "convid", "CONVID", etc.)
3. **Value Truncation**: Values longer than `CONVIVA_TRACING_CONTEXT_MAX_VALUE_LEN` are truncated with "…" suffix
4. **Namespace**: Copied attributes are prefixed with a namespace (default: "c3.")

### Examples

```python
# Set tracing context with multiple keys
with ConvivaAgentSDK.tracing_context({
    "convID": "abc123",
    "user_id": "user456", 
    "session_id": "sess789",
    "request_id": "req001"
}):
    # Only keys in CONVIVA_TRACING_CONTEXT_KEYS will be copied to spans
    # as attributes like: c3.convID, c3.user_id, etc.
    pass
```

### Configuration Examples

```bash

# Comprehensive configuration for microservices
export CONVIVA_TRACING_CONTEXT_KEYS="convID,client_id,user_id,session_id,request_id,operation_type,service_name"
export CONVIVA_TRACING_CONTEXT_MAX_VALUE_LEN="2048"
```

<details>
<summary><h2>Tracing Context Management(Optional)</h2></summary>

The SDK provides methods to manage tracing context and baggage for distributed tracing across your application if trace context propagation does not propagate across all services.

### Context Manager: `tracing_context`

Use `tracing_context` as a context manager to automatically attach and detach tracing context:

```python
from conviva_agent_sdk import ConvivaAgentSDK

# Using context manager (recommended)
with ConvivaAgentSDK.tracing_context({"convID": "<placeholder-unique-chat-id>"}):
    # Your code here - tracing context is automatically available
    # All spans created within this block will have the trace context attached
    pass
# Context is automatically cleaned up when exiting the block
```

### Manual Context Management: `set_tracing_context` and `detach_tracing_context`

For more control, you can manually manage tracing context:

```python
from conviva_agent_sdk import ConvivaAgentSDK

# Set tracing context and get a token
token = ConvivaAgentSDK.set_tracing_context({
    "convID": "ZfhAUaOrnBIdCO4-nz5p2"
})

# Your code here - tracing context is available
# All spans created will have the baggage attached

# Clean up when done
ConvivaAgentSDK.detach_tracing_context(token)
```

### Use Cases

- **Conversation Tracking**: Attach conversation IDs to trace user interactions across multiple requests
- **User Context**: Include user IDs, session IDs, or other identifying information
- **Request Correlation**: Link related operations with custom identifiers
- **Custom Metadata**: Add any key-value pairs that should be propagated with your traces

<details>
<summary><h3>Example: FastAPI Middleware</h3></summary>

```python
from fastapi import FastAPI, Request
from conviva_agent_sdk import ConvivaAgentSDK

app = FastAPI()

@app.middleware("http")
async def add_tracing_context(request: Request, call_next):
    # Extract tracing context from headers
    tracing_dict = {
        "convID": request.headers.get("X-Conversation-ID"),
        "session_id": request.headers.get("X-Session-ID"),
        "user_id": request.headers.get("X-User-ID")
    }
    
    # Filter out None values
    tracing_dict = {k: v for k, v in tracing_dict.items() if v is not None}
    
    # Apply tracing context for the entire request
    with ConvivaAgentSDK.tracing_context(tracing_dict):
        response = await call_next(request)
    
    return response
```
</details>
</details>


## Best Practices
- Call `ConvivaAgentSDK.init(...)` once when your process starts.
- Provide a meaningful `service_name` so you can identify your app in Conviva.
- Ensure your app flushes and shuts down the SDK before exit to avoid losing data.
- The SDK is thread-safe and can be called from multiple threads.
- Use `tracing_context` context manager for automatic cleanup when possible.
- Always call `detach_tracing_context` when using manual context management to prevent memory leaks.

## Troubleshooting
- If you do not see data, confirm `CONVIVA_CUSTOMER_KEY` is set in your environment and that your application can reach the Conviva endpoint.
- For additional help, contact your Conviva representative.


