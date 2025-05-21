# LLM Providers Guide

This document provides detailed information about configuring different LLM providers in the Agent Control Plane.

## Supported Providers

ACP supports the following LLM providers:

- OpenAI
- Anthropic
- Vertex AI (Google Cloud)
- Mistral
- Google AI

## LLM Configuration Structure

The LLM custom resource has a flexible configuration structure to support multiple providers:

```yaml
apiVersion: acp.humanlayer.dev/v1alpha1 
kind: LLM
metadata:
  name: my-llm
spec:
  # Required: The LLM provider name
  provider: openai  # One of: openai, anthropic, vertex, mistral, google

  # Required for all providers: API key or credentials
  apiKeyFrom:
    secretKeyRef:
      name: my-secret
      key: API_KEY

  # Common configuration options shared across providers
  parameters:
    model: "gpt-4o"         # Model name/id
    baseURL: "https://..."  # Optional API endpoint URL
    temperature: "0.7"      # Temperature (0.0-1.0)
    maxTokens: 1000         # Maximum tokens to generate
    topP: "0.95"            # Controls diversity via nucleus sampling (0.0-1.0)
    topK: 40                # Controls diversity by limiting top K tokens
    frequencyPenalty: "0.5" # Reduces repetition by penalizing frequent tokens
    presencePenalty: "0.0"  # Reduces repetition by penalizing tokens that appear at all

  # Provider-specific configuration (only specify the one matching your provider)
  openai:
    organization: "org-123456"
    apiType: "OPEN_AI"      # One of: OPEN_AI, AZURE, AZURE_AD
    apiVersion: "2023-05-15" # Required for Azure API types
    
  # Or for other providers:
  # anthropic:
  #   anthropicBetaHeader: "max-tokens-3-5-sonnet-2024-07-15"
  #
  # vertex:
  #   cloudProject: "my-gcp-project"
  #   cloudLocation: "us-central1"
  #
  # mistral:
  #   maxRetries: 3
  #   timeout: 60
  #   randomSeed: 42
  #
  # google:
  #   cloudProject: "my-gcp-project"
  #   cloudLocation: "us-central1"
```

## Provider-Specific Requirements

### OpenAI

```yaml
spec:
  provider: openai
  apiKeyFrom:
    secretKeyRef:
      name: openai
      key: OPENAI_API_KEY
  parameters:
    model: "gpt-4o"
    temperature: "0.7"
  openai:
    organization: "org-123456"  # Optional: Your OpenAI organization ID
    apiType: "OPEN_AI"          # Optional: One of "OPEN_AI", "AZURE", or "AZURE_AD"
    apiVersion: "2023-05-15"    # Required for Azure API types
```

### Anthropic

```yaml
spec:
  provider: anthropic
  apiKeyFrom:
    secretKeyRef:
      name: anthropic
      key: ANTHROPIC_API_KEY
  parameters:
    model: "claude-3-5-sonnet-20240620"
    temperature: "0.5"
  anthropic:
    anthropicBetaHeader: "max-tokens-3-5-sonnet-2024-07-15"  # Optional: For extended features
```

### Vertex AI

```yaml
spec:
  provider: vertex
  apiKeyFrom:
    secretKeyRef:
      name: vertex-credentials
      key: service-account-json  # Contains GCP service account JSON
  parameters:
    model: "gemini-pro"
    temperature: "0.7"
    maxTokens: 2048
    topP: "0.95"
    topK: 40
  vertex:
    cloudProject: "my-gcp-project"  # Required: GCP project ID
    cloudLocation: "us-central1"    # Required: GCP region
```

Vertex AI requires a Google Cloud service account with appropriate permissions. The `apiKeyFrom` secret should contain the full service account JSON credentials, not just an API key. Both `cloudProject` and `cloudLocation` are required parameters for Vertex AI.

### Mistral

```yaml
spec:
  provider: mistral
  apiKeyFrom:
    secretKeyRef:
      name: mistral
      key: MISTRAL_API_KEY
  parameters:
    model: "mistral-large-latest"
    temperature: "0.7"
    maxTokens: 1000
    topP: "0.95"
  mistral:
    maxRetries: 3       # Optional: Number of retries for API calls
    timeout: 60         # Optional: Timeout in seconds
    randomSeed: 42      # Optional: Seed for deterministic sampling
```

### Google AI

```yaml
spec:
  provider: google
  apiKeyFrom:
    secretKeyRef:
      name: google
      key: GOOGLE_API_KEY
  parameters:
    model: "gemini-pro"
    temperature: "0.7"
    maxTokens: 2048
    topP: "0.95"
    topK: 40    # Particularly useful for Google's models
  google:
    cloudProject: "my-gcp-project"  # Optional: GCP project ID
    cloudLocation: "us-central1"    # Optional: GCP region
```

Google AI uses a standard API key for authentication. The topK parameter is particularly useful with Google's models for controlling output diversity by limiting the number of tokens considered during sampling.

## Credential Handling

Each provider has different credential requirements:

| Provider   | Credential Type      | Secret Key Reference       |
|------------|----------------------|---------------------------|
| OpenAI     | API Key              | `apiKeyFrom.secretKeyRef` |
| Anthropic  | API Key              | `apiKeyFrom.secretKeyRef` |
| Vertex     | Service Account JSON | `apiKeyFrom.secretKeyRef` |
| Mistral    | API Key              | `apiKeyFrom.secretKeyRef` |
| Google     | API Key              | `apiKeyFrom.secretKeyRef` |

### Secret Examples

OpenAI/Anthropic/Mistral/Google:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openai
type: Opaque
data:
  OPENAI_API_KEY: base64-encoded-api-key
```

Vertex AI:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vertex-credentials
type: Opaque
data:
  service-account-json: base64-encoded-service-account-json
```