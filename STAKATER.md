# Stakater fork notes — function-kcl native memory leak

This fork addresses the function-kcl native (off-heap) memory leak that caused
pods on us-2 to climb to their 8Gi limit and get OOMKilled (exit 137), dropping
in-flight reconciles (`connection reset by peer` / `DeadlineExceeded`) and
stalling heavy composites (org XRs, XMesh).

## Root cause (verified)

Every reconcile recompiles the whole KCL module: `RunFunction` → krm-kcl
`kio.Pipeline` → kpm → kcl-go `ExecProgram` → native `libkcl.so`. The native KCL
runtime accumulates off-heap memory **per compile**, and the growth is
**process-global** — it is *not* tied to the service handle (recreating the
handle via `kcl_service_delete`/`new` reclaims nothing), and `GOMEMLIMIT`/`GOGC`
don't bound it (it's not Go heap).

A "compile once, execute many" cache (`BuildProgram`/`ExecArtifact`) would be the
ideal fix, but it is **not reachable** on the pinned KCL stack: libkcl 0.12.3's
native C-ABI dispatcher only routes those methods over the RPC transport, not the
cgo/purego path this function uses. So the leak can only be bounded at a process
boundary. The two mechanisms below do that.

## 1. Graceful self-recycle (`recycle.go`)

A watchdog samples process RSS (which includes the native off-heap memory) plus
optional reconcile-count / lifetime limits. When a limit is crossed it stops
accepting new reconciles, drains in-flight ones, and exits 0 so Kubernetes
restarts the pod **between** renders instead of OOMKilling it **during** one.
With multiple replicas this is a rolling, self-healing recycle and removes the
need for the manual `kubectl rollout restart` stopgap.

| Env var | Default | Meaning |
| --- | --- | --- |
| `FUNCTION_KCL_MAX_RSS_BYTES` | (derived) | Recycle when RSS reaches this. Accepts a plain byte count or `Ki`/`Mi`/`Gi` suffix. |
| `FUNCTION_KCL_MAX_RSS_RATIO` | `0.85` | When `MAX_RSS_BYTES` is unset, recycle at this fraction of the detected cgroup memory limit. |
| `FUNCTION_KCL_MAX_RECONCILES` | `0` (off) | Recycle after this many RunFunction calls. |
| `FUNCTION_KCL_MAX_LIFETIME` | `0` (off) | Recycle after this process uptime (e.g. `6h`). |
| `FUNCTION_KCL_RECYCLE_CHECK_INTERVAL` | `30s` | Watchdog sampling interval. |
| `FUNCTION_KCL_RECYCLE_DRAIN_TIMEOUT` | `15s` | Max wait for in-flight reconciles before forcing the restart. |

With the us-2 8Gi limit and defaults, the pod recycles at ~6.8Gi — before the
kubelet OOMKills at 8Gi. If no cgroup limit is detected and no env trigger is
set, the watchdog never fires (behaviour identical to upstream).

## 2. Render cache (`rendercache.go`) — optional

A composition function is deterministic over its input. In steady state most
reconciles are no-op re-syncs with byte-identical input. The render cache
memoises the KCL pipeline output keyed on the exact serialized input (source +
dependencies + all params + config), so an identical reconcile returns the cached
output **without invoking the KCL native runtime** — skipping the recompile, its
CPU cost, and a leak increment. It does not help when inputs genuinely change
every reconcile (e.g. a composite actively churning during a rollout); the
recycler is the backstop for that.

| Env var | Default | Meaning |
| --- | --- | --- |
| `FUNCTION_KCL_RENDER_CACHE_SIZE` | `0` (off) | Max cached entries (LRU). Set > 0 to enable. |
| `FUNCTION_KCL_RENDER_CACHE_TTL` | `0` (none) | Optional per-entry TTL (e.g. `10m`). |

Enable it only where compositions are deterministic (the normal case). Start with
e.g. `FUNCTION_KCL_RENDER_CACHE_SIZE=512`.

## Deploying

Set the env vars via the existing us-2 env-override (`crossplane-function-kcl.yaml`,
`extraEnv`), the same place the CPU request / memory limit stopgap lives.
