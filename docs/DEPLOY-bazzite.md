# Hermes Agent — Bazzite (rootless podman + SELinux Enforcing) deployment notes

This file captures host-specific deviations from upstream's `docker-compose.yml` that were required to bring Hermes up cleanly on a Bazzite 44 (Fedora 44 immutable) workstation running **rootless podman** with **SELinux Enforcing**. Upstream's recipe assumes rootful Docker; without these adjustments the stack crash-loops on permission and userns errors.

These notes belong on the **`bazzite-deploy` branch** of the fork. Fork `main` stays clean so `gh repo sync` continues to fast-forward from upstream.

## TL;DR — required deltas vs upstream

`docker-compose.yml` (both services):

```yaml
services:
  gateway:
    userns_mode: "keep-id:uid=10000,gid=10000"   # ← added
    volumes:
      - ~/.hermes:/opt/data:z                     # ← :z lowercase, NOT :Z
    environment: []                               # ← do NOT pass HERMES_UID/HERMES_GID
  dashboard:
    userns_mode: "keep-id:uid=10000,gid=10000"   # ← added
    volumes:
      - ~/.hermes:/opt/data:z                     # ← :z lowercase, NOT :Z
    # environment block deleted entirely
```

`~/.hermes/config.yaml` `model:` block (point at local llama.cpp on the host):

```yaml
model:
  default: "Qwen3.6-35B-A3B-UD-Q8_K_XL.gguf"   # whatever your llama.cpp loaded
  provider: "custom"                            # NOT "llamacpp" — see gotcha #5
  api_key: "sk-noauth"                          # llama.cpp ignores it; Hermes requires non-empty
  base_url: "http://localhost:8082/v1"          # reachable from container via network_mode: host
```

Bring it up:

```bash
git checkout bazzite-deploy
podman compose build
podman compose up -d
```

## The five gotchas hit, in the order they fired

### 1. Port 9119 collides with agent-one's pasta range

- Hermes dashboard default port is `9119`.
- The host's `agent-one` container has a rootlessport `pasta` forwarder reserving the **entire** `9100-9200` range (`-t 9100-9200:9100-9200`), so every port in that block is bound on `*:9119` even when the container itself isn't using it.
- **Symptom:** `LISTEN *:9119  pasta.avx2` shown in `ss -tlnp`; compose-up fails or dashboard cannot bind.
- **Fix:** reconfigure `agent-one` quadlet to split the range as `9100-9118 + 9120-9200`. (Done out-of-band; recorded as a separate memory.)
- **Lesson:** Before claiming any port, run `ss -tln` AND check pasta-reserved ranges. Pasta binds the whole mapped range at the host level.

### 2. SELinux `:Z` (private MCS) breaks multi-container shared volume

- Compose recipe shipped with `~/.hermes:/opt/data` (no SELinux label). Adding `:Z` (uppercase = private MCS categories) made the dashboard service unable to read/write files relabelled by the gateway service.
- Each `:Z` mount triggers podman to relabel the host directory with **unique** MCS categories (e.g. `c156,c690`). Gateway and dashboard get different categories → mutual SELinux denial.
- **Symptom:** Container reads OK, writes get `[Errno 13] Permission denied: '/opt/data/skills/...'` despite UID/GID looking correct.
- **Fix:** use `:z` (lowercase = shared label, no per-container categories) on **both** services. Only one of them sees the volume "first", but both see the same shared label.
- **Lesson:** `:Z` is for *single-tenant* volumes. For multi-container shared dirs, always `:z`.

### 3. Rootless podman + `HERMES_UID=$(id -u)` produces unwritable host files

- Upstream's compose passes `HERMES_UID=$(id -u)` (= host UID 1000 here). The entrypoint runs as container UID 0, then `usermod -u 1000 hermes` and `chown -R hermes:hermes /opt/data`.
- Under rootless podman *without* a `userns_mode`, container UID 1000 maps to host UID **525287** (subuid offset, not 1000). The chown ends up writing host UID 525287 — but the host directory is owned by host UID 1000 (the user). The running rootless process (host UID 1000 = container UID 999 in default mapping) is then *not* the file owner, so writes fail with EACCES.
- **Symptom:** `Warning: chown failed (rootless container?) — continuing anyway` followed by `mkdir: cannot create directory '/opt/data': Permission denied` × N, ~115 restarts.
- **Fix:** use `userns_mode: "keep-id:uid=10000,gid=10000"` to map host UID 1000 directly to container UID 10000 (the hermes user baked into the image at build time, `useradd -u 10000`). Files owned by host UID 1000 then appear inside the container as owned by UID 10000 (hermes) — no chown needed, writes work.
- **Lesson:** Rootless podman + bind mounts + an image-baked UID requires explicit userns mapping. Don't rely on the in-image `usermod` dance.

### 4. Plain `userns_mode: keep-id` conflicts with the entrypoint's `usermod`

- First fix attempt used `userns_mode: keep-id` (no `:uid=…` specifier).
- That maps host UID 1000 → container UID 1000 and pre-populates UID 1000 in `/etc/passwd` as the synthetic running user. The entrypoint's `usermod -u 1000 hermes` then fails because UID 1000 is already taken.
- **Symptom:** Logs flood with `usermod: UID '1000' already exists`, ~25 restarts.
- **Fix:** use `keep-id:uid=10000,gid=10000` (explicit container UID), AND remove the `HERMES_UID` / `HERMES_GID` env vars from compose so the entrypoint skips its `usermod` branch entirely.
- **Lesson:** If the image already has a fixed-UID user, map the host user *to that UID* (`keep-id:uid=10000`) instead of letting the entrypoint try to re-number the user at runtime.

### 5. `provider: "llamacpp"` is documented but rejected by the validator

- `cli-config.yaml.example` (line 35–37) and the README claim: *"Aliases: 'ollama', 'vllm', 'llamacpp' all map to 'custom'."*
- `hermes doctor` actually rejects `llamacpp` with: *"model.provider 'llamacpp' is not a recognised provider (known: …, custom, …, lmstudio, …)"*. The alias resolution is not run during config validation.
- **Fix:** use the canonical `provider: "custom"`. Functionally identical.
- **Lesson:** When picking a provider, cross-check against `hermes doctor` output, not just the example/README.

## Sequencing summary (full restart-storm tally)

| Attempt | Change | Restart count before stable |
|---------|--------|------------------------------|
| 1 | Default compose, no userns_mode, `:Z` | 115 (skills_sync chown failure) |
| 2 | Add `keep-id` (no uid) | 25 (`usermod: UID 1000 already exists`) |
| 3 | `keep-id:uid=10000,gid=10000`, removed env, kept `:Z` | 27 (MCS denial) |
| 4 | Switched `:Z` → `:z` | **0 — clean start** |
| 5 (config) | Switched `provider: llamacpp` → `custom` | doctor passes |

## Verification

```
$ podman ps --format '{{.Names}}\t{{.Status}}' | grep hermes
hermes              Up …
hermes-dashboard    Up …

$ curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:9119/
HTTP 200

$ podman exec hermes curl -sS http://localhost:8082/v1/models | jq '.data[].id'
"Qwen3.6-35B-A3B-UD-Q8_K_XL.gguf"

$ podman exec hermes /opt/hermes/.venv/bin/hermes doctor | grep -E '✗|provider'
✗ ~/.local/bin/hermes not found    # benign — only matters outside the venv
```

## Host facts (Bazzite 44 / Lenovo P920)

- Rootless podman 5.8.0 with `pasta` networking
- SELinux: Enforcing
- Container-first policy (no host pip/npm/etc.)
- `~/.hermes` on `/var/home/bahram/.hermes` (`/home` is a symlink)
- llama.cpp router on `http://localhost:8082/v1` (host network)
- Host UID 1000 = `bahram`; in-image hermes user is UID 10000

## Sync hygiene

This branch (`bazzite-deploy`) intentionally diverges from upstream `docker-compose.yml`. Keep `main` in sync with `upstream/main` (fast-forward only) and rebase `bazzite-deploy` on top of new `main` after each upstream pull. The five edits above are small and well-localized so manual conflict resolution is straightforward.
