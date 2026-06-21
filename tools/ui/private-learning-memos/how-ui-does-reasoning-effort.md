# How the UI handles reasoning effort

Summary of how `tools/ui` maps the reasoning effort setting to llama-server request parameters (verified against current master, including PR #23434).

## TL;DR

`reasoningEffort` (`'low' | 'medium' | 'high' | 'max'`) is **not** sent to llama-server as a string field (`reasoningEffort` or `reasoning_effort`). The UI maps it to `thinking_budget_tokens` and sends a few related fields alongside the thinking toggle.

## Flow

1. UI store: `conversationsStore.getReasoningEffort()` returns `'low' | 'medium' | 'high' | 'max'`
2. Chat store: `getApiOptions()` passes it as internal option `reasoningEffort`
3. Chat service: `ChatService.sendMessage()` transforms it into the POST body for `/v1/chat/completions`

Relevant code: `tools/ui/src/lib/services/chat.service.ts` (around lines 248-261).

## Exact server parameters

### `thinking_budget_tokens` (derived from effort)

Only set when **thinking is ON** and the mapped value is `>= 0`:

| UI effort | `thinking_budget_tokens` |
|-----------|--------------------------|
| `low`     | `512`                    |
| `medium`  | `2048`                   |
| `high`    | `8192`                   |
| `max`     | **omitted**              |

Mapping source: `tools/ui/src/lib/constants/reasoning-effort-tokens.ts`

- `max` maps to `-1` (unlimited) and is skipped by the `if (reasoningBudgetTokens >= 0)` check
- If thinking is **off**, effort is ignored and `thinking_budget_tokens` is never set

### Always sent (reasoning-related)

On every chat completion request:

- **`chat_template_kwargs.enable_thinking`**: `true` or `false` from the thinking toggle (`getThinkingEnabled()`), not from effort level
- **`reasoning_control`**: always `true` (enables runtime stop via `/v1/chat/completions/control`)
- **`reasoning_format`**: `"auto"` (default) or `"none"` if `disableReasoningParsing` is on

### What is NOT sent

- No `reasoningEffort` field on the request
- No `reasoning_effort` in `chat_template_kwargs` (llama-server supports this for some templates via presets, but the UI does not use it)

Note: the enum comment in `reasoning-effort.enums.ts` says values are "sent to the server" - that is misleading. Only the token budget is sent, not the string labels.

## Example payloads

**Medium effort, thinking ON:**

```json
{
  "chat_template_kwargs": { "enable_thinking": true },
  "thinking_budget_tokens": 2048,
  "reasoning_control": true,
  "reasoning_format": "auto"
}