### Conviva Agent SDK — Integration Guide (Python)

This guide shows how to enable telemetry for your AI agent using the Conviva Agent SDK for Python. 

### Overview
- The SDK sends telemetry to Conviva using the OTLP/HTTP protocol.
- You must provide your Conviva customer key to activate the SDK.
- Endpoints are computed automatically by the SDK; you normally don’t need to configure them.

### Prerequisites
- Your Conviva Customer Key (provided by Conviva).
- Internet access to Conviva’s telemetry endpoints.

### Environment Variables

The SDK supports various environment variables for configuration. Here are the key ones:

#### Required
- `CONVIVA_CUSTOMER_KEY`: Your Conviva customer key (required for production endpoints)

#### Optional Configuration
- `CONVIVA_SERVICE_NAME`: Override service name (defaults to "unknown_service:python")
- `CONVIVA_SERVICE_VERSION`: Set service version
- `CONVIVA_RESOURCE_ATTRIBUTES`: CSV format key=value pairs for resource attributes
- `CONVIVA_HEADERS`: Additional headers to send with telemetry (JSON format)

#### Endpoint Configuration
- `CONVIVA_AGW_DOMAIN`: Override default AGW domain (defaults to "agw.conviva.com")
- `CONVIVA_ALLOW_TEST_ENDPOINTS`: Set to "true" to allow localhost/127.0.0.1 endpoints
- `CONVIVA_TRACES_ENDPOINT`: Test mode traces endpoint (localhost only)
- `CONVIVA_LOGS_ENDPOINT`: Test mode logs endpoint (localhost only)

#### OpenTelemetry Fallbacks
- `OTEL_SERVICE_NAME`: Fallback service name if CONVIVA_SERVICE_NAME not set
- `OTEL_RESOURCE_ATTRIBUTES`: Fallback resource attributes
- `OTEL_PROPAGATORS`: Trace context propagators (defaults to "tracecontext,baggage")

#### SDK Behavior
- `CONVIVA_SDK_DISABLED`: Set to "true" to disable the SDK entirely
- `CONVIVA_AUTO_INCLUDE`: CSV list of instruments to include
- `CONVIVA_AUTO_EXCLUDE`: CSV list of instruments to exclude
- `CONVIVA_TIMEOUT_MS`: Request timeout in milliseconds (defaults to 10000)

### Python
1) Install the wheel
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

# Flush and shutdown when needed
ConvivaAgentSDK.flush(5000)
ConvivaAgentSDK.shutdown(5000)
```

3) Environment variable examples
```bash
# Required
export CONVIVA_CUSTOMER_KEY=<your_customer_key>

# Optional - Service configuration
export CONVIVA_SERVICE_NAME="my-ai-service"
export CONVIVA_SERVICE_VERSION="2.1.0"
export CONVIVA_RESOURCE_ATTRIBUTES="environment=production,team=ai-platform"

# Optional - Custom headers
export CONVIVA_HEADERS='{"X-API-Key": "your-api-key", "X-Environment": "prod"}'

```

Notes
- By default, telemetry is sent to: `https://<CONVIVA_CUSTOMER_KEY>.agw.conviva.com/v1/{traces|logs}`.
- No additional endpoint configuration is needed in normal operation.
- The SDK uses a singleton pattern - only one instance can exist per process.

### Best Practices
- Call `ConvivaAgentSDK.init(...)` once when your process starts.
- Provide a meaningful `service_name` so you can identify your app in Conviva.
- Ensure your app flushes and shuts down the SDK before exit to avoid losing data.
- The SDK is thread-safe and can be called from multiple threads.

### Troubleshooting
- If you do not see data, confirm `CONVIVA_CUSTOMER_KEY` is set in your environment and that your application can reach the Conviva endpoint.
- For additional help, contact your Conviva representative.


