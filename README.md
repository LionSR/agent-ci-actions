# agent-ci-actions

Run **Claude Code in CI across providers** — Anthropic, **DeepSeek**, **Kimi**,
**Z.AI (GLM)**, or any Anthropic-compatible endpoint — and wire up **self-healing
auto-fix** loops. Works with both pay-per-token API keys and coding-plan
subscriptions. Domain-agnostic: no project-specific tool allowlists are baked in; you
supply them.

De-duplicated from several private repositories into one maintained source of truth.

## Why

- **Provider choice.** One token swap runs the same agent on Anthropic, DeepSeek, Kimi,
  or Z.AI — pick by cost, context window, or availability without rewriting workflows,
  and pay by token or against a coding-plan subscription.
- **Auto-fix.** Turn a red CI run into an agent that reads the failure, pushes a fix,
  and lets CI re-run — bounded by an iteration cap so it never loops forever.
- **No lock-in to one project.** Tool allowlists ("presets") are caller-supplied JSON,
  so the same actions serve a Lean repo, a TypeScript repo, or anything else.

## Actions

| Action | Reference | What it does |
| --- | --- | --- |
| **Claude Code Multi-Provider Runner** | `LionSR/agent-ci-actions@v1` | Run Claude Code against Anthropic / DeepSeek / Kimi / Z.AI / any compatible provider; resolve model by tier; apply a caller-supplied tool preset. |
| **Compose auto-fix prompt** | `LionSR/agent-ci-actions/compose-auto-fix-prompt@v1` | Load a prompt, append CI-failure + PR context, expose as one output. |
| **Auto-create PR for issue work** | `LionSR/agent-ci-actions/auto-create-issue-pr@v1` | Open a PR from a bot-pushed issue branch; optionally add an auto-fix label. |
| **Fetch failure logs** | `LionSR/agent-ci-actions/fetch-failure-logs@v1` | Download + sanitize failed-job logs from a workflow run. |
| **Fetch review comments** | `LionSR/agent-ci-actions/fetch-review-comments@v1` | Fetch unresolved PR review threads via GraphQL. |
| **Attach PR branch** | `LionSR/agent-ci-actions/attach-pr-branch@v1` | Resolve and check out a PR head branch. |
| **Bot-fix guard** | `LionSR/agent-ci-actions/bot-fix-guard@v1` | Count prior bot-fix commits to cap auto-fix loops. |

## Providers

The runner wraps [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action)
and talks to any **Anthropic-compatible** API.

```yaml
# Anthropic (default)
- uses: LionSR/agent-ci-actions@v1
  with:
    provider: anthropic
    claude-oauth-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    model-tier: opus            # -> claude-opus-model (default claude-opus-4-8)
    prompt: 'Fix the failing build.'
```

```yaml
# DeepSeek (see https://api-docs.deepseek.com/quick_start/agent_integrations/claude_code/)
- uses: LionSR/agent-ci-actions@v1
  with:
    provider: deepseek
    deepseek-api-key: ${{ secrets.DEEPSEEK_API_KEY }}
    # defaults: base https://api.deepseek.com/anthropic,
    # model deepseek-v4-pro[1m]; haiku/subagent → deepseek-v4-flash
    prompt: 'Fix the failing build.'
```

```yaml
# Kimi — pay-per-token (platform.kimi.ai) or Coding Plan (kimi.com/code)
- uses: LionSR/agent-ci-actions@v1
  with:
    provider: kimi              # moonshot alias; use kimi-code for the Coding Plan
    kimi-api-key: ${{ secrets.KIMI_API_KEY }}
    # platform default: https://api.moonshot.ai/anthropic, model kimi-k3[1m]
    prompt: 'Fix the failing build.'

- uses: LionSR/agent-ci-actions@v1
  with:
    provider: kimi-code         # or provider: kimi + kimi-base-url: https://api.kimi.com/coding/
    kimi-api-key: ${{ secrets.KIMI_CODE_API_KEY }}   # kimi-code-api-key also accepted
    # coding default: https://api.kimi.com/coding/, model k3[1m]
    prompt: 'Fix the failing build.'
```

```yaml
# Z.AI / GLM Coding Plan (see https://docs.z.ai/devpack/tool/claude and
# https://docs.z.ai/devpack/latest-model)
- uses: LionSR/agent-ci-actions@v1
  with:
    provider: zai               # z.ai and glm are accepted as aliases
    zai-api-key: ${{ secrets.ZAI_API_KEY }}
    # defaults: base https://api.z.ai/api/anthropic,
    # opus/sonnet glm-5.2[1m], haiku/subagent glm-4.7
    prompt: 'Fix the failing build.'
```

```yaml
# Any other Anthropic-compatible endpoint via base-url override
- uses: LionSR/agent-ci-actions@v1
  with:
    provider: anthropic
    anthropic-base-url: https://example.compat/anthropic
    anthropic-api-key: ${{ secrets.COMPAT_API_KEY }}
    claude-opus-model: my-compat-model
    prompt: 'Fix the failing build.'
```

`model-tier` (`opus` / `sonnet`) picks between the opus- and sonnet-tier model inputs,
so one workflow can dial cost/quality per call. Tokens and model names also fall back to
the matching env vars (`CLAUDE_CODE_OAUTH_TOKEN`, `DEEPSEEK_API_KEY`, `KIMI_API_KEY` /
`MOONSHOT_API_KEY` / `ANTHROPIC_AUTH_TOKEN`, `KIMI_CODE_API_KEY`, `ZAI_API_KEY` /
`GLM_API_KEY`, `CLAUDE_OPUS_MODEL`, `KIMI_OPUS_MODEL`, `ZAI_OPUS_MODEL`, …).

For every non-Anthropic provider the action fills all `ANTHROPIC_DEFAULT_*_MODEL` tiers
plus `CLAUDE_CODE_SUBAGENT_MODEL`, `CLAUDE_CODE_AUTO_COMPACT_WINDOW`, and
`CLAUDE_CODE_EFFORT_LEVEL`, so background and sub-agent calls never fall back to
Anthropic model names.

### API keys vs. coding plans

| Provider | Key | Endpoint | Auth |
| --- | --- | --- | --- |
| `kimi` | platform.kimi.ai (pay-per-token) | `https://api.moonshot.ai/anthropic` | `ANTHROPIC_AUTH_TOKEN` |
| `kimi-code` | Kimi Code Console (subscription) | `https://api.kimi.com/coding/` | `ANTHROPIC_API_KEY` |
| `zai` | Open Platform **or** GLM Coding Plan | `https://api.z.ai/api/anthropic` | `ANTHROPIC_AUTH_TOKEN` |
| `deepseek` | platform key (pay-per-token only) | `https://api.deepseek.com/anthropic` | `ANTHROPIC_AUTH_TOKEN` |

`kimi` and `kimi-code` share one code path — `kimi-code` is just the Coding Plan
alias (also selected automatically when `kimi-base-url` points at `api.kimi.com`).
Moonshot's two platforms reject each other's keys, so match the key to the plan.
Z.AI needs no split: both key types share one endpoint, and the defaults are already
Coding Plan–safe (GLM-5.2 / GLM-5-Turbo / GLM-4.7 only).

### DeepSeek models (Claude Code)

IDs from the [DeepSeek Claude Code guide](https://api-docs.deepseek.com/quick_start/agent_integrations/claude_code/)
and [Models & Pricing](https://api-docs.deepseek.com/quick_start/pricing) (both have 1M context):

| Role | Model | Notes |
| --- | --- | --- |
| Main / opus / sonnet default | `deepseek-v4-pro[1m]` | **Default** for `--model` and opus/sonnet tiers |
| Haiku / sub-agent | `deepseek-v4-flash` | Cheaper; set via `deepseek-flash-model` |

Claude Desktop/Code also map `claude-opus*` → pro and `claude-sonnet*` / `claude-haiku*` → flash when you only change base URL + key.

```yaml
# Cheaper CI: run the sonnet tier on flash
- uses: LionSR/agent-ci-actions@v1
  with:
    provider: deepseek
    deepseek-api-key: ${{ secrets.DEEPSEEK_API_KEY }}
    model-tier: sonnet
    deepseek-sonnet-model: deepseek-v4-flash
    prompt: 'Fix the failing build.'
```

### Kimi models (Claude Code)

One provider, two plans — override with `kimi-opus-model` / `kimi-sonnet-model`
(or `KIMI_OPUS_MODEL` / `KIMI_SONNET_MODEL`).

**Platform** ([docs](https://platform.kimi.ai/docs/guide/claude-code-kimi)):

| Model | Context | Notes |
| --- | --- | --- |
| `kimi-k3[1m]` | 1M | **Default** for `provider: kimi`. |
| `kimi-k2.7-code` / `kimi-k2.7-code-highspeed` | 256K | Coding-focused; highspeed ~5–6× faster. |
| `kimi-k2.6` | 256K | Thinking optional. |

**Coding Plan** ([docs](https://www.kimi.com/code/docs/en/third-party-tools/claude-code)):

| Model | Context | Notes |
| --- | --- | --- |
| `k3[1m]` | 1M | **Default** for `provider: kimi-code`. |
| `k3-256k` | 256K | Same model, smaller window. |
| `kimi-for-coding` / `kimi-for-coding-highspeed` | 256K | Version-stable IDs; highspeed needs Allegretto+. |

Compact window: `1048576` for `[1m]` IDs, `262144` for 256K / `k2.x` / `kimi-for-coding*`.
Coding Plan also sets `CLAUDE_CODE_MAX_CONTEXT_TOKENS` to match and
`CLAUDE_CODE_EFFORT_LEVEL=high`.

**Deprecated on the platform — do not use:** `kimi-k2-*-preview` / `kimi-k2-thinking*`
(sunset May 25, 2026). `kimi-k2.5` and `moonshot-v1*` are being removed for new accounts.

```yaml
# Platform, cheaper sonnet tier
- uses: LionSR/agent-ci-actions@v1
  with:
    provider: kimi
    kimi-api-key: ${{ secrets.KIMI_API_KEY }}
    model-tier: sonnet
    kimi-sonnet-model: kimi-k2.7-code
    prompt: 'Fix the failing build.'

# Coding Plan, version-stable highspeed
- uses: LionSR/agent-ci-actions@v1
  with:
    provider: kimi-code
    kimi-api-key: ${{ secrets.KIMI_CODE_API_KEY }}
    kimi-opus-model: kimi-for-coding-highspeed
    prompt: 'Fix the failing build.'
```

### Z.AI / GLM models (Claude Code)

IDs from the [Z.AI Claude Code guide](https://docs.z.ai/devpack/tool/claude) and
[How to Switch Models](https://docs.z.ai/devpack/latest-model). GLM Coding Plan
calls are limited to **GLM-5.2**, **GLM-5-Turbo**, and **GLM-4.7**
([FAQ](https://docs.z.ai/devpack/faq)):

| Role | Model | Notes |
| --- | --- | --- |
| Main / opus / sonnet default | `glm-5.2[1m]` | **Default.** Latest flagship with 1M context (`[1m]` suffix). |
| Haiku / sub-agent | `glm-4.7` | **Default** for background tiers (Coding Plan–safe). |
| Faster / cheaper main | `glm-5-turbo` or `glm-4.7` | Set via `zai-opus-model` / `zai-sonnet-model`. |

Also sets `API_TIMEOUT_MS=3000000`, `CLAUDE_CODE_AUTO_COMPACT_WINDOW=1000000`
(Z.AI’s documented value), and `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`.

```yaml
# Pin everything to glm-4.7
- uses: LionSR/agent-ci-actions@v1
  with:
    provider: zai
    zai-api-key: ${{ secrets.ZAI_API_KEY }}
    zai-opus-model: glm-4.7
    zai-sonnet-model: glm-4.7
    prompt: 'Fix the failing build.'
```

## Tool presets are caller-supplied

The action ships **no** tool allowlists. Keep your presets in a checked-in JSON file and
point the action at it:

```jsonc
// .github/allowed-tools.json
{
  "auto-fix":  "Edit,Write,Read,Glob,Grep,Bash(make *),Bash(git add *),Bash(git commit *),mcp__github__*",
  "review":    "Read,Glob,Grep,Bash(gh pr diff *),mcp__github__*"
}
```

```yaml
- uses: LionSR/agent-ci-actions@v1
  with:
    allowed_tools_preset: auto-fix
    allowed_tools_preset_map_file: .github/allowed-tools.json
```

Resolution precedence: explicit `allowed_tools` → preset looked up in
`allowed_tools_preset_map` (inline JSON) or `allowed_tools_preset_map_file` →
`CLAUDE_ALLOWED_TOOLS` env → empty. An unknown preset against a non-empty map is a hard error.

## Auto-fix loop

The supporting actions compose a bounded self-healing loop: a failed CI run drives an
agent that reads the failure, pushes a fix, and lets CI re-run — capped by `bot-fix-guard`.

```mermaid
flowchart TD
    A[CI fails on a PR] --> B{bot-fix-guard:<br/>under the iteration cap?}
    B -- no --> Z[Stop — hand back to a human]
    B -- yes --> C[fetch-failure-logs:<br/>collect failed-job output]
    C --> D[compose-auto-fix-prompt:<br/>prompt + PR + failure context]
    D --> E[Claude Code Multi-Provider Runner:<br/>Anthropic / DeepSeek / Kimi / Z.AI]
    E --> F[Agent edits and pushes a fix commit]
    F --> G[CI re-runs]
    G -- still red --> A
    G -- green --> H([Done])
```

`bot-fix-guard` counts prior bot-fix commits so the loop is bounded; `fetch-failure-logs`
sanitizes the logs (treat them as untrusted data); `compose-auto-fix-prompt` folds the
failure context into the prompt; the runner executes the fix on your chosen provider.

## Versioning

Pin `@v1` to track `v1.x`, or a specific tag for strict reproducibility.

## License

[MIT](LICENSE).
