# Agent setup: Claude Code, pinned OAuth accounts, and Codex priority mode

This file is an execution procedure for a coding agent configuring Claude Code to use this CLIProxyAPI fork. Follow the steps in order and keep all credentials private.

## Safety contract

An agent following this guide may:

- run OAuth login commands requested by the operator;
- install and run `tools/cliproxy-select`;
- edit the operator's private `config.yaml`; and
- edit the operator's Claude Code user settings when authorized.

An agent must not:

- print, copy, summarize, or commit OAuth access tokens, refresh tokens, ID tokens, credential JSON contents, or API keys;
- place a real key or email address in repository documentation;
- edit `config.example.yaml` as a substitute for the operator's private configuration;
- manually edit OAuth credential JSON files—use `cliproxy-select` instead; or
- enable Claude Code `/fast` for `gpt-5.6-sol` or another custom GPT model ID.

Use placeholders such as `YOUR_LOCAL_CLIPROXY_API_KEY`, `claude-user@example.com`, and `codex-user@example.com` in any reusable files.

## Prerequisites

Confirm the following before making changes:

- CLIProxyAPI is installed and can serve `http://127.0.0.1:8317`.
- The `cli-proxy-api` binary is available.
- Python 3 is installed.
- The host is macOS or Linux; `cliproxy-select` uses POSIX `fcntl` locking.
- The operator has a private CLIProxyAPI configuration file.

Export its path:

```bash
export CLIPROXY_CONFIG=/path/to/config.yaml
```

The selector reads `auth-dir` from this file. If it cannot find a configured directory, it defaults to `~/.cli-proxy-api`.

## 1. Authenticate each account

Run the Claude OAuth login once for every Claude account the operator wants available:

```bash
cli-proxy-api -config "$CLIPROXY_CONFIG" -claude-login -no-browser
```

Run the Codex device login once for every Codex account the operator wants available:

```bash
cli-proxy-api -config "$CLIPROXY_CONFIG" -codex-device-login
```

Credential files are written under the configured `auth-dir`. A newly added credential starts enabled, so account selection must be checked again after every new login.

Do not inspect or display raw credential files.

## 2. Install the account selector

From this repository:

```bash
mkdir -p ~/.local/bin
install -m 755 tools/cliproxy-select ~/.local/bin/cliproxy-select
```

Supported path overrides:

| Variable | Purpose |
| --- | --- |
| `CLIPROXY_CONFIG` | CLIProxyAPI configuration file |
| `CLIPROXY_AUTH_DIR` | OAuth credential directory |
| `XDG_STATE_HOME` | Parent directory for selector state |

The selector stores state outside the watched OAuth directory, never copies token values into that state, uses atomic writes and an exclusive lock, and guards against concurrent token-refresh writes. It changes only the top-level `disabled` value, although the credential JSON file is atomically rewritten in normalized form.

## 3. Pin the active accounts

List credential metadata without displaying tokens:

```bash
cliproxy-select list
```

Select exactly one account for each provider:

```bash
cliproxy-select set \
  --claude claude-user@example.com \
  --codex codex-user@example.com
```

A single-provider installation may omit the provider it does not use:

```bash
cliproxy-select set --claude claude-user@example.com
```

Verify the persisted selection and detect drift:

```bash
cliproxy-select status
```

Restore the disabled states captured immediately before the last `set` operation:

```bash
cliproxy-select rollback
```

Selection is global for every client using the same CLIProxyAPI instance. Run `cliproxy-select status` after adding a credential or changing account configuration.

The selector writes its state before changing credentials so an interrupted operation remains diagnosable. If an operation fails, run `cliproxy-select status`; it reports any mismatch between persisted and active selection.

## 4. Configure reasoning aliases and priority routing

Add the following to the operator's private `config.yaml`:

```yaml
oauth-model-alias:
  codex:
    - name: "gpt-5.6-sol"
      alias: "gpt-5.6-sol-low"
      fork: true
    - name: "gpt-5.6-sol"
      alias: "gpt-5.6-sol-medium"
      fork: true

payload:
  override:
    - models:
        - name: "gpt-5.6-sol"
          protocol: "codex"
      params:
        "reasoning.effort": "high"
        service_tier: priority
    - models:
        - name: "gpt-5.6-sol-medium"
          protocol: "codex"
      params:
        "reasoning.effort": "medium"
        service_tier: priority
    - models:
        - name: "gpt-5.6-sol-low"
          protocol: "codex"
      params:
        "reasoning.effort": "low"
        service_tier: priority
```

The rule order is required. An aliased request can match both the resolved base model and its alias, and later override rules win. Keep the medium and low rules after the base high-effort rule.

CLIProxyAPI normally hot-reloads validated configuration changes. Restart it only when a validated change was not applied, no requests are in flight, and the operator has authorized the interruption.

Do not apply `service_tier: priority` to Claude-protocol models.

## 5. Configure Claude Code

Use settings equivalent to the following in the operator's user-level Claude Code settings:

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

Replace the API-key placeholder only in the operator's private settings. Never copy the real value into chat, logs, commits, issue bodies, or documentation.

The compaction trigger is approximately 244,800 tokens (`272000 × 90%`). These settings alter Claude Code's compaction calculation only; they do not increase an upstream model's context limit.

Claude Code `/fast` is model-gated to supported Anthropic models. Enabling it while a custom GPT model is active can switch the session to an Opus model instead of accelerating GPT. Keep `fastMode` false and use Codex `service_tier: priority`.

## 6. Verify the setup

Check account selection:

```bash
cliproxy-select status
```

Ensure `CLIPROXY_API_KEY` has been exported from the operator's private secret source without displaying its value. Then check the model catalog:

```bash
curl -s http://127.0.0.1:8317/v1/models \
  -H "Authorization: Bearer $CLIPROXY_API_KEY" \
  | jq -r '.data[].id' \
  | grep -F 'gpt-5.6-sol'
```

The output should include the base, medium, and low model IDs. Then send one minimal request through every provider the operator requires and confirm it succeeds.

Troubleshooting:

- Empty model grep: confirm the alias block is in the active private `config.yaml` and that the configuration watcher loaded it.
- HTTP 401: confirm the private local proxy key is correct without printing it.
- Selector drift: run `cliproxy-select status`, then deliberately re-run `set` or use `rollback`.
- Wrong reasoning effort: confirm alias-specific override rules appear after broader base or wildcard rules.
- Model unexpectedly changes to Opus: disable Claude Code `/fast`, keep `fastMode` false, and select `gpt-5.6-sol` again.

## File map

- [Main README](README.md)
- [Detailed operator guide](docs/claude-code-multi-account.md)
- [Account selector](tools/cliproxy-select)
- [Configuration examples](config.example.yaml)
