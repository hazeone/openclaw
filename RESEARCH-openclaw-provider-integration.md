# OpenClaw Provider Integration Deep Research

## Overview

This document provides a deep analysis of how OpenClaw supports Chinese AI providers
(MiniMax, GLM/Z.AI, Moonshot/Kimi, Qwen, DeepSeek, Doubao/Volcengine, BytePlus,
Xiaomi, Qianfan/Baidu) and their "Coding Plan" subscription tiers — along with a
concrete guide for integrating this functionality into **ValueCell-ai/ClawX**.

---

## 1. Architecture Summary

OpenClaw's provider system has **four layers**:

```
┌──────────────────────────────────────────────────────────┐
│  Layer 4: Onboarding / CLI Auth Flows                    │
│  (src/commands/auth-choice.apply.*.ts, onboard-auth.ts)  │
├──────────────────────────────────────────────────────────┤
│  Layer 3: Model Resolution + Forward Compat              │
│  (src/agents/pi-embedded-runner/model.ts,                │
│   model-forward-compat.ts, model-selection.ts)           │
├──────────────────────────────────────────────────────────┤
│  Layer 2: Provider Config Builders + Auto-Discovery      │
│  (src/agents/models-config.providers.ts)                 │
├──────────────────────────────────────────────────────────┤
│  Layer 1: Core Types + Plugin API                        │
│  (src/config/types.models.ts, src/plugins/types.ts)      │
└──────────────────────────────────────────────────────────┘
```

### Core Types

**`ModelProviderConfig`** (the fundamental provider definition):

```typescript
type ModelProviderConfig = {
  baseUrl: string;           // API endpoint
  apiKey?: string;           // API key or env var name
  auth?: "api-key" | "aws-sdk" | "oauth" | "token";
  api?: ModelApi;            // Protocol: "openai-completions" | "anthropic-messages" | ...
  headers?: Record<string, string>;
  authHeader?: boolean;
  models: ModelDefinitionConfig[];
};
```

**`ModelDefinitionConfig`** (per-model metadata):

```typescript
type ModelDefinitionConfig = {
  id: string;                // Model ID sent to API (e.g. "MiniMax-M2.1")
  name: string;              // Display name
  api?: ModelApi;            // Per-model API override
  reasoning: boolean;        // Supports reasoning/thinking mode
  input: Array<"text" | "image">;
  cost: { input: number; output: number; cacheRead: number; cacheWrite: number };
  contextWindow: number;
  maxTokens: number;
  headers?: Record<string, string>;
  compat?: ModelCompatConfig; // Provider-specific quirks
};
```

**`ModelApi`** (supported API protocols):

```typescript
type ModelApi =
  | "openai-completions"          // OpenAI Chat Completions API
  | "openai-responses"            // OpenAI Responses API
  | "anthropic-messages"          // Anthropic Messages API
  | "google-generative-ai"        // Google Gemini API
  | "github-copilot"              // GitHub Copilot API
  | "bedrock-converse-stream"     // AWS Bedrock
  | "ollama";                     // Ollama native API
```

**`ModelCompatConfig`** (provider-specific compatibility flags):

```typescript
type ModelCompatConfig = {
  supportsStore?: boolean;
  supportsDeveloperRole?: boolean;     // Z.AI/Moonshot force this to false
  supportsReasoningEffort?: boolean;
  supportsUsageInStreaming?: boolean;
  supportsStrictMode?: boolean;
  maxTokensField?: "max_completion_tokens" | "max_tokens";
  thinkingFormat?: "openai" | "zai" | "qwen";  // Thinking token format
  requiresToolResultName?: boolean;
  requiresAssistantAfterToolResult?: boolean;
  requiresThinkingAsText?: boolean;
  requiresMistralToolIds?: boolean;
};
```

---

## 2. Chinese AI Provider Details

### 2.1 MiniMax (`minimax`)

**Protocol:** `anthropic-messages` (Anthropic-compatible API)

**Base URL:** `https://api.minimax.io/anthropic`

**Auth:** `MINIMAX_API_KEY` or `MINIMAX_CODE_PLAN_KEY` env vars, or auth profile

**Models:**
| Model ID | Name | Reasoning | Input | Context | Max Tokens |
|---|---|---|---|---|---|
| `MiniMax-M2.1` | MiniMax M2.1 | false | text | 200K | 8192 |
| `MiniMax-M2.1-lightning` | MiniMax M2.1 Lightning | false | text | 200K | 8192 |
| `MiniMax-VL-01` | MiniMax VL 01 | false | text, image | 200K | 8192 |
| `MiniMax-M2.5` | MiniMax M2.5 | true | text | 200K | 8192 |
| `MiniMax-M2.5-Lightning` | MiniMax M2.5 Lightning | true | text | 200K | 8192 |

**Pricing:** $0.30/1M input, $1.20/1M output, $0.03/1M cache read, $0.12/1M cache write

**Special Features:**
- **Coding Plan subscription:** MiniMax offers a "Coding Plan" subscription tier
  - Usage tracking API: `https://api.minimaxi.com/v1/api/openplatform/coding_plan/remains`
  - VLM (Vision) endpoint: `/v1/coding_plan/vlm` for image understanding
  - OAuth portal authentication via `minimax-portal-auth` extension plugin
- **Tool call XML stripping:** MiniMax sometimes wraps tool calls in XML
- **Auto-routing:** Lightning backend auto-selected by MiniMax during normal load
- **Two auth paths:**
  - API key: direct `MINIMAX_API_KEY`
  - OAuth: `minimax-portal-auth` plugin (device code flow, Global + CN endpoints)

**OAuth Plugin (`extensions/minimax-portal-auth/`):**
- Registers `minimax-portal` provider ID
- Two regions: Global (`api.minimax.io`) and CN (`api.minimaxi.com`)
- Uses device code OAuth flow
- Auto-configures `anthropic-messages` API

### 2.2 Z.AI / GLM (`zai`)

**Protocol:** `openai-completions` (OpenAI-compatible)

**Base URLs:**
- Global: `https://api.z.ai/api/paas/v4`
- CN: `https://open.bigmodel.cn/api/paas/v4`
- Coding Global: `https://api.z.ai/api/paas/v4/coding`
- Coding CN: `https://open.bigmodel.cn/api/paas/v4/coding`

**Auth:** `ZAI_API_KEY` or `Z_AI_API_KEY` env vars

**Models:**
| Model ID | Reasoning | Notes |
|---|---|---|
| `glm-5` | true | Newest, preferred on general API |
| `glm-4.7` | true | Default on Coding Plan endpoints |
| `glm-4.7-flashx` | false | Fast variant |
| `glm-4.6` | false | Older |
| `glm-4.6v` | false | Vision model |

**Special Features:**
- **Coding Plan endpoints:** Separate `/coding` URL paths with different model availability
- **Endpoint auto-detection:** `detectZaiEndpoint()` probes each endpoint to find the best one:
  1. First tries GLM-5 on general Global/CN endpoints
  2. Falls back to GLM-4.7 on Coding Plan endpoints
- **Forward compatibility:** GLM-5 -> GLM-4.7 fallback if GLM-5 not in catalog yet
- **Compatibility flag:** `supportsDeveloperRole: false` auto-set in `normalizeModelCompat()`
- **Thinking format:** `thinkingFormat: "zai"` for reasoning tokens
- **Usage tracking:** `https://api.z.ai/api/monitor/usage/quota/limit`
- **Four endpoint choices during onboarding:**
  - Coding-Plan-Global
  - Coding-Plan-CN
  - Global
  - CN

### 2.3 Moonshot / Kimi (`moonshot` + `kimi-coding`)

OpenClaw supports Moonshot through **two separate providers**:

**Moonshot (`moonshot`):**
- Protocol: `openai-completions`
- Base URL: `https://api.moonshot.ai/v1` (Global) or `https://api.moonshot.cn/v1` (CN)
- Auth: `MOONSHOT_API_KEY`
- Default model: `kimi-k2.5`
- Context: 256K, Max tokens: 8192
- Supports text + image input

**Kimi Coding (`kimi-coding`):**
- Protocol: `anthropic-messages`
- Base URL: `https://api.kimi.com/coding/`
- Auth: `KIMI_API_KEY`
- Default model: `k2p5`
- Context: 262K, Max tokens: 32768
- Reasoning: true
- A dedicated coding-optimized endpoint

**Special Features:**
- Web search tool integration
- `supportsDeveloperRole: false` for Moonshot models
- Separate Global (.ai) and CN (.cn) endpoints

### 2.4 Qwen (`qwen-portal`)

**Protocol:** `openai-completions`

**Base URL:** `https://portal.qwen.ai/v1`

**Auth:** OAuth (device code flow) via `qwen-portal-auth` extension plugin

**Models:**
- `coder-model` (Qwen Coder) - text only
- `vision-model` (Qwen Vision) - text + image

**Special Features:**
- Free tier: 2000 requests/day
- OAuth credential sync from `~/.qwen/oauth_creds.json`
- `thinkingFormat: "qwen"` for reasoning tokens

### 2.5 Volcengine / Doubao (`volcengine` + `volcengine-plan`)

**Protocol:** `openai-completions`

**Two provider variants:**
- `volcengine`: General endpoint (`https://ark.cn-beijing.volces.com/api/v3`)
- `volcengine-plan`: Coding endpoint (`https://ark.cn-beijing.volces.com/api/coding/v3`)

**Auth:** `VOLCANO_ENGINE_API_KEY` or `VOLCENGINE_API_KEY`

**General models:** doubao-seed-1-8, doubao-seed-code-preview, kimi-k2-5, glm-4-7, deepseek-v3-2

**Coding Plan models:** ark-code-latest, doubao-seed-code, kimi-k2.5, kimi-k2-thinking, glm-4.7

### 2.6 BytePlus (`byteplus` + `byteplus-plan`)

**Protocol:** `openai-completions`

Same as Volcengine but international:
- General: `https://ark-us-east-1.bytedance.com/api/v3`
- Coding: `https://ark-us-east-1.bytedance.com/api/coding/v3`

**Auth:** `BYTEPLUS_API_KEY`

### 2.7 Xiaomi MiMo (`xiaomi`)

**Protocol:** `anthropic-messages`

**Base URL:** `https://api.xiaomimimo.com/anthropic`

**Auth:** `XIAOMI_API_KEY`

**Model:** `mimo-v2-flash` (262K context, 8192 max tokens)

### 2.8 Qianfan / Baidu (`qianfan`)

**Protocol:** `openai-completions`

**Base URL:** `https://qianfan.baidubce.com/v2`

**Auth:** `QIANFAN_API_KEY` (format: `bce-v3/ALTAK-...`)

**Models:**
- `deepseek-v3.2` (98K context, 32K max tokens, reasoning)
- `ernie-5.0-thinking-preview` (119K context, 64K max tokens, reasoning, image)

---

## 3. What is "Coding Plan"?

"Coding Plan" is **not** a feature in OpenClaw itself — it's a **subscription/product tier**
offered by several Chinese AI providers:

1. **MiniMax Coding Plan:** A subscription that provides API access to MiniMax models
   for coding tasks. Has its own usage tracking endpoint, VLM endpoint, and OAuth portal.

2. **Z.AI/GLM Coding Plan:** A subscription with dedicated `/coding` API endpoints.
   Different model availability (GLM-4.7 instead of GLM-5).

3. **Volcengine/Doubao Coding Plan (`volcengine-plan`):** A separate coding endpoint
   (`/api/coding/v3`) with coding-focused models like `ark-code-latest`.

4. **BytePlus Coding Plan (`byteplus-plan`):** Same as Volcengine but via BytePlus
   international endpoints.

OpenClaw integrates with these Coding Plan tiers by:
- Supporting separate base URLs for the coding endpoints
- Tracking usage via provider-specific APIs
- Auto-detecting which endpoint the user's API key is valid for (Z.AI)
- Providing separate provider IDs (e.g. `volcengine` vs `volcengine-plan`)

---

## 4. Provider Registration Mechanisms

### 4.1 Built-in Provider Builders

Each provider has a `buildXxxProvider()` function in `src/agents/models-config.providers.ts`
that returns a `ProviderConfig` with base URL, API type, and model catalog.

### 4.2 Auto-Discovery (`resolveImplicitProviders()`)

When the gateway starts, OpenClaw automatically discovers providers by checking:

1. **Environment variables:** e.g. `MINIMAX_API_KEY`, `ZAI_API_KEY`, `MOONSHOT_API_KEY`
2. **Auth profiles:** stored in `~/.openclaw/agents/<agentId>/auth-profiles.json`

If credentials are found, the provider is automatically registered with its default
model catalog and the discovered API key.

### 4.3 Plugin Registration

Plugins can register providers via `api.registerProvider()`:

```typescript
api.registerProvider({
  id: "minimax-portal",
  label: "MiniMax",
  aliases: ["minimax"],
  auth: [{
    id: "oauth",
    label: "MiniMax OAuth (Global)",
    kind: "device_code",
    run: async (ctx) => { /* OAuth flow */ },
  }],
});
```

### 4.4 User Configuration

Users can define custom providers in `openclaw.json`:

```json5
{
  models: {
    mode: "merge",
    providers: {
      minimax: {
        baseUrl: "https://api.minimax.io/anthropic",
        apiKey: "${MINIMAX_API_KEY}",
        api: "anthropic-messages",
        models: [{ id: "MiniMax-M2.1", name: "MiniMax M2.1", ... }],
      },
    },
  },
}
```

---

## 5. Model Resolution Flow

When a user requests a model like `minimax/MiniMax-M2.1`:

```
1. normalizeProviderId("minimax") → "minimax"
   (aliases: "z.ai"→"zai", "qwen"→"qwen-portal", "kimi-code"→"kimi-coding", etc.)

2. modelRegistry.find("minimax", "MiniMax-M2.1")
   - Checks pi-ai built-in catalog
   - Checks user's models.json overrides

3. If not found → check inline provider models from config
   (buildInlineProviderModels from models.providers)

4. If not found → check forward-compat fallbacks
   (e.g. glm-5 → glm-4.7 template)

5. If not found → OpenRouter dynamic fallback (for openrouter provider)

6. If not found → generic fallback from provider config

7. normalizeModelCompat() applies provider-specific quirks:
   - Z.AI, Moonshot, DashScope: supportsDeveloperRole = false
   - Anthropic: strip /v1 from baseUrl
```

---

## 6. Integration Guide for ValueCell-ai/ClawX

### Step 1: Define Core Types

Port the type system from `src/config/types.models.ts`:

```typescript
// Core types you need
type ModelApi = "openai-completions" | "anthropic-messages" | /* others as needed */;

type ModelDefinitionConfig = {
  id: string;
  name: string;
  api?: ModelApi;
  reasoning: boolean;
  input: Array<"text" | "image">;
  cost: { input: number; output: number; cacheRead: number; cacheWrite: number };
  contextWindow: number;
  maxTokens: number;
  compat?: ModelCompatConfig;
};

type ModelProviderConfig = {
  baseUrl: string;
  apiKey?: string;
  api?: ModelApi;
  models: ModelDefinitionConfig[];
};
```

### Step 2: Implement Provider Builders

For each Chinese provider you want to support, create a builder function.
Here are the key ones:

**MiniMax:**
```typescript
function buildMinimaxProvider(): ModelProviderConfig {
  return {
    baseUrl: "https://api.minimax.io/anthropic",
    api: "anthropic-messages",
    models: [
      { id: "MiniMax-M2.1", name: "MiniMax M2.1", reasoning: false, input: ["text"],
        cost: { input: 0.3, output: 1.2, cacheRead: 0.03, cacheWrite: 0.12 },
        contextWindow: 200000, maxTokens: 8192 },
      { id: "MiniMax-M2.5", name: "MiniMax M2.5", reasoning: true, input: ["text"],
        cost: { input: 0.3, output: 1.2, cacheRead: 0.03, cacheWrite: 0.12 },
        contextWindow: 200000, maxTokens: 8192 },
    ],
  };
}
```

**Z.AI/GLM:**
```typescript
function buildZaiProvider(endpoint: "global" | "cn" | "coding-global" | "coding-cn"): ModelProviderConfig {
  const baseUrls = {
    "global": "https://api.z.ai/api/paas/v4",
    "cn": "https://open.bigmodel.cn/api/paas/v4",
    "coding-global": "https://api.z.ai/api/paas/v4/coding",
    "coding-cn": "https://open.bigmodel.cn/api/paas/v4/coding",
  };
  const isCoding = endpoint.startsWith("coding");
  return {
    baseUrl: baseUrls[endpoint],
    api: "openai-completions",
    models: isCoding
      ? [{ id: "glm-4.7", name: "GLM 4.7", reasoning: true, ... }]
      : [{ id: "glm-5", name: "GLM 5", reasoning: true, ... },
         { id: "glm-4.7", name: "GLM 4.7", reasoning: true, ... }],
  };
}
```

### Step 3: Implement Provider Auto-Discovery

```typescript
function resolveImplicitProviders(): Record<string, ModelProviderConfig> {
  const providers: Record<string, ModelProviderConfig> = {};

  // MiniMax
  const minimaxKey = process.env.MINIMAX_API_KEY || process.env.MINIMAX_CODE_PLAN_KEY;
  if (minimaxKey) {
    providers.minimax = { ...buildMinimaxProvider(), apiKey: minimaxKey };
  }

  // Z.AI/GLM
  const zaiKey = process.env.ZAI_API_KEY || process.env.Z_AI_API_KEY;
  if (zaiKey) {
    // Optionally: auto-detect endpoint like OpenClaw does
    providers.zai = { ...buildZaiProvider("global"), apiKey: zaiKey };
  }

  // Moonshot
  const moonshotKey = process.env.MOONSHOT_API_KEY;
  if (moonshotKey) {
    providers.moonshot = { ...buildMoonshotProvider(), apiKey: moonshotKey };
  }

  // ... repeat for each provider
  return providers;
}
```

### Step 4: Implement Model Resolution

```typescript
function normalizeProviderId(provider: string): string {
  const n = provider.trim().toLowerCase();
  if (n === "z.ai" || n === "z-ai") return "zai";
  if (n === "qwen") return "qwen-portal";
  if (n === "kimi-code") return "kimi-coding";
  if (n === "bytedance" || n === "doubao") return "volcengine";
  return n;
}

function resolveModel(provider: string, modelId: string, config: Config) {
  const normalized = normalizeProviderId(provider);
  // 1. Check built-in catalog
  // 2. Check user config providers
  // 3. Check forward-compat fallbacks
  // 4. Generic provider fallback
}
```

### Step 5: Handle Provider-Specific Compatibility

Key compat quirks to port:

```typescript
function normalizeModelCompat(model: Model): Model {
  const isZai = model.provider === "zai";
  const isMoonshot = model.provider === "moonshot";
  
  if ((isZai || isMoonshot) && model.api === "openai-completions") {
    // These providers don't support the "developer" role
    model.compat = { ...model.compat, supportsDeveloperRole: false };
  }
  return model;
}
```

### Step 6: Usage Tracking (Optional)

If you want to show usage/quota information:

**MiniMax:** `GET https://api.minimaxi.com/v1/api/openplatform/coding_plan/remains`
- Headers: `Authorization: Bearer <apiKey>`, `MM-API-Source: OpenClaw`

**Z.AI:** `GET https://api.z.ai/api/monitor/usage/quota/limit`
- Headers: `Authorization: Bearer <apiKey>`

### Step 7: Plugin System (Optional)

If you want extensible provider registration:

```typescript
type ProviderPlugin = {
  id: string;
  label: string;
  aliases?: string[];
  envVars?: string[];
  models?: ModelProviderConfig;
  auth: ProviderAuthMethod[];
};

// Allow plugins to register providers
api.registerProvider(plugin: ProviderPlugin);
```

### Step 8: Z.AI Endpoint Auto-Detection (Recommended)

Port the endpoint detection from `src/commands/zai-endpoint-detect.ts`:

```typescript
async function detectZaiEndpoint(apiKey: string): Promise<Endpoint | null> {
  // 1. Try GLM-5 on global endpoint
  // 2. Try GLM-5 on CN endpoint
  // 3. Fall back to GLM-4.7 on coding-global
  // 4. Fall back to GLM-4.7 on coding-cn
  // Returns { endpoint, baseUrl, modelId, note }
}
```

---

## 7. Key Files to Reference

| File | Purpose |
|---|---|
| `src/config/types.models.ts` | Core type definitions |
| `src/agents/models-config.providers.ts` | All provider builders + auto-discovery |
| `src/agents/model-selection.ts` | Provider/model normalization + resolution |
| `src/agents/model-forward-compat.ts` | GLM-5→GLM-4.7 and other fallbacks |
| `src/agents/model-compat.ts` | Provider-specific compat quirks |
| `src/agents/pi-embedded-runner/model.ts` | Main model resolution entry point |
| `src/plugins/types.ts` | Plugin API types (ProviderPlugin) |
| `src/infra/provider-usage.fetch.minimax.ts` | MiniMax usage tracking |
| `src/infra/provider-usage.fetch.zai.ts` | Z.AI usage tracking |
| `src/infra/provider-usage.auth.ts` | Provider auth resolution |
| `src/commands/zai-endpoint-detect.ts` | Z.AI endpoint auto-detection |
| `src/agents/minimax-vlm.ts` | MiniMax VLM (vision) integration |
| `src/agents/doubao-models.ts` | Volcengine/Doubao model catalog |
| `src/agents/byteplus-models.ts` | BytePlus model catalog |
| `extensions/minimax-portal-auth/` | MiniMax OAuth plugin |
| `extensions/qwen-portal-auth/` | Qwen OAuth plugin |
| `docs/providers/minimax.md` | MiniMax docs |
| `docs/providers/zai.md` | Z.AI docs |

---

## 8. Summary of Provider → API Protocol Mapping

| Provider | API Protocol | Key Gotcha |
|---|---|---|
| MiniMax | `anthropic-messages` | XML tool call stripping needed |
| Z.AI/GLM | `openai-completions` | `supportsDeveloperRole: false`, thinking format "zai" |
| Moonshot | `openai-completions` | `supportsDeveloperRole: false`, CN/Global variants |
| Kimi Coding | `anthropic-messages` | Separate endpoint from Moonshot |
| Qwen | `openai-completions` | OAuth only, thinking format "qwen" |
| Volcengine | `openai-completions` | Coding Plan is a separate provider ID |
| BytePlus | `openai-completions` | International Volcengine |
| Xiaomi | `anthropic-messages` | Free tier |
| Qianfan | `openai-completions` | `bce-v3/ALTAK-...` key format |

---

## 9. Minimum Viable Integration Checklist

For ValueCell-ai/ClawX, the minimum steps are:

- [ ] Port `ModelProviderConfig` and `ModelDefinitionConfig` types
- [ ] Implement provider builders for target providers (MiniMax, Z.AI at minimum)
- [ ] Implement `resolveImplicitProviders()` with env var detection
- [ ] Implement `normalizeProviderId()` for alias handling
- [ ] Implement model resolution with inline config fallback
- [ ] Add `supportsDeveloperRole: false` compat for Z.AI/Moonshot
- [ ] Support both `openai-completions` and `anthropic-messages` API protocols
- [ ] (Optional) Z.AI endpoint auto-detection
- [ ] (Optional) Usage tracking for Coding Plan providers
- [ ] (Optional) OAuth plugin system for MiniMax portal / Qwen portal
