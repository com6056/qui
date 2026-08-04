# Remote Agent Design (RFC)

Status: historical reference, see note below
Audience: qui maintainers + collaborators

> **Status note, August 2026:** this doc is from April 2026 and predates the maintainer-side work in PR #1820 and its successor stack (#1913/#1914/#1917), which went with an SSH transport and a `qui-helper` binary instead of the standalone HTTPS daemon described here. I'm not pushing this design over that one. The parts that still apply to either transport are the path safety analysis in section 7, the auth and at-rest encryption notes in section 8, and the failure semantics in section 11. Everything transport-specific is kept for reference only. Current state lives in the discussion: https://github.com/autobrr/qui/discussions/1814

## 1. Problem

Cross-seed's hardlink/reflink tree creation needs direct filesystem access to a qbit instance's save paths. `pkg/hardlinktree.Create` (and `reflinktree.Create`) call `os.Link`, `os.MkdirAll`, `os.Stat`, `os.SameFile` directly. This only works when qui runs on the same host as the qbit instance — gated today by `Instance.HasLocalFilesystemAccess`.

Network-mount workarounds don't bridge the gap reliably:

- **SSHFS** doesn't propagate `nlink`/inode semantics through FUSE — `os.SameFile` returns wrong answers, so the idempotency check at `pkg/hardlinktree/create.go:64-77` fails.
- **rclone** synthesises inodes; same problem worse.
- **NFS** works but most seedbox providers don't allow exporting it.

Users running qbit on a remote VPS / seedbox and qui on a home server can't use cross-seed. An agent-based solution has been endorsed by maintainers (see linked Discussion).

## 2. Scope

**v1: cross-seed only.** Adds a `qui-agent` daemon that runs colocated with a remote qbit instance and exposes the narrow set of filesystem ops cross-seed needs (link, mkdir, stat, remove, reflink, samefs). qui calls the agent over HTTPS. From cross-seed's perspective, the only thing that changes is which `LinkBackend` it gets from `internal/qbittorrent/pool`.

**v2 fast-follow: orphan-scan + dirscan migrate to the same agent.** The interface package and agent binary stay the same; v2 adds `ScanBackend`/`ReadBackend` and the corresponding `/v1/fs/walk` and `/v1/fs/read` endpoints. Operators do not need to keep network mounts around for those features; the v1 architecture is intentionally shaped so v2 is a strict superset.

This is a hard scope split, not a "we'll see" — the design's job is to make v2 a mechanical addition. The doc calls out where v2 differs in §13.

## 3. Non-goals (v1 and v2)

- Replacing/proxying any of qbit's HTTP API. Agent is FS-only.
- Multi-tenant security model. Single operator.
- Failover, clustering, agent discovery.
- Auto-upgrade of agents.
- Path translation. Agent operates in its host's namespace.
- **NAT traversal.** Agent needs an address qui can reach. v1 deployment shape — qui at home behind NAT, agent on a public seedbox/VPS — works without a tunnel. If the *agent* is behind NAT (rare for seedboxes), operator brings tailscale / cloudflared / wireguard / frp / chisel.
- Per-request replay protection. Once an attacker has the bearer token AND can MITM TLS, the model is already compromised.

## 4. Architecture

```
┌────────── qui host ──────────┐                 ┌────── qbit host ──────┐
│                              │                 │                       │
│  internal/services/crossseed │                 │   ┌────────────────┐  │
│              │               │  HTTPS          │   │ qui-agent      │  │
│              ▼               │  bearer + UUID  │   │  - chi router  │  │
│  fsops.{LinkBackend, ...}    │ ──────────────▶│   │  - auth        │  │
│              │               │                 │   │  - path guard  │  │
│        ┌─────┴─────┐         │                 │   │  - fsops.Local │  │
│        ▼           ▼         │                 │   └────────────────┘  │
│   fsops.Local  fsops.Remote ─┘                 │           │           │
│                                                │           ▼           │
└──────────────────────────────┘                 │   qbit save paths     │
                                                 └───────────────────────┘
```

Connection direction is qui → agent. Stateless HTTP/JSON. Matches qui's existing patterns (everything else is qui-initiated).

`Instance.FSAccessMode()` derives:
- `FSAccessLocal` ← `HasLocalFilesystemAccess && AgentURL == ""`
- `FSAccessRemote` ← `AgentURL != "" && !HasLocalFilesystemAccess`
- `FSAccessNone` ← otherwise

Setting both flags is rejected at save AND at instance load — direct DB writes don't bypass.

## 5. Interface design

`internal/fsops` introduces consumer-shaped interfaces — each call site takes the narrowest one it actually uses, per `AGENTS.md` line 73's "keep interfaces small (≤5 methods)" guideline. `CloneBackend` at 7 methods (5 embedded from `LinkBackend` + 2 declared) is the principled exception: reflink is an additive capability over the unit operations any tree backend needs. No umbrella `Backend` interface bundling everything.

```go
// package internal/fsops

type FileInfo struct {
    Name    string
    Size    int64
    Mode    fs.FileMode  // lstat semantics
    ModTime time.Time
    IsDir   bool
}

// LinkBackend — used by pkg/hardlinktree (5 methods)
type LinkBackend interface {
    Stat(ctx context.Context, path string) (FileInfo, error)
    Lstat(ctx context.Context, path string) (FileInfo, error)
    MkdirAll(ctx context.Context, path string, perm fs.FileMode) error
    Link(ctx context.Context, oldpath, newpath string) error
    Remove(ctx context.Context, path string) error
}

// CloneBackend — used by pkg/reflinktree (7 methods: 5 embedded + 2 declared)
type CloneBackend interface {
    LinkBackend
    Reflink(ctx context.Context, src, dst string) error
    SupportsReflink(ctx context.Context, dir string) (bool, string, error)
}

// SameFSBackend — used by pkg/fsutil/samefs (1 method)
type SameFSBackend interface {
    SameFilesystem(ctx context.Context, a, b string) (bool, error)
}

// v2 will add ScanBackend (Walk + Stat + Lstat + Remove + RemoveAll) and
// ReadBackend (ReadFile + LinkInfo). Both Local and Remote implementations
// already have the underlying capability — v2 is purely consumer-side.
```

Backends are accessed via `pool.GetFSBackend(ctx, instanceID) (fsops.Capabilities, error)` next to the existing `GetClient` — but **introduced in Phase 3, not Phase 1.** Phase 1's crossseed call sites pass `fsops.Local{}` directly; the resolver only matters once `Remote` exists. `Capabilities` is a struct of named sub-interfaces (`Link`, `Clone`, `SameFS` in v1; `Scan`, `Read` added in v2) — struct rather than interface umbrella so a sub-interface can be `nil` for backends that don't support a capability.

`ClientPool` already owns instance-ID→live-resource with health/backoff/error-store; the Phase 3 FS resolver reuses the same machinery so "agent unreachable" surfaces in the same banner as "qbit unreachable." Per-instance `*http.Client` cached with shared `Transport{MaxIdleConnsPerHost: 32}`.

Existing tests at `pkg/hardlinktree/create_test.go` and friends call `Create(plan)` directly — Phase 1 updates the signatures (no shim per `AGENTS.md` line 75) and tightens `0644` perms to `0o600` while there.

## 6. Wire protocol

HTTP/JSON over TLS, chi router. NDJSON streaming is **not in v1** (no walks). Endpoints debugged with curl.

Min TLS 1.2. qui's client refuses plaintext to any non-loopback URL.

### v1 endpoints

All require bearer + `X-Qui-Instance` header. No anonymous version banner.

| Method | Path | Notes |
| -- | -- | -- |
| GET | `/v1/health` | `{version, capabilities, allowedRoots, kernelOpenat2}` |
| POST | `/v1/fs/stat` | `{path, follow}` → `FileInfo` |
| POST | `/v1/fs/linkinfo` | `{path}` → `{fileID, nlink}` |
| POST | `/v1/fs/mkdir` | `{path, mode}` — `MkdirAll` semantics |
| POST | `/v1/fs/link` | `{src, dst}` — idempotent: success if dst already `os.SameFile(src)`, else `kind:"exists"` |
| POST | `/v1/fs/reflink` | `{src, dst}` — `kind:"unsupported"` if reflinks not available |
| POST | `/v1/fs/remove` | `{path, recursive}` |
| POST | `/v1/fs/samefs` | `{a, b}` → bool |
| POST | `/v1/fs/reflink-support` | `{dir}` → `{supported, reason}` |

v2 will add `/v1/fs/walk` (NDJSON streaming with sentinel termination + cursor resume) and `/v1/fs/read` (range-bounded, rejects non-regular files).

Errors: `{"error": "...", "kind": "notfound|exists|permission|unsupported|pathescape|invalid|unauth"}`. Remote backend re-hydrates into Go error sentinels.

Auth handler order: read `Authorization` (verify `Bearer ` prefix); read `X-Qui-Instance`; base64-decode token; `subtle.ConstantTimeCompare(decoded, ref) != 0`; pairing UUID match. Mismatches return 401 (or 409 for pairing) with same body and timing.

Idempotency: `mkdir` is `MkdirAll`; `link`/`reflink` succeed if dst is already same-file as src; `remove` returns `notfound` for ENOENT (caller decides).

## 7. Path safety

Caller-supplied paths cross a trust boundary. `filepath.Clean` + prefix match against `--allowed-root` is insufficient — `/downloads/evil → /etc` defeats the prefix check.

**Linux (kernel ≥5.6):** `os.Root` (Go 1.24+) — `OpenFile` issues `openat2(RESOLVE_BENEATH)`, atomically rejecting escapes. Doesn't pass `RESOLVE_NO_SYMLINKS` automatically; agent passes `O_NOFOLLOW` per-op on the final component.

**Symlink policy:** v1 disallows symlinks under allowed roots. Operators with tiered storage (`/downloads/movies → /mnt/big`) register each tier as its own `--allowed-root` rather than symlinking. Avoids per-component policy decisions for safe `Remove`/`RemoveAll` semantics.

**Linux <5.6:** `os.Root` falls back to userspace `O_NOFOLLOW` walk with weaker guarantees against concurrent renames. Agent's `/v1/health` reports `kernelOpenat2: false` so qui surfaces it.

**macOS / Windows:** `os.Root` exists but no `RESOLVE_BENEATH`-equivalent. v1 destructive endpoints (`link`, `reflink`, `mkdir`, `remove`) refuse on non-Linux unless operator passes `--allow-non-linux-destructive` (footgun flag, documented as such).

`--allowed-root` defaults to **deny-all** (empty list). All FS endpoints return `kind:"pathescape"` until operator opts in.

## 8. Authentication & transport

### 8.1 Bearer token

- 32 bytes random, `base64.RawURLEncoding`.
- Agent stores in config file (mode 0600) or `--token-file`. **CLI `--token` is forbidden** (visible in `/proc/<pid>/cmdline`).
- qui stores AES-GCM encrypted with AAD bound to instance ID: `gcm.Seal(nonce, nonce, plaintext, fmt.Appendf(nil, "agent_token:v1:instance=%d", id))`. AAD prevents cross-row secret confusion (the dangerous case where an attacker with DB-write swaps the agent token into `password_encrypted` and gets it sent to qbit's login form). It does NOT prevent lockout-DoS from blob-swapping; that's accepted.
- `internal/models/instance.go`'s `encrypt`/`decrypt` gain an `aad []byte` parameter; existing password call sites pass `nil` (Go's GCM treats `nil` and `[]byte{}` identically — legacy ciphertexts decrypt unchanged). `internal/models/arr_instance.go` is out of scope.
- Constant-time compare at the **decoded 32-byte level**, not the base64 string.

### 8.2 TLS posture

Motivating use case is internet-reachable seedboxes — naked `tls_skip_verify` there means MITM steals the token on first connect.

- Agent supports `--tls-cert/--tls-key`; auto-generates Ed25519 self-signed if absent (NotAfter = now+100y; trust anchor is the SPKI pin, not the validity window). Key persists at `--tls-key` mode 0600.
- qui stores `agent_cert_fingerprint` = `base64(sha256(cert.RawSubjectPublicKeyInfo))`. Pin is on SPKI, so renewing the cert with the same key does not invalidate the pin.
- qui's HTTPS client: `tls.Config{InsecureSkipVerify: true, VerifyPeerCertificate: pinVerify}`. Pin compared via `subtle.ConstantTimeCompare`.
- **qui refuses all agent calls — including health probes — until `agent_cert_fingerprint` is populated.** No "fetch fingerprint via TLS handshake" UX (TOFU = configuration trap on public agents).
- Operator obtains fingerprint via OOB (SSH session into agent host, console). Agent provides:
  - One-time print to stderr on first boot.
  - `qui-agent print-fingerprint` subcommand.
  - `qui-agent rotate-cert [--new-key]`.
- `agent_tls_skip_verify` allowed only with `agent_cert_fingerprint` set. Validated at save AND load.

### 8.3 Plaintext fallback

`--insecure-listen` only when bound to `127.0.0.1`/`::1`. qui's client refuses plaintext to non-loopback URLs.

### 8.4 Token rotation

Agent config supports `token` + `previous_token`. Order: (1) operator adds new token to `token`, moves current to `previous_token`, restarts agent; (2) operator updates qui to new token; (3) operator removes `previous_token` on a later restart.

### 8.5 Cert/key rotation

- Same key, new cert: pin unchanged, no qui-side action.
- New key (suspected compromise): `qui-agent rotate-cert --new-key` regenerates. Operator obtains new fingerprint via OOB, updates qui's `agent_cert_fingerprint`. Agent prints both old and new during rotation for audit.

**Footgun warning for docs (Phase 4):** certbot/Let's Encrypt rotates **keys** by default. Operators using LE in front of the agent will break the SPKI pin on every renewal unless they configure key reuse. Document explicitly.

### 8.6 At-rest threats

Token + TLS key are at-rest plaintext on agent host. Disk-image theft = full compromise. No KDF / TPM sealing in v1. Mitigation: rotation per §8.4/§8.5. systemd hardening (§9.3) reduces in-process surface.

## 9. Agent binary

`cmd/qui-agent/`. Single Go binary, multi-arch via `.goreleaser.yml`. Container image. systemd unit under `distrib/systemd/qui-agent.service`.

```
qui-agent --config /etc/qui-agent/config.toml
qui-agent --listen 0.0.0.0:7477 --token-file /etc/qui-agent/token \
          --allowed-root /downloads --allowed-root /data \
          --tls-cert /etc/qui-agent/cert.pem --tls-key /etc/qui-agent/key.pem \
          --state-dir /var/lib/qui-agent
qui-agent print-fingerprint
qui-agent rotate-cert [--new-key]
qui-agent reset-pairing
```

Footprint goal: <15 MB binary, <30 MB RSS at idle.

### 9.1 Audit log

Destructive ops (`link`, `mkdir`, `reflink`, `remove`) log at INFO to stderr/journal: `{ts, peer_ip, op, path_resolved, result, duration_ms}`. Strings emitted with `strconv.Quote` (never raw `%s`) — defends against log injection via paths with newlines/escape sequences. Token never logged; `sha256(token)[:8]` logged once at startup for correlation.

### 9.2 Hardened systemd unit

Shipped with: `DynamicUser`, `ProtectSystem=strict`, `ProtectHome`, `NoNewPrivileges`, `MemoryDenyWriteExecute`, `LimitCORE=0`, `SystemCallFilter=@system-service`, etc. `ReadWritePaths` includes the operator's allowed-roots. `http.Server{ReadTimeout: 10s, WriteTimeout: 0, IdleTimeout: 60s}` — `WriteTimeout: 0` matters in v2 for long walks, harmless in v1.

## 10. Database schema

One SQLite migration + one Postgres migration (per `AGENTS.md` lines 130-131):

```sql
-- SQLite
ALTER TABLE instances ADD COLUMN agent_url TEXT NOT NULL DEFAULT '';
ALTER TABLE instances ADD COLUMN agent_token_encrypted TEXT NOT NULL DEFAULT '';
ALTER TABLE instances ADD COLUMN agent_cert_fingerprint TEXT NOT NULL DEFAULT '';
ALTER TABLE instances ADD COLUMN agent_tls_skip_verify INTEGER NOT NULL DEFAULT 0;
```

(Postgres migration uses `BOOLEAN NOT NULL DEFAULT FALSE` for `agent_tls_skip_verify`; other three columns are identical.)

Empty `agent_url` means the row is in `FSAccessLocal` or `FSAccessNone`; the other agent columns are ignored.

`internal/models/instance.go` updates: new fields + JSON marshaling, `encrypt`/`decrypt` AAD parameter, `FSAccessMode()` getter, cross-field validation (rejects `HasLocalFilesystemAccess && AgentURL != ""`, rejects `agent_tls_skip_verify=1 && agent_cert_fingerprint == ""`) at save and load.

## 11. Failure semantics

| Failure | Behavior |
| -- | -- |
| Agent unreachable / 5xx | `fsops.ErrAgentUnreachable` wraps the underlying transport error. Routes through `cp.errorStore`; same banner as qbit-unreachable. |
| Auth (401/403) | Banner: "agent rejected qui's credentials." |
| Pairing conflict (409) | Banner: "another qui is paired; run `qui-agent reset-pairing` on agent host." |
| Mid-batch link failure | `hardlinktree.Rollback` runs over the wire (best-effort). Rollback failure logs WARN with `{rootDir, orphanPaths, cause}` AND records to `cp.errorStore` so the banner shows "rollback incomplete; manual cleanup needed." |
| Disk full | Same as mid-batch failure. |
| Version skew | Single failure mode: FS features disabled with banner `"agent vX.Y too old, need ≥A.B"`. No separate badge layer. |

## 12. Concurrency: single-qui pairing

qui generates an install UUID on first boot (single row `qui_install`). Every agent request includes `X-Qui-Instance: <install-uuid>`. Agent persists the first authenticated UUID it sees to `<state-dir>/paired.json` (mode 0600). Subsequent requests with a different UUID return 409 even if the token authenticates. `qui-agent reset-pairing` clears the file.

Why: prevents a second qui (e.g. a forgotten test instance) from concurrently mutating the same agent's filesystem state. Token = trust anchor; UUID = pairing anchor.

(HKDF-derived per-qui tokens were considered and rejected — both qui instances would share the operator's input keying material, so HKDF gives no extra protection.)

## 13. Versioning

`/v1/health` reports agent version + capabilities (`reflink`, `kernelOpenat2`, etc.). qui maintains `MinAgentVersion`; mismatch disables FS features with a banner. No separate "incompatible" UI badge — single failure-state surface.

`/v1/...` is the wire prefix. **v2 is additive**, not a wire-protocol break: orphan-scan/dirscan migration adds `/v1/fs/walk` and `/v1/fs/read` endpoints alongside the v1 set, plus `ScanBackend`/`ReadBackend` interfaces alongside the v1 ones. v1 agents will simply 404 the v2 endpoints; qui reads `capabilities` to know what to call.

## 14. Phased PR plan

External contributors can't open PRs directly per `.github/CONTRIBUTING.md` lines 32-34. Path is design issue → maintainer applies patch from fork.

**Phase 1 — `internal/fsops` + Local + crossseed swap.** Single PR, plumbing-only:
- `internal/fsops` package: `FileInfo`, `LinkBackend`, `CloneBackend`, `SameFSBackend`, `Capabilities` struct, `Local` impl using `os.*` directly (no path-safety guard — local FS is the operator's blanket permission).
- `pkg/hardlinktree.Create/Rollback` and `pkg/reflinktree.Create/Rollback` accept `LinkBackend`/`CloneBackend`. If the two `Create` bodies converge after the refactor, extract a shared `createTree(b, plan, linkFn)` helper.
- crossseed call sites at `service.go:11258, 11852` pass `fsops.Local{}` directly — no resolver yet. Avoids hanging indirection if Phase 3 doesn't land in the same release cycle.
- Tests: existing `pkg/hardlinktree/create_test.go` and `reflink_test.go` updated for new signatures (perms tightened to `0o600` per `AGENTS.md` line 104). Behavior-preservation: a test that calls `Create` against `fsops.Local` against a tempdir, snapshots the resulting tree's `(relpath, mode, size, st_ino, st_nlink)` tuples sorted, and diffs against a golden fixture captured from a pre-refactor run.
- Zero behavior change. Mergeable on its own; nothing in this PR depends on the agent existing.

**Phase 2 — Agent binary.** Single PR:
- `cmd/qui-agent/main.go`: chi router, bearer + UUID-pairing auth, path-safety guard (`os.Root` + `O_NOFOLLOW`), allowed-roots loader.
- v1 endpoints: `/v1/health`, `/v1/fs/{stat,linkinfo,mkdir,link,reflink,remove,samefs,reflink-support}`. (No walk/read in v1.)
- Subcommands: `print-fingerprint`, `rotate-cert`, `reset-pairing`.
- `.goreleaser.yml` updates. systemd unit + Dockerfile.
- Tests: `httptest`-driven; symlink-attack corpus (planted symlinks → expect `pathescape`); auth/pairing matrix; legacy-kernel path (`os.Root` userspace fallback) smoke.

**Phase 3 — Remote backend + DB schema + resolver + UI.** Single PR (split if reviewer prefers):
- `fsops.Remote` HTTP client implementing `LinkBackend`, `CloneBackend`, `SameFSBackend`.
- Introduce `pool.GetFSBackend(ctx, instanceID) (fsops.Capabilities, error)` returning `Remote` when `AgentURL != ""`, else `Local`. crossseed call sites swap from passing `fsops.Local{}` directly to obtaining backends from the pool.
- Migration: 1 SQLite + 1 Postgres, four columns.
- `Instance` model: new fields, AAD-bound encrypt/decrypt, `FSAccessMode`, cross-field validation at save and load.
- UI: instance edit form (URL, token, fingerprint, skip-verify gated on fingerprint), "test connection" button, banner copy for the §11 failure modes. `internal/web/swagger` updates → `make test-openapi` per `AGENTS.md` line 132.
- Tests: in-process e2e via `httptest.NewTLSServer` wrapping the agent's chi router with a self-signed cert + fingerprint pin, exercising every endpoint plus auth/pairing failure paths.

**Phase 4 — Documentation.** `documentation/docs/` page: install, allowed-roots, OOB fingerprint copy, systemd example, NAT/tunnel guidance, certbot key-rotation footgun, troubleshooting, rotation procedures, uninstall.

**v2 fast-follow PRs (separate, after v1 ships):**
- v2.1: Add `ScanBackend` interface + `Local` impl + refactor orphanscan call sites.
- v2.2: Add `ReadBackend` interface + `Local` impl + refactor dirscan call sites.
- v2.3: Agent gains `/v1/fs/walk` (NDJSON streaming, terminal sentinel, cursor resume, gzip Content-Encoding, default 10-min wall-clock) and `/v1/fs/read` (range-bounded, non-regular files rejected).
- v2.4: `Remote` backend implements the new interfaces.

The v2 plan is intentionally not phased here in detail — the doc's job is to commit to the architecture supporting it. Operators following this design should not need to keep network mounts around once v2 ships.

Conventional commits per `CONTRIBUTING.md` line 38. No AI attribution per `AGENTS.md` lines 110-114.

## 15. Open questions for maintainers

Three. The rest the contributor will decide and run with unless flagged.

1. **Connection direction.** qui → agent (chosen) vs agent → qui (call-home). qui→agent matches every existing qui integration and keeps the protocol stateless; call-home would only need qui to be reachable but adds persistent bidi channel + multiplexing + reconnect state. Recommend qui→agent for v1; document tunneling.
2. **Agent in this repo or sibling.** Same repo couples versions, simplifies CI, lets `fsops.Local` be shared by import. Sibling repo lets the agent move independently. Recommend same repo, separate `cmd/`, agent excluded from default `make build`.
3. **`MinAgentVersion` enforcement.** Recommend hard-block on mismatch with the §11 banner — silent wire skew between qui and a stale agent is a support-burden generator. Confirm vs. warn-and-allow?

## 16. Out of scope (v1)

- Auto-upgrade, mDNS discovery, multi-agent failover, call-home, path-mapping translation, rate limiting on agent, multi-qui coordination, TPM-sealed token storage, HKDF per-qui tokens, per-request nonce/replay protection.
- Walk and Read endpoints + `ScanBackend`/`ReadBackend` interfaces — explicitly v2, not "never."
- Per-pairing concurrent-request caps — `AGENTS.md` line 78 ("skip paranoid defensive programming") applies; defer until needed.
- `/metrics` Prometheus endpoint — defer.
- `--readonly` agent mode — defer until someone asks.

## 17. Estimated effort

| Phase | Days | Notes |
| -- | -- | -- |
| 1 fsops + Local + crossseed swap | 3-4 | plumbing, tests |
| 2 agent binary + v1 endpoints | 4-5 | new code, security-sensitive |
| 3 Remote + schema + UI | 5-7 | heaviest |
| 4 docs | 1-2 |  |

**v1 total: ~13–18 working days (3–4 calendar weeks).** v2 fast-follow: ~5–7 days for orphan-scan + dirscan migration + walk/read endpoints (most of the cost is walk streaming infrastructure, not the consumer migration).
