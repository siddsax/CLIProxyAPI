# Claude Code: pinned OAuth accounts and Codex priority mode

This guide configures Claude Code to use CLIProxyAPI with:

- one explicitly selected Claude OAuth account for Claude models;
- one explicitly selected Codex OAuth account for GPT models;
- persistent account selection across Claude Code sessions;
- model aliases for high, medium, and low reasoning effort; and
- Codex `service_tier: priority`, the Codex-side equivalent of a faster service tier.

## Important: Claude Code Fast Mode is not Codex priority mode

Claude Code's `/fast` feature is model-gated to supported Anthropic models. Enabling it while a custom model such as `gpt-5.6-sol` is active can switch the session to an Opus model instead of accelerating GPT.

For Codex models, keep Claude Code Fast Mode off and set this request parameter through CLIProxyAPI:

```yaml
service_tier: priority
```

This preserves the `gpt-5.6-sol` model while requesting Codex's priority service tier.

## Authenticate multiple accounts

Set the configuration path for your installation first:

```bash
export CLIPROXY_CONFIG=/path/to/config.yaml
```

Claude OAuth:

```bash
cli-proxy-api -config "$CLIPROXY_CONFIG" -claude-login -no-browser
```

Codex OAuth using device login:

```bash
cli-proxy-api -config "$CLIPROXY_CONFIG" -codex-device-login
```

Run each command once per account. Credential JSON files are stored under the configured `auth-dir` (commonly `~/.cli-proxy-api`).

## Install the account selector

The included selector persists a single active account for each provider by updating the supported top-level `disabled` field in OAuth credential files.

```bash
mkdir -p ~/.local/bin
install -m 755 tools/cliproxy-select ~/.local/bin/cliproxy-select
```

The selector uses `./config.yaml` by default. Its paths can be overridden with:

```text
CLIPROXY_CONFIG   CLIProxyAPI configuration file
CLIPROXY_AUTH_DIR OAuth credential directory
XDG_STATE_HOME    selector state directory parent
```

The selector requires Python 3 and a POSIX system with `fcntl` support (macOS or Linux).

### List accounts

```bash
cliproxy-select list
```

### Select one Claude and one Codex account

```bash
cliproxy-select set \
  --claude claude-user@example.com \
  --codex codex-user@example.com
```

The selection is global for clients using the same CLIProxyAPI instance because every alternative credential for that provider is disabled.

### Inspect or undo the selection

```bash
cliproxy-select status
cliproxy-select rollback
```

The selector:

- changes only the `disabled` value while atomically rewriting normalized JSON;
- never prints or copies OAuth token values into its state file;
- uses atomic writes and an exclusive lock;
- guards against concurrent token-refresh writes; and
- stores rollback state outside the watched OAuth directory.

State is written before credential changes so an interrupted operation remains diagnosable; `cliproxy-select status` reports any resulting drift.

Run `cliproxy-select status` after adding a new OAuth credential because a newly created credential starts enabled.

## Configure GPT reasoning aliases and priority service tier

Add the following to `config.yaml`, adapting the upstream model name if necessary:

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

Keep the alias-specific override rules after the base-model rule. Aliased requests can match both, and later override rules win.

CLIProxyAPI watches configuration changes. A restart is normally unnecessary; only restart when the watcher does not apply a validated change and no requests are in flight.

## Point Claude Code at CLIProxyAPI

Example user-level Claude Code settings:

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

At a 272,000-token auto-compact window and a 90% threshold, Claude Code compacts at approximately 244,800 tokens. This setting changes Claude Code's compaction calculation; it does not increase the upstream model's actual context limit.

Keep the local API key private and do not commit a real key to this repository.

## Verify

```bash
cliproxy-select status
curl -s http://127.0.0.1:8317/v1/models \
  -H "Authorization: Bearer $CLIPROXY_API_KEY" \
  | jq -r '.data[].id' \
  | grep -F 'gpt-5.6-sol'
```

Then send a small request through each required provider and confirm both return successfully.
