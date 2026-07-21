# Claude Code model and agent policy

Use this file as a starting point for a project-level `CLAUDE.md` or merge the relevant sections into `~/.claude/CLAUDE.md`. Review it before use and adapt agent names to the agents installed in your Claude Code environment.

## Model roles

Use three model families, with high reasoning as the default for substantive work:

| Model | Role |
| --- | --- |
| `gpt-5.6-sol` | Execution manager: task breakdown, implementation, integration, debugging, verification, and final reporting. |
| `gpt-5.6-sol-medium` | Moderately scoped or mostly mechanical implementation. |
| `gpt-5.6-sol-low` | Trivial file access, repetitive edits, glue, and mechanical checks. |
| Claude Fable 5 | Architecture, planning, contracts, design judgment, and senior-engineer review. |
| Claude Sonnet 5 | Read-only exploration, code search, flow tracing, and API inventory. |

Do not use Opus 4.8 or Haiku for this workflow. Do not substitute a GPT model behind the Fable or Sonnet aliases.

## Staffing policy

1. **Fable designs substantive work.** Ask Fable for a concrete, self-contained execution brief covering architecture, contracts, file boundaries, risks, and verification.
2. **The current `gpt-5.6-sol` session manages execution.** It owns the task graph, integrates worker changes, runs combined verification, and reports the final result.
3. **Sol workers implement bounded scopes.** Give every mutating worker explicit file ownership, acceptance criteria, and required checks.
4. **Fable reviews the result.** Use Fable for senior-engineer review of the actual diff and verification evidence, especially for user-facing behavior.
5. **Sonnet only explores.** Sonnet may locate code and trace behavior, but it does not design, implement, or review changes.

Keep Fable on high-leverage design and review decisions. Keep Sol on execution. Do not let implementation workers silently change an approved architecture.

## Suggested subagent routing

Use the closest available agent type in the current Claude Code installation:

| Work | Suggested invocation |
| --- | --- |
| Context-heavy Sol work that needs the parent conversation | `subagent_type: "fork"` |
| Explicit high-reasoning Sol worker | `subagent_type: "sol-high-worker"` |
| Explicit medium-reasoning Sol worker | `subagent_type: "sol-medium-worker"` |
| Explicit low-reasoning Sol worker | `subagent_type: "sol-low-worker"` |
| Fable architecture, design, or review | `subagent_type: "claude", model: "fable"` |
| Sonnet read-only exploration | `subagent_type: "Explore", model: "sonnet"` |

Named Sol workers require corresponding custom agent definitions in the user's Claude Code agents directory. If they are not installed, use the current Sol session or a context-inheriting fork rather than silently substituting another model family.

A fork inherits the parent conversation and model. When the parent is `gpt-5.6-sol`, the fork is also a Sol worker and is appropriate when full conversation context is required.

## Execution loop

For each substantive work package:

1. Use Fable to produce or refine the execution brief.
2. Let the main Sol session create and own the task breakdown.
3. Invoke bounded Sol workers only where work can be safely separated.
4. Integrate all changes in the main session.
5. Run the combined tests, builds, linters, and product checks.
6. Use Fable to review the actual diff and verification evidence.
7. Apply concrete findings and repeat verification until clean.

Anything user-facing—UI, copy, APIs, permissions, protocols, or failure behavior—receives a Fable review before shipping.

When parallelizing:

- launch only genuinely independent work concurrently;
- avoid parallel mutation of the same files or shared contracts;
- state file ownership and acceptance criteria in every worker prompt; and
- treat worker completion as input to integration, not completion of the overall task.

## Session management

Do not launch a second headless Claude Code session merely to obtain another execution manager. The current session already has the conversation, tools, and orchestration context.

Use a separate session only for a concrete lifecycle requirement that a bounded subagent cannot satisfy, such as:

- independently resumable long-running work;
- strict context isolation for an unrelated project; or
- a process that must outlive the current conversation.

Explain the reason before creating the extra session.

## CLIProxyAPI routing

The normal Claude Code client configuration is:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:8317",
    "ANTHROPIC_AUTH_TOKEN": "YOUR_LOCAL_CLIPROXY_API_KEY",
    "ANTHROPIC_MODEL": "gpt-5.6-sol",
    "ANTHROPIC_SMALL_FAST_MODEL": "gpt-5.6-sol",
    "CLAUDE_CODE_AUTO_COMPACT_WINDOW": "272000",
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "90"
  },
  "model": "gpt-5.6-sol",
  "fastMode": false,
  "autoCompactEnabled": true
}
```

Never place a real API key in this file or commit one to the repository.

Reasoning effort is selected by model ID and enforced by CLIProxyAPI:

- `gpt-5.6-sol` → high
- `gpt-5.6-sol-medium` → medium
- `gpt-5.6-sol-low` → low

The GPT routes use the Codex protocol and `service_tier: priority`. Do not enable Claude Code `/fast` for these custom GPT model IDs; it can switch the session to an Anthropic model instead of accelerating GPT.

## How model and account selection work

```text
model ID selected by Claude Code or a subagent
  → provider/protocol selection
  → enabled OAuth credential for the selected provider
  → provider-specific model alias resolution
  → ordered payload overrides
  → reasoning effort and service tier
  → upstream request
```

Model selection is per request. The chosen model ID determines whether the request follows the Codex or Claude provider route and which proxy-side request behavior is applied.

Account selection is global per proxy instance. Use `cliproxy-select` to keep exactly one Claude credential and one Codex credential enabled:

```bash
cliproxy-select list
cliproxy-select set \
  --claude claude-user@example.com \
  --codex codex-user@example.com
cliproxy-select status
```

Changing the GPT reasoning alias does not change the enabled Codex account. Selecting Fable or Sonnet uses the enabled Claude account. Re-run `cliproxy-select status` after adding any OAuth credential because a new credential starts enabled.

## Safety rules

- Never print, log, copy, or commit API keys or OAuth token values.
- Never commit private `config.yaml` or files from the OAuth credential directory.
- Use `cliproxy-select`; do not manually edit credential JSON.
- Confirm before restarting the proxy when active requests may be interrupted.
- Report failed or skipped checks accurately.
- Commit or push only when the user asks.

See [`AGENT_SETUP.md`](AGENT_SETUP.md) for the complete proxy and account setup procedure.
