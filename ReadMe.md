# Conviva Python Agent SDK

This guide shows how to enable telemetry for your AI agent using the Conviva Agent SDK. 

## Overview
- The SDK sends telemetry to Conviva using the OTLP/HTTP protocol.
- You must provide your Conviva customer key to activate the SDK.
- Endpoints are computed automatically by the SDK; you normally don’t need to configure them.

## Prerequisites
- Your Conviva Customer Key (provided by Conviva).
- Access to Conviva’s telemetry endpoints.

## Environment Variables
The SDK supports various environment variables for configuration. Here are the key ones:

### Required
- `CONVIVA_AGW_DOMAIN` : "appgw.conviva.com"
- `CONVIVA_CUSTOMER_KEY`: Your Conviva customer key (required for production endpoints)
- `CONVIVA_SERVICE_NAME`: Override service name (defaults to "unknown_service:python") Ex:"chat-assistant-agent"
- `CONVIVA_SERVICE_VERSION`: Set service version Ex:"1.0.0"

### Optional Configuration
- `CONVIVA_RESOURCE_ATTRIBUTES`: CSV format key=value pairs for resource attributes

## Python
1) Download and install the wheel file
   - Download the wheel file from https://github.com/Conviva/conviva-python-agent-sdk/releases/download/v1.0.0/conviva_agent_sdk-1.0.0-py3-none-any.whl
   - Install it with pip
      ```bash
      pip install ./conviva-agent-sdk-1.0.0-py3-none-any.whl
      ```

2) Configure Required environment variables
   ```python
   export CONVIVA_AGW_DOMAIN="appgw.conviva.com"
   export CONVIVA_CUSTOMER_KEY=<your_customer_key>
   export CONVIVA_SERVICE_NAME="my-ai-service"
   export CONVIVA_SERVICE_VERSION="1.0.0"
   #Optional - Custom headers
   export CONVIVA_RESOURCE_ATTRIBUTES="environment=production,team=ai-platform"
   ```
3) Initialize early in your application
```python
from conviva_agent_sdk import ConvivaAgentSDK
import os

# Initialize the SDK
ConvivaAgentSDK.init(
    customer_key=os.environ.get("CONVIVA_CUSTOMER_KEY"),
    service_name=os.environ.getenv("CONVIVA_SERVICE_NAME"),
    service_version=os.environ.getenv("CONVIVA_SERVICE_VERSION"),
)

# Flush and shutdown when needed,may be in on service shutdown
ConvivaAgentSDK.flush(5000)
ConvivaAgentSDK.shutdown(5000)
```
## Best Practices
- Call `ConvivaAgentSDK.init(...)` once when your process starts.
- Provide a meaningful `service_name` so you can identify your app in Conviva.
- Ensure your app flushes and shuts down the SDK before exit to avoid losing data.
- Captures key telemetry from AI agents out-of-the-box (no extra wiring needed).

## Troubleshooting
- If you do not see data, confirm `CONVIVA_CUSTOMER_KEY` is set in your environment and that your application can reach the Conviva endpoint.
- For additional help, contact your Conviva representative.


