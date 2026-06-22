# droplet-workspace

A Codex plugin that provisions a DigitalOcean droplet from the **Codex Universal**
image (id `233103029`) and wires it up as a remote SSH workspace for the Codex
desktop app. It uses the installed Codex DigitalOcean Accounts and Droplets
apps, creates and boots the droplet without doctl or API tokens, configures
local SSH access, and hands off to Codex. The user picks the region and size
interactively (with sensible defaults), and the droplet is billed hourly until
deleted.

## Components

```
droplet-workspace/
  .codex-plugin/plugin.json          # plugin manifest (name, version, skills/apps paths)
  .app.json                          # connected DigitalOcean app manifest (Accounts + Droplets)
  skills/provision-droplet/
    SKILL.md                         # the step-by-step workflow Codex follows
    ssh_config.tmpl                  # ~/.ssh/config Host block template
    scripts/
      keygen.py                      # generates a unique name + ed25519 SSH key pair
      configure_ssh.py               # writes SSH config, scans host keys, waits for cloud-init
```

- **Codex DigitalOcean apps** — two required external app integrations. Install
  and authenticate both in Codex before running this plugin:
  - **DigitalOcean Accounts** — SSH key tools (`key-create`, `key-list`,
    `key-delete`).
  - **DigitalOcean Droplets** — droplet tools (`droplet-create`, `droplet-get`,
    `droplet-delete`).
- **`.app.json`** — the plugin-local app dependency manifest. It binds this
  plugin to the two DigitalOcean app ids: Accounts
  `asdk_app_6a395072801c81918c90d5e3de24ea83` and Droplets
  `asdk_app_6a3950a7914c8191bc3ce04d2c6f230d`.
- **`SKILL.md`** — the orchestration guide. It drives the Codex DigitalOcean app
  tools and the two scripts in order, prompts the user for region/size, and
  explains the one manual Codex step at the end.
- **`scripts/keygen.py`** — stdlib-only; mints a unique `adjective-noun-hex4`
  prefix, derives the droplet name / DO key label, and creates the local SSH key.
- **`scripts/configure_ssh.py`** — stdlib-only; renders `ssh_config.tmpl` into
  `~/.ssh/config`, refreshes `known_hosts`, and probes SSH login until cloud-init
  finishes before printing the `DROPLET READY` handoff.
- **`ssh_config.tmpl`** — the `Host` block template (`{{ALIAS}}`, `{{IP}}`,
  `{{USER}}`, `{{IDENTITY_FILE}}`).

## The one manual step

There is no supported CLI to register a desktop remote SSH project in Codex
(tracking: openai/codex#21554). After provisioning, add the host via
**Codex App → Settings → Connections → Add SSH Host → pick the alias → choose
the remote folder.**
