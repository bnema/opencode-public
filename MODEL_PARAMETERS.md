# Model Parameters

Scope: OpenCode-facing knobs only.

This file documents what OpenCode config and runtime actually use today for `github-copilot`, `opencode-go`, and `openai`.

Primary source of truth in the OpenCode repo:
- `packages/opencode/src/config/config.ts`
- `packages/opencode/src/agent/agent.ts`
- `packages/opencode/src/session/llm.ts`
- `packages/opencode/src/provider/transform.ts`
- `packages/opencode/src/provider/provider.ts`
- `packages/opencode/src/provider/models.ts`
- `packages/opencode/src/plugin/github-copilot/models.ts`
- `packages/opencode/src/plugin/github-copilot/copilot.ts`
- `packages/sdk/openapi.json`

Secondary source:
- `models.dev` metadata, used only to cross-check provider/model metadata

Non-source:
- custom agent markdown under `~/.config/opencode/agents/` is not authoritative for provider/model behavior

## Accuracy Hierarchy

This reference separates behavior truth from availability truth.

### Behavior Truth

For parameter names, default injection, variant synthesis, and request shaping, trust the OpenCode source code first:
- `config/config.ts`
- `agent/agent.ts`
- `session/llm.ts`
- `provider/transform.ts`
- `provider/provider.ts`
- provider-specific SDK and plugin code

### Availability Truth

For whether a model is actually available right now, use this order:
1. the live model picker or runtime provider list
2. provider/plugin-fetched runtime metadata
3. cached live catalog data
4. bundled snapshot files as a fallback only

Important rule: absence from `models-snapshot.js` is not enough to conclude that a model is unavailable.

## Shared Knobs

### Agent Config

These are the main agent-level knobs OpenCode accepts in config:

| Field | Meaning | Notes |
|---|---|---|
| `model` | Default model for the agent | String model ref in config; parsed into `{ providerID, modelID }` at runtime |
| `variant` | Default model variant | Applies when the agent is using its configured model |
| `temperature` | Sampling temperature | Only sent when `model.capabilities.temperature` is `true` |
| `top_p` | Sampling top-p | Mapped into runtime `topP` |
| `prompt` | Agent prompt override | Plain string |
| `steps` | Max agentic iterations | Replaces deprecated `maxSteps` |
| `options` | Provider/model-specific extra options | Passed through provider option namespacing |

Notes:
- Config uses `top_p`; runtime agent state uses `topP`.
- Unknown extra agent keys are folded into `options` by the config transform.

### Provider Config

These provider-level options are explicitly described in `openapi.json` / config schema:

| Field | Meaning |
|---|---|
| `apiKey` | API key override |
| `baseURL` | Base URL override |
| `enterpriseUrl` | GitHub Enterprise URL for Copilot auth |
| `setCacheKey` | Forces `promptCacheKey` injection for that provider |
| `timeout` | Request timeout in ms, or `false` to disable |
| `chunkTimeout` | SSE chunk timeout in ms |

### Request Assembly Rules

OpenCode assembles request params in `session/llm.ts` and `provider/transform.ts`.

| Field | Rule |
|---|---|
| `temperature` | `agent.temperature ?? ProviderTransform.temperature(model)`, but only if the model capability allows temperature |
| `topP` | `agent.topP ?? ProviderTransform.topP(model)` |
| `topK` | Internal only. Comes from `ProviderTransform.topK(model)` |
| `maxOutputTokens` | `min(model.limit.output, OUTPUT_TOKEN_MAX)` where `OUTPUT_TOKEN_MAX` defaults to `32000` unless `OPENCODE_EXPERIMENTAL_OUTPUT_TOKEN_MAX` overrides it |
| `providerOptions` | Rewritten by `ProviderTransform.providerOptions(model, options)` into the SDK-specific namespace |

### Variant Overrides In `opencode.json`

OpenCode synthesizes variants from `ProviderTransform.variants(model)`, then merges any per-model config overrides on top. Setting `disabled: true` removes a variant.

```json
{
  "provider": {
    "openai": {
      "models": {
        "gpt-5.4-mini": {
          "variants": {
            "xhigh": { "disabled": true },
            "custom": {
              "reasoningEffort": "high",
              "reasoningSummary": "auto"
            }
          }
        }
      }
    }
  }
}
```

## Summary

| Provider | Effective SDK / transport | Main knobs | Synthesized variants | Notable OpenCode defaults | Main caveats |
|---|---|---|---|---|---|
| `github-copilot` | Runtime models use `@ai-sdk/github-copilot` | Agent knobs plus `enterpriseUrl`, timeouts, `options` | Claude: `low`/`medium`/`high`; GPT/Codex: `low`/`medium`/`high` and sometimes `xhigh`; Gemini: none | `store: false`; GPT models omit `maxOutputTokens`; GPT/Codex variants add `reasoningSummary: "auto"` and `include: ["reasoning.encrypted_content"]` | Snapshot provider metadata says openai-compatible, but runtime plugin rewrites models to Copilot SDK behavior |
| `opencode-go` | Usually `@ai-sdk/openai-compatible`, but per-model overrides are possible | Agent knobs plus provider `apiKey` / `baseURL` / timeouts / `options` | Current known `glm`, `minimax`, and `kimi` families synthesize no variants | Family-specific `temperature`, `topP`, and `topK` defaults for some model IDs | Live picker and runtime metadata beat the bundled snapshot for model availability |
| `openai` | `@ai-sdk/openai` | Agent knobs plus provider `apiKey` / `baseURL` / `setCacheKey` / timeouts / `options` | GPT-5 family is dynamic; Codex differs slightly; `gpt-5-pro` gets none | `store: false`; `promptCacheKey`; GPT-5 defaults `reasoningEffort: "medium"` and `reasoningSummary: "auto"`; some GPT-5.x models add `textVerbosity: "low"` | Many reasoning models have `temperature: false`; release date affects available variants |

## github-copilot

### Identity

- Provider ID: `github-copilot`
- Bundled snapshot metadata lists provider `npm` as `@ai-sdk/openai-compatible`
- Runtime model loader: `plugin/github-copilot/models.ts` sets `model.api.npm = "@ai-sdk/github-copilot"`
- Docs URL: `https://docs.github.com/en/copilot`

Important caveat: behavior should follow the runtime model metadata loaded by the Copilot plugin, not the snapshot provider `npm` field.

### Source Control

- Behavior source:
  - `plugin/github-copilot/models.ts`
  - `plugin/github-copilot/copilot.ts`
  - `provider/transform.ts`
  - Copilot SDK adapter code under `provider/sdk/copilot/`
- Availability source:
  - live Copilot runtime metadata fetched by the plugin from the provider `/models` endpoint
  - the model picker/runtime provider list
- Stale or fallback source:
  - bundled `models-snapshot.js` entries are descriptive fallback data only
  - if snapshot and plugin/runtime disagree, trust plugin/runtime

### User-Facing Knobs

- Agent: `model`, `variant`, `temperature`, `top_p`, `options`, `steps`
- Provider config: `enterpriseUrl`, `apiKey`, `baseURL`, `timeout`, `chunkTimeout`
- Provider-options namespace: `copilot`

Local Copilot option schema in `provider/sdk/copilot/chat/openai-compatible-chat-options.ts` explicitly includes:
- `user`
- `reasoningEffort`
- `textVerbosity`
- `thinking_budget`

### How Copilot Model Capabilities Are Derived

`plugin/github-copilot/models.ts` infers `capabilities.reasoning` from remote Copilot model metadata when any of these are present:
- `adaptive_thinking`
- `reasoning_effort`
- `max_thinking_budget`
- `min_thinking_budget`

That same loader maps Copilot limits into OpenCode model limits.

### Variants

`ProviderTransform.variants()` applies these rules for `@ai-sdk/github-copilot`:

| Model family | Variants | Payload |
|---|---|---|
| Gemini models | none | `{}` |
| Claude models | `low`, `medium`, `high` | `{ reasoningEffort: <effort> }` |
| GPT / Codex models | `low`, `medium`, `high`, sometimes `xhigh` | `{ reasoningEffort: <effort>, reasoningSummary: "auto", include: ["reasoning.encrypted_content"] }` |

`xhigh` is added for:
- `gpt-5.1-codex-max`
- `gpt-5.2*`
- `gpt-5.3*`
- some `gpt-5` releases on or after `2025-12-04`

Concrete examples:
- `gpt-5.3-codex`
- `gpt-5.1`
- `claude-sonnet-4.6`
- `gemini-3-flash-preview`

### OpenCode Defaults And Transforms

- `ProviderTransform.options()` sets `store: false` for Copilot models.
- `plugin/github-copilot/copilot.ts` removes `maxOutputTokens` for GPT models to match Copilot CLI behavior.
- `plugin/github-copilot/copilot.ts` adds `anthropic-beta: interleaved-thinking-2025-05-14` for Anthropic-backed Copilot requests.
- `ProviderTransform.providerOptions()` maps agent `options` into `providerOptions.copilot`.

### Caveats

- In the Copilot OpenAI-responses path, reasoning models strip unsupported `temperature` and `topP` values.
- In that same path, non-reasoning models reject `reasoningEffort` and `reasoningSummary`.
- Many Copilot GPT-5 and Codex entries expose `temperature: false`, so user-set `temperature` will often not be sent at all.

## opencode-go

### Identity

- Provider ID: `opencode-go`
- Snapshot provider metadata:
  - `env = ["OPENCODE_API_KEY"]`
  - `npm = "@ai-sdk/openai-compatible"`
  - `api = "https://opencode.ai/zen/go/v1"`
  - `doc = "https://opencode.ai/docs/zen"`

Current known examples:
- `glm-5`
- `glm-5.1`
- `minimax-m2.7`
- `kimi-k2.5`
- `minimax-m2.5`

Availability note: `glm-5.1` is treated as valid even though it was not present in the bundled local snapshot, because live picker/runtime evidence is stronger than snapshot absence.

### Source Control

- Behavior source:
  - `provider/transform.ts`
  - `provider/provider.ts`
  - `session/llm.ts`
  - model-level runtime metadata after provider/model override resolution
- Availability source:
  - the live model picker/runtime provider list
  - live provider metadata exposed through OpenCode runtime resolution
- Stale or fallback source:
  - bundled `models-snapshot.js`
  - cached catalog data
  - if live picker/runtime shows a model and the snapshot does not, trust live picker/runtime

### User-Facing Knobs

- Agent: `model`, `variant`, `temperature`, `top_p`, `options`, `steps`
- Provider config: `apiKey`, `baseURL`, `setCacheKey`, `timeout`, `chunkTimeout`

Provider-options namespace depends on the model's effective SDK:
- ordinary openai-compatible models fall back to `providerOptions["opencode-go"]`
- models with `model.provider.npm = "@ai-sdk/anthropic"` use `providerOptions.anthropic`

Why this matters: `provider/provider.ts` builds `model.api.npm` from `model.provider?.npm ?? provider.npm`, so model-level overrides win.

### Variants

At first glance, openai-compatible models look like they should get `low` / `medium` / `high` reasoning variants.

The catch is the early family filter in `ProviderTransform.variants()`. Any reasoning model whose ID includes one of these families gets no synthesized variants:
- `deepseek`
- `minimax`
- `glm`
- `mistral`
- `kimi`
- `k2p5`
- `qwen`
- `big-pickle`

That means the current known `opencode-go` `glm`, `minimax`, and `kimi` models synthesize no variants even when `reasoning: true`.

### OpenCode Defaults And Transforms

Family-specific defaults from `ProviderTransform` matter more here than variants.

| Model ID pattern | `temperature()` | `topP()` | `topK()` |
|---|---|---|---|
| `minimax-m2*` | `1.0` | `0.95` | `40` for `m2.` / `m25` / `m21`, else `20` |
| `kimi-k2.5` / `kimi-k2p5` / `kimi-k2-5` | `1.0` | `0.95` | none |
| `glm-4.6` / `glm-4.7` | `1.0` | none | none |
| `glm-5` / `glm-5.1` | none | none | none |

Additional notes:
- `maxOutputTokens` is still capped by `ProviderTransform.maxOutputTokens()`.
- For any future `opencode-*` provider model whose API ID includes `gpt-5`, `ProviderTransform.options()` adds `promptCacheKey`, `include: ["reasoning.encrypted_content"]`, and `reasoningSummary: "auto"`.

### Caveats

- Do not assume one uniform SDK across the whole provider.
- Current known models under `opencode-go` are mostly reasoning models, but the family filter means that reasoning often does not translate into OpenCode variants.
- The bundled local snapshot may lag behind the live provider catalog exposed in the model picker.
- Do not infer invalidity from snapshot absence alone.
- The right provider-options namespace can change model by model because it follows the effective SDK.

## openai

### Identity

- Provider ID: `openai`
- Provider `npm`: `@ai-sdk/openai`
- Docs URL: `https://platform.openai.com/docs/models`

Relevant examples:
- `gpt-5.4-mini`
- `gpt-5.4`
- `gpt-5-pro`
- `gpt-5.3-codex`
- `gpt-4o-2024-11-20`

### Source Control

- Behavior source:
  - `provider/transform.ts`
  - `session/llm.ts`
  - `provider/provider.ts`
  - OpenAI SDK request assembly paths used by OpenCode
- Availability source:
  - the runtime provider model list
  - live catalog data when refreshed
- Stale or fallback source:
  - bundled `models-snapshot.js`
  - if runtime/live catalog and snapshot differ, trust runtime/live catalog

### User-Facing Knobs

- Agent: `model`, `variant`, `temperature`, `top_p`, `options`, `steps`
- Provider config: `apiKey`, `baseURL`, `setCacheKey`, `timeout`, `chunkTimeout`
- Provider-options namespace: `openai`

### Variants

`ProviderTransform.variants()` applies these rules for `@ai-sdk/openai` reasoning models:

| Model family | Variants | Notes |
|---|---|---|
| `gpt-5-pro` | none | Explicit special case |
| Codex models | usually `low`, `medium`, `high` | `gpt-5.2-codex` and `gpt-5.3-codex` also get `xhigh` |
| GPT-5 family | starts from `low`, `medium`, `high` | `minimal` is added for `gpt-5` / `gpt-5-*`; `none` is added for release dates `>= 2025-11-13`; `xhigh` is added for release dates `>= 2025-12-04` |

The synthesized payload for OpenAI reasoning variants is:

```json
{
  "reasoningEffort": "high",
  "reasoningSummary": "auto",
  "include": ["reasoning.encrypted_content"]
}
```

### OpenCode Defaults And Transforms

- `ProviderTransform.options()` sets `store: false` for OpenAI models.
- `ProviderTransform.options()` sets `promptCacheKey = sessionID` for the OpenAI provider.
- For `gpt-5` models except `gpt-5-pro`, OpenCode also defaults:

```json
{
  "reasoningEffort": "medium",
  "reasoningSummary": "auto"
}
```

- For non-chat `gpt-5.x` models that are not Codex and not Azure-backed, OpenCode also sets:

```json
{
  "textVerbosity": "low"
}
```

- `maxOutputTokens` is capped by `ProviderTransform.maxOutputTokens()`.
- `ProviderTransform.providerOptions()` maps agent `options` into `providerOptions.openai`.

### Caveats

- Many OpenAI reasoning models expose `temperature: false`, so user `temperature` is often ignored before request assembly.
- Release date changes available variant names. Do not assume all GPT-5 models expose the same ladder.
- `provider/provider.ts` removes `gpt-5-chat-latest` from the provider model list.

## Quick Rules Of Thumb

- If you set `temperature`, first check whether that model's capability allows temperature.
- If you set `top_p` in config, remember the runtime field is `topP`.
- If a model is reasoning-capable, variants may still be empty because OpenCode suppresses variants for some model families.
- For openai-compatible providers, provider-options namespacing usually falls back to the provider ID unless OpenCode has a specific SDK key mapping.
- For Copilot, trust the plugin-loaded runtime model metadata more than the bundled snapshot.
- For `opencode-go`, trust the live picker/runtime provider list over snapshot absence.
