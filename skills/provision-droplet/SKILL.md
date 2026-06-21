---
name: provision-droplet
description: >
  Use when the user wants to spin up / create / launch / provision a
  DigitalOcean droplet (or "a remote dev box on DO") and connect to it from
  Codex as a remote SSH workspace.
---

# Provision a DigitalOcean droplet as a Codex remote workspace

Follow these steps in order. Do not skip or reorder them.
**Only the installed Codex DigitalOcean App tools and the bundled Python scripts
may be used. doctl, ad hoc integration configs, and any other DigitalOcean CLI
tools are prohibited.**

## Before you start

- **Prerequisites:** a funded DigitalOcean account, the installed and
  authenticated Codex DigitalOcean App, a local `ssh`/`ssh-keygen` (OpenSSH),
  Python 3, and the Codex desktop app.
- **Cost:** the droplet bills **hourly from creation until you delete it**.
  Sizes in step 5 show approximate monthly rates. Remind the user to delete it
  when done (see *Cleanup* below).
- **Time:** end-to-end takes ~10–15 minutes — roughly 7 minutes waiting for the
  droplet to boot (step 7) plus up to 7 minutes for cloud-init (step 8). This is
  normal; do not abort.
- **Locate the bundled scripts first (do this before Step 2).** The helper
  scripts live in the `scripts/` folder **next to this `SKILL.md`** (i.e.
  `provision-droplet/scripts/`). Your current working directory is **not** the
  skill directory, so bare relative paths like `scripts/keygen.py` will fail.
  Resolve the absolute directory that contains this `SKILL.md` and call it
  `<skill_dir>`. If you don't already know it, find it — the installed plugin may
  nest it under a version folder (e.g. `.../<version>/provision-droplet/`), so
  locate the directory that actually contains `scripts/keygen.py`. Use
  `<skill_dir>/scripts/<name>.py` (an absolute path) in **every** command below.

## Step 1 — Verify DigitalOcean App access

Use the installed Codex DigitalOcean App for all DigitalOcean operations. Do not
register or log in to separate plugin-owned app integrations.

The app uses a **templated MCP URL** (`{workspace}.digitalocean.mcp`). The
`{workspace}` value **cannot** be set from the plugin manifest — the user must
supply it during the app's connection/setup flow. This workflow uses **two**
workspaces:

- **`accounts`** (`accounts.digitalocean.mcp`) — SSH key tools: `key-create`,
  `key-list`, `key-delete`.
- **`droplets`** (`droplets.digitalocean.mcp`) — droplet tools:
  `droplet-create`, `droplet-get`, `droplet-delete`.

Connect the DigitalOcean App to **both** workspaces. If the user is prompted for
a workspace and is unsure, tell them to enter `accounts` and `droplets`.

Confirm that the DigitalOcean App tools from both workspaces are available
before continuing. If the tools are missing or unauthenticated, stop and tell
the user to install or authenticate the Codex DigitalOcean App (workspaces
`accounts` and `droplets`). Do not fall back to doctl, API tokens, or a local
integration config.

## Step 2 — Generate SSH key pair

```bash
python3 <skill_dir>/scripts/keygen.py
```

Parse the JSON output and keep these values for the steps below:
`prefix`, `name`, `key_name`, `key_path`, `pub_key`.

How these relate (all derived from one random `prefix` like `bright-hawk-a3f2`):
- `name` = `codex-<prefix>` — the **droplet name** and the **local SSH alias**
  (they are identical).
- `key_name` = `codex-key-<prefix>` — the label for the key on DigitalOcean's
  side only.
- `key_path` — the local private key file.

## Step 3 — Upload SSH public key

Call DigitalOcean App tool **`key-create`** (**`accounts`** workspace):

| Parameter | Value |
|-----------|-------|
| `Name` | `key_name` from step 2 |
| `PublicKey` | `pub_key` from step 2 |

Extract `ssh_key.id` from the response — this is `<key_id>`.

If the call fails because a key with that name or fingerprint **already exists**
(e.g. a previous run), do not create a duplicate: call DigitalOcean App tool
**`key-list`** (**`accounts`** workspace), find the entry whose `name` matches
`key_name` (or whose
fingerprint matches the uploaded key), and use its `id` as `<key_id>`.

## Step 4 — Choose a region

Ask the user, in chat:

> Use the default region **`nyc3`** (New York, US), or pick a custom region?

If they choose the default, use `nyc3` as `<region>` and continue.

If they want a custom region, present this list and ask them to reply with a
slug:

| Slug | Location |
|------|----------|
| `nyc3` *(default)* | New York, US |
| `sfo3` | San Francisco, US |
| `tor1` | Toronto, CA |
| `lon1` | London, UK |
| `fra1` | Frankfurt, DE |
| `ams3` | Amsterdam, NL |
| `sgp1` | Singapore, SG |
| `blr1` | Bangalore, IN |
| `syd1` | Sydney, AU |

Validate their reply against this table. If it is not one of these slugs, ask
again — do not pass an unlisted value through. The chosen slug is `<region>`.

## Step 5 — Choose a droplet size

Ask the user, in chat:

> Use the default size **`s-2vcpu-4gb`** (2 vCPU / 4 GB, ~$24/mo), or pick a
> custom size?

If they choose the default, use `s-2vcpu-4gb` as `<size>` and continue.

If they want a custom size, present this list and ask them to reply with a
slug. Every size below is above the **1 vCPU / 2 GB** floor required by the
Codex Universal image. Prices are approximate — confirm in the DigitalOcean
dashboard.

| Slug | vCPU | RAM | Tier | ~$/mo |
|------|------|-----|------|-------|
| `s-2vcpu-4gb` *(default)* | 2 | 4 GB | Shared basic | $24 |
| `s-4vcpu-8gb` | 4 | 8 GB | Shared basic | $48 |
| `s-8vcpu-16gb` | 8 | 16 GB | Shared basic | $96 |
| `c-2` | 2 | 4 GB | Premium CPU-optimized | $42 |
| `g-2vcpu-8gb` | 2 | 8 GB | Premium general-purpose | $63 |

Validate their reply against this table. If it is not one of these slugs, ask
again — do not pass an unlisted value through. The chosen slug is `<size>`.

## Step 6 — Create droplet

Call DigitalOcean App tool **`droplet-create`** (**`droplets`** workspace):

| Parameter | Value | Notes |
|-----------|-------|-------|
| `Name` | `name` from step 2 | |
| `Region` | `<region>` from step 4 | |
| `Size` | `<size>` from step 5 | |
| `ImageID` | `233103029` | DigitalOcean **Codex Universal** image |
| `SSHKeys` | `["<key_id>"]` | |

Extract `droplet.id` from the response — this is `<droplet_id>`.

If `droplet-create` fails, show the user the error and handle it by cause — do
not blindly retry the same values:
- **Size not available in this region** (premium tiers like `c-2` and
  `g-2vcpu-8gb` are not in every region): go back to step 4 or 5 and pick a
  different region/size combination.
- **Payment / quota / limit** errors: stop and tell the user to resolve it in
  the DigitalOcean dashboard, then re-run.
- **Invalid image or any other error:** stop and report it.

The uploaded SSH key from step 3 is harmless to leave, but if you abort here see
*Cleanup* below.

## Step 7 — Wait for droplet to become active

Poll DigitalOcean App tool **`droplet-get`** (**`droplets`** workspace) with
`ID: <droplet_id>`, waiting **20 seconds** between calls.

Repeat until the response has `status == "active"` **and** `networks.v4`
contains an entry with `type == "public"`. Extract `ip_address` from that entry
— this is `<ip>`.

Poll **at most ~21 times (about 7 minutes)**. Keep polling with the DigitalOcean
App — do not use doctl or any other tool to check status. If it is still not
active after ~21 attempts, stop, report it to the user, and offer to delete the
droplet (see *Cleanup*).

## Step 8 — Configure local SSH

```bash
python3 <skill_dir>/scripts/configure_ssh.py \
  --alias codex-<prefix> \
  --ip <ip> \
  --user root \
  --key-path <key_path>
```

**⚠️ Do not interrupt this step.** It waits for cloud-init to finish (up to 7
minutes) and prints a `⏳` status line every 5 seconds — that output means it is
working normally. **Do not run any other commands. Wait for the line
`DROPLET READY`.** If the script exits with an error instead, report it and
offer to delete the droplet (see *Cleanup*).

## Final step: adding it to Codex

There is **no supported automation command** to register a desktop remote SSH
project yet (tracking: openai/codex#21554). Because the script writes the host
into `~/.ssh/config`, the Codex App auto-detects it. Tell the user to open:
**Codex App → Settings → Connections → Add SSH Host → pick the alias → choose
the remote folder.**

## Cleanup (on failure or when done)

The droplet bills hourly until deleted. To tear down:

1. **Delete the droplet** — DigitalOcean App tool **`droplet-delete`**
   (**`droplets`** workspace) with `ID: <droplet_id>`.
2. **Delete the SSH key** (optional) — DigitalOcean App tool **`key-delete`**
   (**`accounts`** workspace) with the `<key_id>` from step 3.

Always confirm with the user before deleting. Do not use doctl for cleanup.
