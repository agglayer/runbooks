# Aggsender - Manual Certificate Trigger Runbook

## Overview

This runbook describes how to manually trigger certificate creation in the Aggsender component. The `aggsender_triggerCertitificate` RPC method forces the publication of an epoch event, which initiates the certificate creation process.

## Prerequisites

### Configuration Requirements

Before triggering a certificate, ensure the following configuration is in place:

1. **Enable RPC in Aggsender Configuration**

   Set `EnableRPC` to `true` in your aggsender configuration file:

   ```toml
   [AggSender]
   EnableRPC = true
   ```

2. Verify Aggsender is Running

2. Confirm the aggsender service is running and accessible on the configured port (default: 5576)

## Execution Steps

### Trigger Certificate Creation

Execute the following command to manually trigger certificate creation:

```sh
curl -X POST http://localhost:5576/ \
  -H "Content-Type: application/json" \
  -d '{"method":"aggsender_triggerCertitificate", "params":[], "id":1}'
```

## Expected Response

A successful request should return a JSON-RPC response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": "success"
}
```

## Troubleshooting

### Connection Refused

Symptom: curl: (7) Failed to connect to localhost port 5576

Solutions:
1. Verify aggsender is running
2. Check the configured RPC port matches your curl command
3. Verify firewall rules allow connections to the RPC port

### RPC Method Not Found

Symptom: JSON-RPC error indicating method not found
Solutions:
1. Confirm EnableRPC = true in configuration
2. Restart aggsender after configuration changes
3. Check aggsender logs for RPC initialization messages

### Certificate Not Created

Symptom: Successful RPC response but no certificate generated

Solutions:
1. Check aggsender logs for epoch event processing
2. Verify sufficient data exists to create a certificate
3. Ensure dependent services (agglayer, L2 nodes) are operational

## Security Considerations

- Production Environments: Restrict RPC access to authorized operators only
- Network Segmentation: Run RPC on internal networks, not publicly exposed
