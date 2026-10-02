# Travis's Tweaks

Local modifications to [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
on the `travis-tweaks` branch, built into the `travnewmatic/hermes-cloud` image.

This is a **coarse index of the *nature* of the changes** — what the fork does and
why — not a fine-grained changelog. For the exact per-commit history use:

```
git log upstream/main..travis-tweaks --oneline
git diff upstream/main travis-tweaks --stat
```

Only two files diverge from upstream: `Dockerfile` and `docker/stage2-hook.sh`.
Everything else is upstream, kept current by periodic merges of `upstream/main`.

---

## `Dockerfile` — extra CLI tools for homelab maintenance

Layers a set of ops/DevOps CLI tools onto the upstream image so the hermes pod can
do cluster + backup + monitoring work without a separate toolchain. Versions are
pinned via `ARG` (overridable at build time with `--build-arg`). Added after the
upstream `VOLUME ["/opt/data"]` line.

| # | Tool | Version | Notes |
|---|------|---------|-------|
| 1 | kubectl | v1.36.4 | |
| 2 | Google Cloud SDK (gcloud, gsutil) | apt | also pulls in `vim`, `tmux` |
| 3 | Helm | v4.2.4 | |
| 4 | Argo CD CLI | v3.4.2 | |
| 5 | Velero CLI | v1.17.2 | from velero-io (was vmware-tanzu) |
| 6 | B2 CLI | v4.7.1 | Backblaze B2 official binary |
| 7 | GitHub CLI (gh) | apt | official GitHub apt repo |
| 8 | Blogwatcher CLI | v0.2.1 | RSS/Atom feed reader |
| 9 | Fairwinds Nova | 3.2.0 | |
| 10 | krew | v0.5.0 | `KREW_ROOT` pinned to `/opt/krew` (not `$HOME/.krew`) so plugins work for the runtime UID 10000 user |
| 11 | Prometheus promtool | 3.14.0 | bundled in the prometheus release tarball |
| 12 | Alertmanager amtool | 0.34.0 | bundled in the alertmanager release tarball |
| 13 | apt tools: jq, rclone, unzip, dig (dnsutils), nmap, mtr (mtr-tiny) | apt | rclone from Debian trixie main repo (avoids pkg.rclone.org) |
| 14 | yq | v4.53.6 | mikefarah/yq Go binary |
| 15 | stern | 1.34.0 | multi-pod log tailing |
| 16 | kubeconform | v0.8.0 | K8s manifest schema validator |
| 17 | kopia | 0.23.1 | Velero backup engine CLI (not in trixie, from GitHub) |
| 18 | joplin | `${JOPLIN_VERSION}` (3.7.1) | `npm -g install --allow-scripts=sqlite3 joplin@${JOPLIN_VERSION}` + stable symlink `/usr/local/bin/joplin` — see notes below |

> **18. joplin — version pin + stable symlink (2026-10-02).** Two things:
>
> - **`>=3.7.1` is required, not just newer-nice.** The `tnewman-joplin-2` B2
>   sync target now rejects clients older than 3.7.0 at the API level
>   (`In order to synchronise, please upgrade your application to version
>   3.7.0+`). `joplin-mcp-server` bundles joplin `^3.6.2` in its npx install,
>   so the MCP sidecar's `sync` tool was permanently blocked. Bumping the
>   global joplin and pointing the MCP server at it via
>   **`JOPLIN_CLI=/usr/local/bin/joplin`** in the `mcp_servers.joplin.env`
>   block of `config.yaml` fixes it (the sidecar checks `JOPLIN_CLI` before
>   falling back to the stale npx-cached 3.6.2).
> - **`--allow-scripts=sqlite3` is required.** npm >=12 blocks dependency
>   install scripts by default; without it the sqlite3 native binding is
>   never downloaded and `joplin` crashes at startup
>   (`Cannot find module .../node_sqlite3.node`). This is exactly how the
>   original unpinned `npm -g install joplin` (3.6.2) got left broken in the
>   image.
> - The `$(npm prefix -g)/bin/joplin -> /usr/local/bin/joplin` symlink gives
>   the CLI a stable path that survives pm-managed node upgrades (the pm
>   node lives at a versioned path like `/opt/hermes/tools/node-26.7.0-linux-x64`).
>
> **Pod config side (required companion to this image change):** the
> `mcp_servers.joplin` entry in the hermes pod's `config.yaml` must carry
> `env.JOPLIN_CLI=/usr/local/bin/joplin` — otherwise the sidecar keeps
> resolving the stale npx-cached joplin 3.6.2 and sync stays blocked.

Also sets `ENV DEBIAN_FRONTEND=noninteractive` (no interactive prompts during
installs) and appends a build/run reference block at the bottom of the file.

## `docker/stage2-hook.sh` — file perms survive pod recreation

The s6-overlay `cont-init` hook (runs as root on every container start, after the
data volume is mounted) already tightened `$HERMES_HOME/.env` to `0600`. Extended
that same unconditional pattern to the CLI credential files, so they come back
`0600` after every pod restart instead of drifting to a permissive mode and
breaking the CLIs:

- `$HERMES_HOME/home/.config/argocd/config`
- `$HERMES_HOME/home/.config/gh/hosts.yml`
- `$HERMES_HOME/home/.config/gh/config.yml`
- `$HERMES_HOME/home/.ssh/id_ed25519` (the `$HOME`-based copy)
- `$HERMES_HOME/.ssh/id_ed25519` (ssh's real home — ssh ignores `$HOME`)

Each is guarded by the existing `refuse_symlinked_path` check and degrades with
`|| true` on a read-only volume. No-op when a file is absent.

---

<!-- Add a new section here when a new *category* of tweak lands (not per-commit). -->
