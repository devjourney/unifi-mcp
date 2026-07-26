# Security Hardening Checklist (local)

Local operator notes from the 2026-07-25 security review of this clone
(upstream `sirkirby/unifi-mcp`, HEAD `c1a7a1c`). The review found no malicious
behavior — no telemetry, no hidden endpoints, no install-time hooks, no
prompt-injection content. The items below are the deployment-side hardening
steps that code changes alone can't enforce.

## Local code changes applied (candidates for upstream PRs)

- TLS verification to the controller now **defaults to on** and the
  `verify_ssl` toggle is honored everywhere (previously four call sites
  hardcoded `ssl=False`: `dpi_manager`, `firewall_manager`, and two probes in
  the network `connection_manager`).
- API server binds `127.0.0.1` by default (was `0.0.0.0`); Docker still
  exposes it explicitly via `UNIFI_API_HTTP_HOST` / the Dockerfile CMD.
- CORS no longer uses wildcard methods/headers with credentials.
- All Docker images now run as a non-root user.
- `docker compose --profile api` now refuses to start without an explicit
  `UNIFI_API_DB_KEY` (no more baked-in dev passphrase).
- Plugin manifests (`plugins/*/.mcp.json`, `.claude-plugin/plugin.json`)
  default `UNIFI_VERIFY_SSL` to `true`.

## Before connecting a controller

- [ ] **TLS**: keep `UNIFI_VERIFY_SSL=true` (now the default). If your
      controller uses a self-signed certificate, either install its CA in the
      trust store or consciously set `UNIFI_VERIFY_SSL=false` and accept the
      on-LAN MITM risk. Same for `UNIFI_NETWORK_VERIFY_SSL`,
      `UNIFI_PROTECT_VERIFY_SSL`, `UNIFI_ACCESS_VERIFY_SSL`.
- [ ] **Least privilege**: create a dedicated UniFi local account / scoped API
      key for the MCP servers. Do not reuse your primary admin credentials.
- [ ] **Secrets**: keep credentials in `.env` (git-ignored) or a secret
      manager; never in shell profiles or committed files.

## Before connecting an AI agent (Claude, etc.)

The MCP servers expose genuinely high-impact tools: `access_unlock_door`,
`unifi_create/delete_firewall_policy`, `unifi_reboot_device`,
`protect_delete_recording`, and more. Policy gates default to **allowed**
when unset — set explicit denials for anything you don't want an agent to be
able to do even with confirmation (see `docs/permissions.md`):

- [ ] `UNIFI_POLICY_ACCESS_DOORS_UNLOCK=false` unless you truly want
      agent-driven physical access.
- [ ] `UNIFI_POLICY_DELETE=false` globally, then allow specific categories
      back as needed.
- [ ] Keep `UNIFI_TOOL_PERMISSION_MODE=confirm` (default). Never set
      `bypass` or the deprecated `UNIFI_AUTO_CONFIRM=true` for a server an
      LLM can reach.
- [ ] Keep response redaction on (`UNIFI_REDACT_SENSITIVE_FIELDS=true`,
      the default).

## API server (`--profile api`)

- [ ] Generate `UNIFI_API_DB_KEY` with `openssl rand -base64 32` (compose now
      fails fast without it).
- [ ] Keep the API bound to localhost or a trusted interface; front it with a
      reverse proxy + TLS if it must be reachable beyond the machine.
- [ ] The admin UI stores the API key in browser localStorage — use it only
      from a trusted browser/machine.

## Relay (optional feature — off unless you configure it)

- [ ] The relay only runs if you set `UNIFI_RELAY_URL`; it connects outbound
      to *your own* Cloudflare Worker. Treat `UNIFI_RELAY_TOKEN` as a secret;
      controller credentials are never sent to the relay.

## On every future `git pull` from upstream

- [ ] Re-check `.cursor/hooks.json` and `.codex/config.toml` — both are
      currently benign (no hook commands defined), but hooks added upstream
      would auto-execute in those editors.
- [ ] Re-check `plugins/*/scripts/` and `.claude-plugin`/`.mcp.json` launch
      commands (currently exact-pinned `uvx` PyPI installs).
- [ ] Diff `uv.lock` for new dependencies or non-PyPI sources
      (`grep -c 'files.pythonhosted.org' uv.lock` should account for all
      distribution URLs).
