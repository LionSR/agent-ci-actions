# agent-ci-actions

Run **Claude Code in CI across providers** — Anthropic, **DeepSeek**, **Kimi (Moonshot)**,
or any Anthropic-compatible endpoint — and wire up **self-healing auto-fix** loops.
Domain-agnostic: no project-specific tool allowlists are baked in; you supply them.

De-duplicated from several private repositories into one maintained source of truth.

## Why

- **Provider choice.** One token swap runs the same agent on Anthropic, DeepSeek, or
  Kimi — pick by cost, context window, or availability without rewriting workflows.
- **Auto-fix.** Turn a red CI run into an agent that reads the failure, pushes a fix,
  and lets CI re-run — bounded by an iteration cap so it never loops forever.
- **No lock-in to one project.** Tool allowlists ("presets") are caller-supplied JSON,
  so the same actions serve a Lean repo, a TypeScript repo, or anything else.

## Actions

| Action | Reference | What it does |
| --- | --- | --- |
| **Claude Code Multi-Provider Runner** | `LionSR/agent-ci-actions@v1` | Run Claude Code against Anthropic / DeepSeek / Kimi / any compatible provider; resolve model by tier; apply a caller-supplied tool preset. |
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
# DeepSeek (built-in convenience: default base URL + model)
- uses: LionSR/agent-ci-actions@v1
  with:
    provider: deepseek
    deepseek-api-key: ${{ secrets.DEEPSEEK_API_KEY }}
    prompt: 'Fix the failing build.'
```

```yaml
# Kimi / Moonshot (Anthropic-compatible endpoint; see
# https://platform.kimi.ai/docs/guide/claude-code-kimi)
- uses: LionSR/agent-ci-actions@v1
  with:
    provider: kimi              # moonshot is accepted as an alias
    kimi-api-key: ${{ secrets.KIMI_API_KEY }}
    # defaults: base https://api.moonshot.ai/anthropic, model kimi-k3[1m]
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
`MOONSHOT_API_KEY` / `ANTHROPIC_AUTH_TOKEN`, `CLAUDE_OPUS_MODEL`, `KIMI_OPUS_MODEL`, …).

For `provider: kimi` the action also sets the Claude Code env vars Kimi requires
(`ANTHROPIC_AUTH_TOKEN`, all `ANTHROPIC_DEFAULT_*_MODEL` tiers,
`CLAUDE_CODE_SUBAGENT_MODEL`, `CLAUDE_CODE_AUTO_COMPACT_WINDOW`,
`CLAUDE_CODE_EFFORT_LEVEL=max`) so background and sub-agent calls do not fall back to
Anthropic model names.

### Kimi models (Claude Code)

IDs for the Anthropic-compatible endpoint — see [Model List](https://platform.kimi.ai/docs/models)
and [Use Kimi in Claude Code](https://platform.kimi.ai/docs/guide/claude-code-kimi):

| Model | Context | Notes |
| --- | --- | --- |
| `kimi-k3[1m]` | 1M | **Default.** Flagship; Claude Code spelling of `kimi-k3`. |
| `kimi-k2.7-code` | 256K | Dedicated coding model; thinking always on. |
| `kimi-k2.7-code-highspeed` | 256K | Same as `kimi-k2.7-code`, ~5–6× faster output. |
| `kimi-k2.6` | 256K | Thinking optional; good for latency-sensitive tasks. |

Compact window is set automatically: `1048576` for `kimi-k3*`, `262144` for `k2.7` / `k2.6`.
Override with `kimi-opus-model` / `kimi-sonnet-model` (or `KIMI_OPUS_MODEL` / `KIMI_SONNET_MODEL`).

**Deprecated — do not use:** `kimi-k2-0905-preview`, other `kimi-k2-*-preview` /
`kimi-k2-thinking*` IDs (sunset May 25, 2026). `kimi-k2.5` and `moonshot-v1*` are
being removed for new accounts.

```yaml
# Cheaper/faster coding CI on the sonnet tier
- uses: LionSR/agent-ci-actions@v1
  with:
    provider: kimi
    kimi-api-key: ${{ secrets.KIMI_API_KEY }}
    model-tier: sonnet
    kimi-sonnet-model: kimi-k2.7-code
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
    D --> E[Claude Code Multi-Provider Runner:<br/>Anthropic / DeepSeek / Kimi]
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
