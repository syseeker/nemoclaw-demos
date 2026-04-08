# Connecting Google Workspace to OpenClaw in a NemoClaw Sandbox

This guide walks you through giving an OpenClaw agent access to Gmail, Calendar, and Drive using `gogcli` — a Google Workspace CLI — running inside an OpenShell sandbox. By the end, your agent will be able to search your inbox, summarize calendar events, and browse Drive files on your behalf, with every outbound API call subject to NemoClaw's egress approval flow.

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Running OpenClaw sandbox | A working OpenShell sandbox with OpenClaw. See [NemoClaw hello-world setup](https://github.com/NVIDIA/NemoClaw). |
| GCP OAuth credentials | A client secret JSON file downloaded from [Google Cloud Console](https://console.cloud.google.com) — see [Getting the GCP OAuth credentials JSON](gogcli-skill/README.md#getting-the-gcp-oauth-credentials-json) in `gogcli-skill/README.md`. |
| Go toolchain | Go 1.21+ (installed automatically by `bootstrap.sh` if missing). |

## Part 1: Bootstrap (Build, Credentials, Token Server, Sandbox)

The bootstrap script handles everything end-to-end:

1. **Installs Go** if not found or below 1.21.
2. **Clones and builds `gogcli`** (if the binary isn't already present).
3. **Runs the GCP OAuth consent flow** — a browser window opens for you to sign in and grant access.
3. **Starts the token server** on the host, which holds the refresh token and serves short-lived access tokens.
5. **Pushes gogcli into the sandbox** with a thin wrapper that fetches tokens from the host on each call.
6. **Applies the network policy** restricting sandbox egress to Google APIs (read-only).
7. **Uploads the skill** and restarts the OpenClaw gateway.

```bash
cd <nemoclaw-demos-repo>
export GOG_KEYRING_PASSWORD=<password>

./gogcli-skill/bootstrap.sh \
  --credentials /path/to/client_secret.json \
  --email your@gmail.com \
  --sandbox <sandbox-name>
```

`GOG_KEYRING_PASSWORD` is the password used to encrypt the local token file. Use the same value in every subsequent step.

## Part 2: Re-deploy (After a Reboot or Sandbox Reset)

`setup.sh` restarts the token server, re-uploads the sandbox wrapper, and reapplies the network policy — without repeating the OAuth consent flow:

```bash
cd <nemoclaw-demos-repo>
GOG_KEYRING_PASSWORD=<password> \
  ./gogcli-skill/setup.sh <sandbox-name>
```

Replace `<sandbox-name>` with your OpenShell sandbox name (e.g. `email`).

Verify inside the sandbox:

```bash
openshell sandbox connect <sandbox-name>
/sandbox/.config/gogcli/gog auth list
```

## Trying It Out

Open the OpenClaw web UI and try these prompts:

- "Search my Gmail for unread messages from NVIDIA and summarize them."
- "Check my calendar for meetings tomorrow and give me a prep briefing."
- "List the most recent files in my Google Drive."

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `gog auth list` shows no accounts | Re-run `bootstrap.sh` to redo the OAuth consent flow. |
| `gog: could not reach token server` inside sandbox | The host-side token server isn't running. Re-run `setup.sh` to restart it, or check `~/.config/gogcli/token-server.log` for errors. |
| `l7_decision=deny` for `curl` to host IP | The `google_token_server` policy block is missing or has the wrong IP/port. Re-run `setup.sh` to reapply. |
| `l7_decision=deny` for `gmail.googleapis.com` | The Google API policy blocks weren't applied. Re-run `setup.sh` and confirm `google_gmail` / `google_calendar` / `google_drive` appear in `openshell policy get --full <sandbox-name>`. |
| Agent doesn't know about `gog` | Confirm `SKILL.md` was uploaded to `/sandbox/.openclaw/skills/gogcli/` and the gateway was restarted. |
| Gmail send fails | Only `gmail.readonly` scope is enabled in the current GCP project. Read and search operations work; sending requires the Gmail send scope to be added to your OAuth client. |
