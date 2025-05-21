# Custom Resource Definition (CRD) Reference

This document provides comprehensive reference information for the Custom Resource Definitions (CRDs) used in the Agent Control Plane.

## MCPServer

The MCPServer CRD represents a Model Control Protocol server instance that provides tools for agents.

### Spec Fields

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `transport` | string | Connection type: "stdio" or "http" | Yes |
| `command` | string | Command to run (for stdio transport) | No |
| `args` | []string | Arguments for the command | No |
| `env` | []EnvVar | Environment variables | No |
| `url` | string | URL (for http transport) | No |
| `resources` | ResourceRequirements | CPU/memory resource requests/limits | No |
| `approvalContactChannel` | LocalObjectReference | Reference to a ContactChannel for tool approval | No |

#### EnvVar

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `name` | string | Environment variable name | Yes |
| `value` | string | Direct value for the environment variable | No* |
| `valueFrom` | EnvVarSource | Source for the environment variable value | No* |

*Either `value` or `valueFrom` must be specified.

#### EnvVarSource

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `secretKeyRef` | SecretKeyRef | Reference to a secret | No |

#### SecretKeyRef

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `name` | string | Name of the secret | Yes |
| `key` | string | Key within the secret | Yes |

#### ResourceRequirements

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `limits` | ResourceList | Maximum resource limits | No |
| `requests` | ResourceList | Minimum resource requests | No |

ResourceList is a map of ResourceName to resource.Quantity (e.g., `cpu: 100m`).

### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `connected` | boolean | Whether the MCP server is currently connected and operational |
| `status` | string | Current status: "Ready", "Error", or "Pending" |
| `statusDetail` | string | Detailed status message |
| `tools` | []MCPTool | List of tools provided by this MCP server |

#### MCPTool

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `name` | string | Name of the tool | Yes |
| `description` | string | Description of the tool | No |
| `inputSchema` | runtime.RawExtension | JSON schema for the tool's input parameters | No |

## LLM

The LLM CRD represents a Large Language Model configuration.

### Spec Fields

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `provider` | string | LLM provider (one of: "openai", "anthropic", "mistral", "google", "vertex") | Yes |
| `apiKeyFrom` | APIKeySource | Secret containing the API key | Yes |
| `parameters` | BaseConfig | Common configuration options across providers | No |
| `openai` | OpenAIConfig | OpenAI-specific configuration | No |
| `anthropic` | AnthropicConfig | Anthropic-specific configuration | No |
| `vertex` | VertexConfig | Vertex AI-specific configuration | No |
| `mistral` | MistralConfig | Mistral-specific configuration | No |
| `google` | GoogleConfig | Google AI-specific configuration | No |

#### BaseConfig

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `model` | string | Model name to use | No |
| `baseURL` | string | Base URL for API endpoints | No |
| `temperature` | string | Controls randomness (0.0 to 1.0) | No |
| `maxTokens` | int | Maximum number of tokens to generate | No |
| `topP` | string | Controls diversity via nucleus sampling (0.0 to 1.0) | No |
| `topK` | int | Controls diversity by limiting the top K tokens | No |
| `frequencyPenalty` | string | Reduces repetition by penalizing frequent tokens (-2.0 to 2.0) | No |
| `presencePenalty` | string | Reduces repetition by penalizing tokens that appear at all (-2.0 to 2.0) | No |

#### Provider-Specific Configurations

##### OpenAIConfig
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `organization` | string | OpenAI organization ID | No |
| `apiType` | string | API type: "OPEN_AI", "AZURE", or "AZURE_AD" | No |
| `apiVersion` | string | API version (required for Azure API types) | No |

##### AnthropicConfig
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `anthropicBetaHeader` | string | Anthropic Beta header for extended options | No |

##### VertexConfig
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `cloudProject` | string | Google Cloud project ID | Yes |
| `cloudLocation` | string | Google Cloud region | Yes |

##### MistralConfig
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `maxRetries` | int | Maximum number of retries for API calls | No |
| `timeout` | int | Timeout in seconds for API calls | No |
| `randomSeed` | int | Seed for deterministic sampling | No |

##### GoogleConfig
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `cloudProject` | string | Google Cloud project ID | No |
| `cloudLocation` | string | Google Cloud region | No |

### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `ready` | boolean | Whether the LLM is ready to use |
| `status` | string | Current status: "Ready", "Error", or "Pending" |
| `statusDetail` | string | Detailed status message |

## Agent

The Agent CRD represents an LLM agent with specific tools and capabilities.

### Spec Fields

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `llmRef` | LocalObjectReference | Reference to an LLM resource | Yes |
| `system` | string | System prompt for the agent | Yes |
| `mcpServers` | []LocalObjectReference | MCP servers providing tools to the agent | No |
| `humanContactChannels` | []LocalObjectReference | Contact channels for human interactions | No |
| `subAgents` | []LocalObjectReference | Other agents that can be delegated to | No |
| `description` | string | Optional description for the agent | No |

### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `ready` | boolean | Whether the agent's dependencies are valid and ready |
| `status` | string | Current status: "Ready", "Error", or "Pending" |
| `statusDetail` | string | Detailed status message |
| `validMCPServers` | []ResolvedMCPServer | List of MCP servers successfully validated |
| `validHumanContactChannels` | []ResolvedContactChannel | List of contact channels successfully validated |
| `validSubAgents` | []ResolvedSubAgent | List of sub-agents successfully validated |

#### ResolvedMCPServer
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `name` | string | Name of the MCP server | Yes |
| `tools` | []string | List of tools available from this server | No |

#### ResolvedContactChannel
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `name` | string | Name of the contact channel | Yes |
| `type` | string | Type of the contact channel | Yes |

#### ResolvedSubAgent
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `name` | string | Name of the sub-agent | Yes |

## Task

The Task CRD represents a task instance to be executed by an agent.

### Spec Fields

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `agentRef` | LocalObjectReference | Reference to the agent to execute the task | Yes |
| `userMessage` | string | Message to send to the agent | No* |
| `contextWindow` | []Message | Initial conversation context with multiple messages | No* |
| `responseURL` | string | URL for receiving task results | No |

*Either `userMessage` or `contextWindow` must be specified, but not both.

### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `ready` | boolean | Whether the task is ready to be executed |
| `status` | string | Current status: "Ready", "Error", or "Pending" |
| `statusDetail` | string | Detailed status message |
| `phase` | string | Current phase of execution |
| `contextWindow` | []Message | The conversation context |
| `userMsgPreview` | string | Preview of the user message |
| `output` | string | Result of the task execution |
| `error` | string | Error message if the task failed |
| `startTime` | time | When the task started |
| `completionTime` | time | When the task completed |
| `messageCount` | int | Number of messages in the context window |
| `spanContext` | SpanContext | OpenTelemetry span context information |
| `toolCallRequestId` | string | Unique ID for tool calls from a single LLM response |

## ToolCall

The ToolCall CRD represents a call to a tool during task execution.

### Spec Fields

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `toolCallId` | string | Unique identifier for this tool call | Yes |
| `taskRef` | LocalObjectReference | Reference to the parent task | Yes |
| `toolRef` | LocalObjectReference | Reference to the tool to execute | Yes |
| `toolType` | string | Type of the tool: "MCP", "HumanContact", or "DelegateToAgent" | No |
| `arguments` | string | Arguments for the tool call in JSON format | Yes |

### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `phase` | string | Current phase of the tool call | No |
| `ready` | boolean | Whether the tool call is ready to be executed | No |
| `status` | string | Current status: "Ready", "Error", "Pending", or "Succeeded" | No |
| `statusDetail` | string | Detailed status message | No |
| `externalCallID` | string | Unique identifier in external services | No |
| `result` | string | Result of the tool call if completed | No |
| `error` | string | Error message if the tool call failed | No |
| `startTime` | time | When the tool call started | No |
| `completionTime` | time | When the tool call completed | No |
| `spanContext` | SpanContext | OpenTelemetry span context information | No |

## ContactChannel

The ContactChannel CRD represents a communication channel for human interactions.

### Spec Fields

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `type` | string | Type of channel: "slack" or "email" | Yes |
| `apiKeyFrom` | APIKeySource | Secret containing the API key or token | Yes |
| `slack` | SlackChannelConfig | Slack-specific configuration | No |
| `email` | EmailChannelConfig | Email-specific configuration | No |

#### SlackChannelConfig
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `channelOrUserID` | string | Slack channel ID or user ID | Yes |
| `contextAboutChannelOrUser` | string | Context for the LLM about the channel or user | No |
| `allowedResponderIDs` | []string | IDs of users allowed to respond | No |

#### EmailChannelConfig
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `address` | string | Recipient email address | Yes |
| `contextAboutUser` | string | Context for the LLM about the recipient | No |
| `subject` | string | Custom email subject line | No |

### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `ready` | boolean | Whether the ContactChannel is ready to be used |
| `status` | string | Current status: "Ready", "Error", or "Pending" |
| `statusDetail` | string | Detailed status message |
| `humanLayerProject` | string | Project ID from HumanLayer API |

## Common Types

### LocalObjectReference
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `name` | string | Name of the referenced resource | Yes |

### APIKeySource
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `secretKeyRef` | SecretKeyRef | Reference to a secret | Yes |

### SpanContext
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `traceID` | string | Trace ID for the span | No |
| `spanID` | string | Span ID | No |

### Message
| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `role` | string | Role of the message sender: "system", "user", "assistant", or "tool" | Yes |
| `content` | string | Message content | Yes |
| `name` | string | Name of the tool that was called | No |
| `toolCallId` | string | Unique identifier for this tool call | No |
| `toolCalls` | []ToolCall | Tool calls requested by this message | No |