# Model Provider Agnostic Client Research Report

## Executive Summary

This research investigates adding a model provider agnostic client to the Agent Framework to support multiple AI providers (Google GenAI/Gemini, Anthropic Claude, OpenRouter, Mistral) in addition to the existing Azure OpenAI and OpenAI clients. The goal is to enable developers to seamlessly switch between different model providers while maintaining full integration with the framework's features, including the DevUI.

## Current Architecture Analysis

### .NET Architecture

The .NET Agent Framework is built around a layered architecture with clear abstractions:

#### Core Abstractions
- **`AIAgent`** (Microsoft.Agents.AI.Abstractions): Abstract base class for all AI agents
- **`IChatClient`** (Microsoft.Extensions.AI): Provider-agnostic interface for chat completions
- **`ChatClientAgent`** (Microsoft.Agents.AI): Concrete agent implementation that wraps any `IChatClient`

#### Existing Provider Implementations
| Package | Provider | Implementation Pattern |
|---------|----------|------------------------|
| Microsoft.Agents.AI.OpenAI | OpenAI | Extension methods on OpenAI SDK clients → `IChatClient` → `ChatClientAgent` |
| Microsoft.Agents.AI.AzureAI | Azure AI | `AzureAIProjectChatClient` implementing `IChatClient` |
| Microsoft.Agents.AI.Anthropic | Anthropic | Extension methods on `IAnthropicClient` → `IChatClient` → `ChatClientAgent` |

#### Key Insight
The framework already leverages **Microsoft.Extensions.AI's `IChatClient`** as its provider-agnostic abstraction. Any provider implementing `IChatClient` can be used with `ChatClientAgent`, which handles:
- Thread/conversation management
- Function invocation
- Message storage
- Context providers
- Streaming support

### Python Architecture

The Python Agent Framework follows a similar pattern:

#### Core Abstractions
- **`ChatClientProtocol`**: Protocol defining the chat client interface
- **`BaseChatClient`**: Abstract base class with middleware support
- **`ChatAgent`**: Agent class that wraps any `BaseChatClient` implementation

#### Existing Provider Implementations
| Package | Provider | Implementation |
|---------|----------|---------------|
| agent_framework (core) | OpenAI | `OpenAIChatClient` extending `BaseChatClient` |
| agent_framework (core) | Anthropic | Re-exported from `agent_framework_anthropic` |
| agent_framework_anthropic | Anthropic | `AnthropicClient` extending `BaseChatClient` |
| agent_framework_azure_ai | Azure AI | `AzureAIAgentClient` extending `BaseChatClient` |

### DevUI Integration

The DevUI (Microsoft.Agents.AI.DevUI) integrates with agents through:

1. **OpenAI Responses API hosting** (Microsoft.Agents.AI.Hosting.OpenAI)
   - Maps OpenAI Responses API format to internal agent operations
   - Uses `AIAgentResponseExecutor` to execute against any `AIAgent`
   
2. **Key Finding**: DevUI doesn't care about the underlying provider - it works with the `AIAgent` abstraction. Any new provider that produces a valid `ChatClientAgent` (or implements `AIAgent`) will work with DevUI.

## Provider-Agnostic Solutions Research

### 1. Microsoft.Extensions.AI (M.E.AI)

**Status**: Already in use by the framework

**Description**: Microsoft's official abstraction layer for AI services in .NET, providing:
- `IChatClient` interface for chat completions
- `IEmbeddingGenerator` interface for embeddings
- Middleware/decorator pattern support
- Multi-modal content support

**Available Implementations**:
- `Microsoft.Extensions.AI.OpenAI` - OpenAI and Azure OpenAI
- `Microsoft.Extensions.AI.Ollama` - Local models via Ollama
- Community implementations for other providers

**Pros**:
- Official Microsoft support
- Already integrated into the framework
- Mature middleware ecosystem
- Dependency injection friendly

**Cons**:
- Not all providers have first-party M.E.AI implementations
- Some providers may require community/third-party implementations

### 2. LiteLLM (Python)

**Status**: Potential solution for Python side

**Description**: Python library providing unified API for 100+ LLMs including:
- OpenAI (GPT-3.5, GPT-4, GPT-4o)
- Anthropic (Claude 2, Claude 3)
- Google (Gemini, Vertex AI)
- Mistral
- Open-source models (Llama, etc.)

**Key Features**:
- Drop-in OpenAI compatibility
- Automatic retries and fallbacks
- Cost tracking
- Unified error handling

**Integration Approach**:
```python
# LiteLLM provides OpenAI-compatible interface
import litellm

response = litellm.completion(
    model="anthropic/claude-3-sonnet",
    messages=[{"role": "user", "content": "Hello"}]
)
```

**Pros**:
- Massive model coverage
- Single dependency
- Active maintenance
- OpenAI API compatibility

**Cons**:
- Another dependency to manage
- May lag behind provider SDK updates
- Abstraction overhead

### 3. OpenRouter

**Status**: Infrastructure solution (not a library)

**Description**: Universal AI gateway service providing:
- Single API endpoint for 400+ models
- OpenAI API compatibility
- Unified billing
- Provider failover

**Integration Approach**:
```python
# Use existing OpenAI client with OpenRouter
from openai import OpenAI

client = OpenAI(
    api_key="OPENROUTER_API_KEY",
    base_url="https://openrouter.ai/api/v1"
)
```

**Pros**:
- Zero code changes (just endpoint swap)
- Works with existing OpenAI implementations
- Handles provider switching at infrastructure level

**Cons**:
- External service dependency
- Additional cost/latency
- Requires internet connectivity
- Not suitable for air-gapped environments

## Provider SDK Analysis

### Google GenAI (Gemini)

**Python SDK**: `google-genai`
```python
from google import genai

client = genai.Client(api_key='YOUR_API_KEY')
response = client.models.generate_content(
    model='gemini-2.5-pro',
    contents=[{"role": "user", "content": "Hello"}]
)
```

**OpenAI Compatibility Mode**:
```python
from openai import OpenAI

client = OpenAI(
    api_key="GEMINI_API_KEY",
    base_url="https://generativelanguage.googleapis.com/v1beta/openai/"
)
```

**.NET**: No official M.E.AI implementation exists. Community implementations or direct SDK integration needed.

### Anthropic Claude

**Python SDK**: `anthropic` (already supported via `agent_framework_anthropic`)

**.NET**: 
- Community SDK: `Anthropic.SDK` with M.E.AI support
- Framework already has `Microsoft.Agents.AI.Anthropic` package

### Mistral AI

**Python SDK**: `mistralai`
```python
from mistralai import Mistral

client = Mistral(api_key="MISTRAL_API_KEY")
response = client.chat.complete(
    model="mistral-medium-latest",
    messages=[{"role": "user", "content": "Hello"}]
)
```

**.NET**: No official M.E.AI implementation. Would require custom implementation.

## Recommended Implementation Strategy

### Phase 1: Leverage OpenAI API Compatibility

Many providers offer OpenAI-compatible endpoints:
- **Google Gemini**: `https://generativelanguage.googleapis.com/v1beta/openai/`
- **OpenRouter**: `https://openrouter.ai/api/v1`
- **Mistral**: Provides OpenAI-compatible endpoint

**Implementation**:

1. **Python**: Create a `GenericOpenAIChatClient` that accepts custom base URLs:

```python
class GenericOpenAIChatClient(OpenAIChatClient):
    """OpenAI-compatible client for any provider with OpenAI API compatibility."""
    
    def __init__(
        self,
        *,
        model_id: str,
        api_key: str,
        base_url: str,  # Provider's OpenAI-compatible endpoint
        **kwargs
    ):
        # Configure with custom endpoint
        pass
```

2. **.NET**: The existing OpenAI implementation already supports custom base URLs. Document this pattern:

```csharp
// Using OpenAI SDK with custom endpoint
var client = new OpenAIClient(
    new ApiKeyCredential("PROVIDER_API_KEY"),
    new OpenAIClientOptions { Endpoint = new Uri("https://provider.api/v1") }
);

var chatClient = client.GetChatClient("model-id").AsIChatClient();
var agent = new ChatClientAgent(chatClient);
```

### Phase 2: Native Provider Support

For providers requiring native SDK features:

1. **Google GenAI (.NET)**:
   - Create `Microsoft.Agents.AI.GoogleAI` package
   - Implement `IChatClient` wrapper around Google GenAI SDK

2. **Mistral (.NET)**:
   - Create `Microsoft.Agents.AI.Mistral` package  
   - Implement `IChatClient` wrapper around Mistral SDK

3. **Python**:
   - Create `agent_framework_google` package with `GoogleGenAIChatClient`
   - Create `agent_framework_mistral` package with `MistralChatClient`

### Phase 3: LiteLLM Integration (Python)

Create an optional `agent_framework_litellm` package:

```python
@use_function_invocation
@use_observability
@use_chat_middleware
class LiteLLMChatClient(BaseChatClient):
    """Universal chat client using LiteLLM for 100+ model support."""
    
    def __init__(self, model: str, **kwargs):
        self.model = model
        self.kwargs = kwargs
    
    async def _inner_get_response(self, *, messages, chat_options, **kwargs):
        response = await litellm.acompletion(
            model=self.model,
            messages=self._convert_messages(messages),
            **self._merge_options(chat_options)
        )
        return self._convert_response(response)
```

## DevUI Integration Assessment

### Current State
DevUI works with any `AIAgent` implementation through the OpenAI Responses API hosting layer. The key integration points are:

1. `IResponseExecutor` - Executes responses (implemented by `AIAgentResponseExecutor`)
2. `IResponsesService` - Manages response storage
3. OpenAI-format request/response mapping

### New Provider Compatibility
✅ **All new providers will work with DevUI** because:
- DevUI operates at the `AIAgent` abstraction level
- Any provider implementing `IChatClient` → `ChatClientAgent` → DevUI
- No DevUI changes required for new providers

### Verification Checklist for New Providers
1. Provider returns proper `ChatMessage` format
2. Streaming responses yield correct `ChatResponseUpdate` objects
3. Function/tool calling follows `FunctionCallContent`/`FunctionResultContent` patterns
4. Token usage reporting (optional but recommended)

## Feature Compatibility Matrix

| Feature | OpenAI | Azure OpenAI | Anthropic | Google GenAI | Mistral | OpenRouter |
|---------|--------|--------------|-----------|--------------|---------|------------|
| Chat Completion | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Streaming | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Function Calling | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Multi-modal (Images) | ✅ | ✅ | ✅ | ✅ | ❌ | Varies |
| Structured Output | ✅ | ✅ | ✅ | ✅ | ✅ | Varies |
| Token Usage | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| DevUI Compatible | ✅ | ✅ | ✅ | ✅* | ✅* | ✅* |

*With proper implementation following framework patterns

## Recommendations

### Short-term (Immediate)
1. **Document OpenAI-compatible endpoint usage** for Google Gemini, Mistral, and OpenRouter
2. **Add configuration examples** showing how to use custom base URLs
3. **Create samples** demonstrating multi-provider scenarios

### Medium-term (1-3 months)
1. **Python**: Create `agent_framework_litellm` package for universal provider support
2. **.NET**: Create thin wrapper packages for Google GenAI and Mistral
3. **Add provider selection guidance** in documentation

### Long-term (3-6 months)
1. **Evaluate M.E.AI ecosystem growth** for new provider implementations
2. **Consider model capability detection** for automatic feature negotiation
3. **Build provider comparison/benchmarking tools**

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Provider API changes | Breaking changes | Pin SDK versions, version-specific adapters |
| Feature parity gaps | Reduced functionality | Feature detection, graceful degradation |
| Maintenance burden | Technical debt | Focus on OpenAI-compatible approach first |
| Performance overhead | Latency | Benchmark abstractions, optimize hot paths |

## Conclusion

The Agent Framework's existing architecture, built on `IChatClient` (M.E.AI) for .NET and `BaseChatClient` for Python, provides an excellent foundation for model provider agnostic support. The recommended approach prioritizes:

1. **OpenAI API compatibility** - Minimal code changes, broad coverage
2. **Native SDK wrappers** - When specific features are needed
3. **LiteLLM integration** (Python) - Maximum flexibility

This strategy ensures:
- ✅ Full DevUI compatibility
- ✅ Minimal framework changes
- ✅ Support for Google GenAI, Claude, Mistral, OpenRouter
- ✅ Future extensibility for new providers

## Appendix A: Implementation Examples

### Example: Using Google Gemini via OpenAI Compatibility

**.NET**:
```csharp
using Microsoft.Extensions.AI;
using OpenAI;

// Create OpenAI client pointing to Gemini's OpenAI-compatible endpoint
var options = new OpenAIClientOptions 
{ 
    Endpoint = new Uri("https://generativelanguage.googleapis.com/v1beta/openai/") 
};
var openAiClient = new OpenAIClient(new ApiKeyCredential(geminiApiKey), options);
var chatClient = openAiClient.GetChatClient("gemini-2.5-flash").AsIChatClient();

// Create agent using existing framework
var agent = new ChatClientAgent(
    chatClient,
    instructions: "You are a helpful assistant."
);

// Use with DevUI - works automatically!
var response = await agent.RunAsync("Hello, how are you?");
```

**Python**:
```python
from openai import AsyncOpenAI
from agent_framework import ChatAgent

# Create OpenAI client with Gemini endpoint
client = AsyncOpenAI(
    api_key="GEMINI_API_KEY",
    base_url="https://generativelanguage.googleapis.com/v1beta/openai/"
)

# Wrap in framework's OpenAI client
from agent_framework.openai import OpenAIChatClient

chat_client = OpenAIChatClient(
    model_id="gemini-2.5-flash",
    async_client=client
)

# Create agent
agent = chat_client.create_agent(
    name="assistant",
    instructions="You are a helpful assistant."
)

# Works with DevUI!
response = await agent.run("Hello!")
```

### Example: Using OpenRouter

**.NET**:
```csharp
var options = new OpenAIClientOptions 
{ 
    Endpoint = new Uri("https://openrouter.ai/api/v1") 
};
var openAiClient = new OpenAIClient(new ApiKeyCredential(openRouterApiKey), options);
var chatClient = openAiClient.GetChatClient("anthropic/claude-3-sonnet").AsIChatClient();

var agent = new ChatClientAgent(chatClient);
```

**Python**:
```python
from openai import AsyncOpenAI
from agent_framework.openai import OpenAIChatClient

client = AsyncOpenAI(
    api_key="OPENROUTER_API_KEY",
    base_url="https://openrouter.ai/api/v1"
)

chat_client = OpenAIChatClient(
    model_id="anthropic/claude-3-sonnet",
    async_client=client
)
```

## Appendix B: Proposed Package Structure

### .NET Packages (Future)
```
Microsoft.Agents.AI.GoogleAI      - Google GenAI/Gemini support
Microsoft.Agents.AI.Mistral       - Mistral AI support
Microsoft.Agents.AI.OpenRouter    - OpenRouter convenience wrapper
```

### Python Packages (Future)
```
agent_framework_google    - Google GenAI/Gemini support
agent_framework_mistral   - Mistral AI support  
agent_framework_litellm   - LiteLLM universal provider support
```

---

*Report prepared for Agent Framework repository*
*Date: December 2025*
