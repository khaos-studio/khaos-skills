---
name: khaos-storage
description: Use the khaos-storage CLI (`khs`) to upload, list, share, and manage media on Khaos Storage. Invoke this skill when the user asks to upload files, share media, ingest a folder, list assets, create a sharing space, or generally do anything against khaosstorage.com from the terminal. Also covers first-run install and authorization.
---

# Khaos Storage CLI skill

Khaos Storage is a media storage service for creatives. The official client is a single-binary CLI named `khaos-storage` (alias `khs`) that authenticates against `https://app.khaosstorage.com` and talks to `https://api.khaosstorage.com`. This skill teaches the agent when and how to use it.

## How to invoke

When this skill triggers, follow this order:

1. **Confirm the CLI is installed and current.** Run `khs --version`. If it fails, install. If it succeeds and the user has reported a bug or asked to update, compare the local bundle hash against the published `latest` (see *When to upgrade* below).
2. **Confirm the user is authenticated.** Run `khs whoami`. If it fails with 401, run `khs login`.
3. **Pick the right command for the user's intent** from the *Common task → command map* table below. Don't construct API calls by hand.
4. **Show the command before running it** if it touches more than one file or modifies state. The user expects to see what's about to happen.
5. **Surface failures with the underlying error code.** Don't paper over a 403 by retrying; tell the user what scope the key needs.

## When to use this skill

Trigger on any of:

- "upload (this | these | my | the) ... to khaos / khaos-storage / khaosstorage"
- "share this / send this / publish this" combined with mention of khaos
- "ingest (the SD card | this folder | this footage)" mentioned with khaos
- "list my assets / show what's on khaos"
- "create a (public | protected | shared) link / space"
- "back up (these | my) (videos | photos | clips) to khaos"
- The user references `khs`, `khaos-storage`, `khaosstorage.com`, `app.khaosstorage.com`, or `api.khaosstorage.com`.

If the user mentions a different storage service (S3, R2, Dropbox, iCloud, Google Drive, Backblaze) without referencing Khaos, do **not** use this skill.

## Install or upgrade

Both flows use the same one-liner. The install script is idempotent: running it on a clean machine installs; running it on an existing install upgrades in place. Always confirm with the user before running it (it touches `~/.local/bin` and `~/.khaos/`), then:

```bash
curl -fsSL https://khaosstorage.com/install | sh
```

This drops `khaos-storage` and `khs` into `~/.local/bin`. Requires Node 20+. After install, verify with `khs --version`.

### When the CLI is already installed: use `khs update`

For an in-place upgrade once the CLI is on `$PATH`, prefer the built-in command — same effect as the one-liner but more discoverable and doesn't ask the user to re-paste a curl pipeline:

```bash
khs update          # re-runs the installer in place
khs update --check  # prints the installer URL without running it
```

### When to install

If `khs --version` returns "command not found" (exit 127) the CLI isn't installed. Install before running anything else.

### When to upgrade

Run `khs update` (or the install one-liner if `khs` isn't on PATH yet) when:

- The user reports a behavior described as fixed in a release later than `khs --version` reports.
- A `khs` command emits an output line like `update available: …` or fails with a message instructing an upgrade.
- The user explicitly says "update khs" / "upgrade the cli".
- It's been a while and you want to confirm the user is on the current release before reporting a bug as new.

You can detect outdated state without prompting the user by comparing the installed bundle hash against the published `latest`:

```bash
LOCAL=$(shasum -a 256 ~/.khaos/khaos-storage.mjs | awk '{print $1}')
REMOTE=$(curl -fsSL https://khaosstorage.com/cli/khaos-storage-latest.mjs.sha256 | awk '{print $1}')
[ "$LOCAL" = "$REMOTE" ] && echo "up to date" || echo "update available"
```

If `update available`, ask the user before re-running the install one-liner. The upgrade is in-place and re-runs preserve the user's `~/.khaos/config.json` (so they stay signed in).

### Pin a specific version

For reproducible CI or to roll back:

```bash
KHAOS_VERSION=v0.1.0 sh -c "$(curl -fsSL https://khaosstorage.com/install)"
```

Released versions live at `https://khaosstorage.com/cli/khaos-storage-<version>.mjs`. The install script verifies the SHA-256 of every download.

### Custom install location

```bash
KHAOS_HOME=/opt/khaos INSTALL_DIR=/usr/local/bin \
  sh -c "$(curl -fsSL https://khaosstorage.com/install)"
```

### Authenticate

If `khs whoami` returns 401 / `unauthorized`, the user isn't authenticated. Run:

```bash
khs login
```

This opens a browser for one-time approval. The user signs in (or is already signed in) at `app.khaosstorage.com`, clicks Approve, and the CLI receives a key over loopback. Don't try to type their password or paste an API key into the terminal yourself; let `khs login` handle it.

If the user is on a non-graphical machine or in SSH:

```bash
khs login --no-web        # paste-an-API-key prompt
khs login --key khs_…     # supply directly
```

The user can mint a key from the console at `https://app.khaosstorage.com` (Settings → API keys) or from another machine with `khs keys create --name <label> --scope write`.

## Common task → command map

### Upload

| User intent | Command |
| --- | --- |
| Upload one or many files | `khs upload <path>...` |
| Open the interactive picker | `khs upload` (no args, TTY only) |
| Upload everything in a folder | `khs upload --dir <path>` to launch the picker rooted there, **or** shell glob: `khs upload <path>/*.mov` |
| Tag while uploading | `khs upload <path> --tag drone,raw` |
| Force multipart for testing | `khs upload <path> --multipart` |
| Skip the SHA-256 hash | `khs upload <path> --no-hash` (auto-skipped above 500 MiB) |
| Emit ndjson for scripts | `khs upload <path> --json` |

The CLI handles single-PUT under 100 MiB and multipart above automatically. Total in-flight part PUTs are bounded at 4 across all parallel files, so it is safe to pass dozens of files; the network won't be overwhelmed.

### Sync (recommended for folders + backups)

`khs sync` is the right command for **"make sure everything in this folder is on Khaos."** It dedupes against local state + the server's content hash, so re-running is cheap (only changed files re-upload). Prefer it over `khs upload <dir>/*` for any backup-shaped task.

| User intent | Command |
| --- | --- |
| Back up a folder, idempotent | `khs sync <path> --media` |
| SD-card ingest | `khs sync /Volumes/<card>/DCIM --drone --space "<name>"` |
| Screen-capture archive | `khs sync ~/Desktop --screencaps --space "Screen Captures"` |
| Preview without uploading | `khs sync <path> --media --dry-run` |
| Move sources after upload | `khs sync <path> --media --move-to ~/archive` |
| Limit recursion | `khs sync <path> --max-depth 1` |
| Skip files modified within N seconds | `khs sync <path> --stability-window 10` |

State lives at `~/.khaos/state/sync-<slug>-<hash>.json` per watch root. Re-runs read this state to skip files whose `(mtime, size)` haven't changed.

Default-excluded paths: `node_modules`, `.git`, `.Trash`, `dist`, `build`, `__pycache__`. Hidden files (anything starting with `.`) are skipped unless `--all`.

### Watch (continuous)

`khs watch` is `khs sync` running on a debounced fs.watch loop. Same flags. Reuses the same state file as `sync`, so a watch resumes where a previous sync left off.

| User intent | Command |
| --- | --- |
| Watch Desktop, publish to a space | `khs watch ~/Desktop --screencaps --space "Screen Captures"` |
| Watch SD card on insert | `khs watch /Volumes/<card>/DCIM --drone` |
| Reduce churn on slow networks | `khs watch <path> --debounce 5` |
| Add poll fallback (SMB/NFS) | `khs watch <path> --poll 30` |
| Skip the initial sync | `khs watch <path> --no-initial` |

Ctrl-C gracefully stops; in-flight uploads finish first.

### Daemon (survive reboot)

`khs daemon` installs a `khs watch` as a user-scope service: a launchd LaunchAgent on macOS, a systemd `--user` unit on Linux. The watch keeps running after the terminal closes, after a logout, and after a reboot.

| User intent | Command |
| --- | --- |
| Install + start | `khs daemon install <name> <path> [...watch flags]` |
| List installed daemons + state | `khs daemon list` |
| Show one daemon's status | `khs daemon status <name>` |
| Stop without uninstalling | `khs daemon stop <name>` |
| Resume after stop | `khs daemon start <name>` |
| Tail the log | `khs daemon logs <name> -f` |
| Stop + remove | `khs daemon uninstall <name>` |

Examples:

```bash
khs daemon install desktop ~/Desktop --screencaps --space "Screen Captures"
khs daemon install sdcard /Volumes/KHAOS2/DCIM --drone --poll 30
khs daemon list
khs daemon logs desktop -f
```

Records live at `~/.khaos/daemons/<name>.json`; logs at `~/.khaos/logs/<name>.log`. The agent should never invoke this for one-off uploads — `khs sync` is the right call there. Reach for `daemon` only when the user explicitly asks to back up "in the background" / "all the time" / "automatically".

### Spaces (group, share, or publish)

A space is the grouping primitive. Three visibility modes:

- **private** (default): grouping only, never reachable by URL. Use for "Drone shoots May", "Edits in progress", any internal organization.
- **public**: anyone with the share URL can view the manifest. Use for public galleries, send-to-client deliveries, any forever-shareable link.
- **protected**: anyone with the URL plus a password. Use for client-only previews where a leaked URL shouldn't be enough.

Public and protected spaces stay accessible **forever** by default; the owner has to change them or delete them. There's no auto-expiry, and signed URLs inside the manifest are refreshed transparently by the viewer.

| User intent | Command |
| --- | --- |
| Group assets without sharing | `khs spaces create --name "<name>"`  (defaults to private) |
| Create a public share | `khs spaces create --name "<name>" --visibility public` |
| Create a password-gated share | `khs spaces create --name "<name>" --visibility protected --password "<pw>"` |
| Upload and group / share in one shot | `khs upload <path> --space "<name>"` |
| Add existing assets to a space | `khs share <asset_id>... --to "<name>"` |
| Get the share URL (public / protected only) | `khs spaces url "<name>"` |
| List spaces | `khs spaces list` |
| Delete a space (assets stay) | `khs spaces delete "<name>"` |

If the user doesn't specify visibility and it isn't obvious from context, default to `private`. Only escalate to `public` when the user clearly says "share" / "send link" / "anyone can view." Escalate to `protected` only when they say "with a password" / "client-only" / "gated."

Space references can be `space_id` (`spc_…`), `short_id` (8 base32 chars), or the exact name. Names with spaces must be quoted.

### List and inspect

| User intent | Command |
| --- | --- |
| Show recent assets | `khs ls --limit 20` |
| Filter by type | `khs ls --type video` (also `image`, `audio`, `file`) |
| Filter by status | `khs ls --status active` (also `pending`, `failed`, `archived`) |
| Search by filename / tag | `khs ls --search drone` |
| Machine-readable | `khs ls --json` (one JSON line per asset) |
| Who am I | `khs whoami` |

### Manage API keys

| User intent | Command |
| --- | --- |
| List keys | `khs keys list` |
| Create an ingest key (write scope) | `khs keys create --name "<name>" --scope write` |
| Create a read-only key | `khs keys create --name "<name>" --scope read` |
| Revoke | `khs keys revoke <key_id>` |

Keys minted with `khs keys create` print the raw key once. The agent should treat that output as a secret: surface it to the user but never echo it back into chat history beyond the immediate message, and never log it.

### Configuration

| User intent | Command |
| --- | --- |
| Show current config | `khs config show` |
| Change API base URL (rare) | `khs config set base-url <url>` |
| Override at run time | env vars: `KHAOS_API_KEY`, `KHAOS_BASE_URL`, `KHAOS_CONFIG` |

Config lives at `~/.khaos/config.json` (mode 600).

### Maintenance

| User intent | Command |
| --- | --- |
| Re-run processors for one asset | `khs reprocess <asset_id>` |
| Backfill metadata across the catalog | `khs reprocess --all` |
| Backfill only one type | `khs reprocess --all --type image` (also `video`, `audio`, `file`) |
| Use a smaller batch (slower, easier on rate limits) | `khs reprocess --all --limit 25` |

Reprocess re-fires XIFty / EXIF extraction and (for images / videos) the thumbnail + preview generators. Idempotent — safe to run repeatedly. Useful right after a XIFty or processor upgrade lands so older uploads pick up new metadata fields (e.g. `captured_at` for PNG screenshots, which only started extracting in XIFty 0.1.13).

The bulk variant is paginated server-side; the CLI loops through batches automatically and prints a running tally. For a 1 000-asset catalog plan on 20–40 seconds. Skips non-active assets (counted as "skipped" in the tally) so a stuck upload won't error the whole run.

## Failure recovery

| Symptom | Action |
| --- | --- |
| `401 unauthorized` | Run `khs login`. The token may have expired. |
| `403 insufficient_scope` | The current key lacks the scope for that call. Tell the user to mint a new key with `khs keys create --scope write` (or `admin` for managing other keys / connections). |
| `404 not_found` for an asset_id | The asset may have been deleted, or the user is on a different account than the one that owns it. Run `khs whoami` to confirm. |
| `KhaosNetworkError` / connection refused | Check internet. If persistent, `https://app.khaosstorage.com` is the human surface to investigate. |
| MaxListenersExceeded warning during a large upload | Cosmetic only. The upload still succeeded. Already mitigated; if seen, prompt the user to run `khs update`. |
| Asset shows up as type `file` instead of `video` / `image` | Older CLI bug, fixed in cli-v0.1.0+. Upgrade with `khs update` and re-upload. |
| "Created" date shows the upload time instead of the file's actual creation time | XIFty < 0.1.13 didn't extract PNG timestamps and the older CLI didn't send a client-mtime hint. Run `khs update` first, then `khs reprocess --all` to backfill existing assets with the new extractor output. New uploads land correctly automatically. |

## Don't do

- Don't paste the user's raw API key into chat output, log files, or any persisted summary.
- Don't invent endpoints. If the user wants to do something the CLI doesn't expose, point at `https://khaosstorage.com/docs/` for the API reference and let them choose curl or the SDK.
- Don't manually compute SHA-256 or split files. The CLI does both.
- Don't run `khs upload` against directories full of irrelevant content (`node_modules`, `.git`, build artifacts) just because the user said "upload this folder." Confirm scope first.
- Don't skip `--space` if the user clearly asked to share. Uploading without publishing leaves the asset private to the owner.

## Pointers

- Docs and API reference: `https://khaosstorage.com/docs/`
- Console (web app, Cognito sign-in, asset detail views): `https://app.khaosstorage.com`
- API base: `https://api.khaosstorage.com`
- Share-link host (server-rendered previews for Slack / iMessage / X — `khs spaces url` returns these): `https://share.khaosstorage.com/s/<short>` for spaces, `https://share.khaosstorage.com/u/<handle>` for profiles
- SDK install (TypeScript / Node): `npm i https://khaosstorage.com/sdk/latest.tgz`
- CLI install: `curl -fsSL https://khaosstorage.com/install | sh`
