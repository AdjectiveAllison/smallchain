# Model Control Protocol (MCP) Servers

## Overview

The Model Control Protocol (MCP) is a standard interface for connecting AI/LLM agents with external tools and capabilities. In ACP, MCP servers are defined using the `MCPServer` custom resource type.

## MCPServer Resource

The `MCPServer` resource defines how to connect to an MCP server, which can provide tools to LLM agents.

### Spec Fields

| Field | Description | Example |
|-------|-------------|---------|
| `transport` | Communication method (`stdio` or `http`) | `stdio` |
| `command` | Command to run (for stdio transport) | `"/usr/local/bin/mcp-server"` |
| `args` | Arguments for the command | `["--verbose"]` |
| `env` | Environment variables | See below |
| `url` | URL for HTTP transport | `"https://mcp-server.example.com"` |
| `resources` | CPU/memory requests and limits | See below |
| `approvalContactChannel` | Reference to a ContactChannel for tool approval | `{ "name": "slack-channel" }` |

### Environment Variables

The `env` field supports two ways to specify environment variables:

1. **Direct Values**:
   ```yaml
   env:
     - name: DEBUG
       value: "true"
   ```

2. **Secret References**:
   ```yaml
   env:
     - name: API_KEY
       valueFrom:
         secretKeyRef:
           name: mcp-credentials
           key: api-key
   ```

Secret references allow you to securely provide sensitive information like API keys and credentials to your MCP server without hardcoding them in the resource definition.

### Resource Requirements

You can specify resource requests and limits for the MCP server process:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

### Tool Approval

By specifying an `approvalContactChannel`, you can configure a ContactChannel to be used for approving sensitive tool calls:

```yaml
approvalContactChannel:
  name: slack-approval-channel
```

This allows users to approve or reject tool calls before they're executed, which is especially useful for tools that make changes or perform sensitive operations.

## Example: MCP Server with Secret Reference

```yaml
apiVersion: acp.humanlayer.dev/v1alpha1 
kind: MCPServer
metadata:
  name: fetch-mcp-server
  namespace: default
spec:
  transport: stdio
  command: "uvx"
  args: ["mcp-server-fetch"]
  env:
    - name: LOG_LEVEL
      value: "debug"
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: fetch-api-credentials
          key: api-key
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi
  approvalContactChannel:
    name: slack-approval-channel
```

You'll need to create the corresponding Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: fetch-api-credentials
  namespace: default
type: Opaque
data:
  api-key: <base64-encoded-api-key>
```

## Status Fields

| Field | Description |
|-------|-------------|
| `connected` | Whether the MCP server is connected and operational |
| `status` | Current status (`Ready`, `Error`, or `Pending`) |
| `statusDetail` | Detailed status message |
| `tools` | List of tools provided by the MCP server |

## Tool Discovery and Naming

When an MCPServer is created, the controller automatically:
1. Connects to the MCP server
2. Discovers available tools
3. Makes these tools available to agents that reference this server

### Tool Naming Convention

Tools from MCP servers are automatically made available to referencing agents with a naming convention that ensures uniqueness:

```
serverName__toolName
```

For example, a tool named `fetchURL` from an MCP server named `fetch-server` would be available to agents as `fetch-server__fetchURL`.

## Using MCP-provided Tools in Agents

To use tools from an MCP server in an agent, you need to reference the MCPServer in the agent definition:

```yaml
apiVersion: acp.humanlayer.dev/v1alpha1
kind: Agent
metadata:
  name: research-agent
  namespace: default
spec:
  llmRef:
    name: my-llm
  system: "You are a research assistant."
  mcpServers:
    - name: fetch-mcp-server
```

The agent will now have access to all tools provided by `fetch-mcp-server`, and the controller manages the routing of tool calls to the appropriate MCP server.

## State Transitions

MCPServer resources follow a simple state machine:

1. `Pending`: Initial state when the server is being created or connection is being established
2. `Ready`: The server is connected and operational
3. `Error`: There was an error connecting to the server or during tool discovery

When a server is in the `Ready` state, the `connected` field will be set to `true` and the `tools` field will contain the list of discovered tools.

## Troubleshooting

If you encounter issues with MCP servers:

1. Check the `status` and `statusDetail` fields of the MCPServer resource
2. Verify that the MCP server is reachable from the controller
3. For stdio servers, check that the command and arguments are correct
4. For HTTP servers, ensure the URL is accessible and valid
5. Check that any referenced secrets exist and contain the correct keys
6. Inspect controller logs for detailed error messages