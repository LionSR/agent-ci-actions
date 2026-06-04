# agent-ci-actions

Reusable GitHub Actions for running **Claude Code** across providers and for
building **CI auto-fix** automation. Domain-agnostic: no project-specific tool
allowlists are baked in — you supply them.

Extracted and de-duplicated from several private repositories so there is one
maintained source of truth.

| Action | Reference | What it does |
| --- | --- | --- |
| **Claude Code Multi-Provider Runner** | `LionSR/agent-ci-actions@v1` | Run Claude Code against Anthropic / DeepSeek / any Anthropic-compatible provider; resolve model by tier; apply a caller-supplied tool preset. |
| **Compose auto-fix prompt** | `LionSR/agent-ci-actions/compose-auto-fix-prompt@v1` | Load a prompt, append CI-failure + PR context, expose as one output. |
| **Auto-create PR for issue work** | `LionSR/agent-ci-actions/auto-create-issue-pr@v1` | Open a PR from a bot-pushed issue branch; optionally add an auto-fix label. |
| **Fetch failure logs** | `LionSR/agent-ci-actions/fetch-failure-logs@v1` | Download + sanitize failed-job logs from a workflow run. |
| **Fetch review comments** | `LionSR/agent-ci-actions/fetch-review-comments@v1` | Fetch unresolved PR review threads via GraphQL. |
| **Attach PR branch** | `LionSR/agent-ci-actions/attach-pr-branch@v1` | Resolve and check out a PR head branch. |
| **Bot-fix guard** | `LionSR/agent-ci-actions/bot-fix-guard@v1` | Count prior bot-fix commits to cap auto-fix loops. |

Each action is self-contained — use the one you need, or compose them.

## Claude Code Multi-Provider Runner

Wraps [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action):
picks the provider, resolves the model for the requested `model-tier`, and applies
`--allowedTools`.

```yaml
- uses: LionSR/agent-ci-actions@v1
  with:
    provider: anthropic           # or "deepseek"
    model-tier: opus              # or "sonnet"
    claude-oauth-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    prompt: 'Fix the failing build.'
```

### Tool presets are caller-supplied

The action ships **no** tool allowlists. Pass a JSON map and reference its keys —
keep the map in your own repo (a `vars.*` value or a checked-in `.github/allowed-tools.json`):

```yaml
- uses: LionSR/agent-ci-actions@v1
  with:
    allowed_tools_preset: my-fix
    allowed_tools_preset_map: |
      {
        "my-fix": "Edit,Write,Read,Glob,Grep,Bash(make *),Bash(git add *),Bash(git commit *),mcp__github__*",
        "review": "Read,Glob,Grep,Bash(gh pr diff *),mcp__github__*"
      }
```

Resolution precedence: explicit `allowed_tools` → `allowed_tools_preset_map[allowed_tools_preset]`
→ `CLAUDE_ALLOWED_TOOLS` env → empty. An unknown preset against a non-empty map is a hard error.

Key inputs (all optional unless noted; model/token inputs also read the matching env var):

| Input | Default | Notes |
| --- | --- | --- |
| `provider` | `anthropic` | `anthropic` or `deepseek`. |
| `model-tier` | `opus` | `opus` or `sonnet`. |
| `claude-opus-model` / `claude-sonnet-model` | `claude-opus-4-8` / `claude-sonnet-4-6` | Override per shop; env `CLAUDE_OPUS_MODEL` etc. also honored. |
| `allowed_tools` | `''` | Explicit `--allowedTools`; wins over a preset. |
| `allowed_tools_preset` / `allowed_tools_preset_map` | `''` | Preset key + the caller's JSON map. |
| `prompt` / `claude-prompt-file` | `''` | Prompt text or a file path. |
| `system-prompt` / `system-prompt-file` | `''` | Appended via `--append-system-prompt`. |
| `plugins` / `plugin_marketplaces` | `''` | Forwarded to claude-code-action. |

## Versioning

Pin `@v1` to track `v1.x`, or a tag / full SHA for strict reproducibility.

## License

[MIT](LICENSE).
