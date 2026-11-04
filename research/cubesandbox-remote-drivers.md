# CubeSandbox and remote sandbox driver shapes

Research for [#12](https://github.com/jonaswre-labs/t3code/issues/12) (map: [#10](https://github.com/jonaswre-labs/t3code/issues/10)).

The question: what CubeSandbox is, and what remote drivers like it (and E2B-style APIs) need that a same-machine driver like BoxLite does not. This page lists constraints. It does not design the remote driver.

Sources were read on 2026-11-04. CubeSandbox claims come from the repo at commit [`39491ad`](https://github.com/TencentCloud/CubeSandbox/tree/39491adc42065d746e8046f1b85643dd12d89e30) (v0.7.x line). E2B claims come from `docs.e2b.dev` and `e2b.dev/pricing`. Each claim is tagged:

- **[V]** verified in a primary source (linked).
- **[I]** my inference from verified facts. Treat it as unconfirmed.


## 1. What CubeSandbox is

| Aspect | Finding |
|---|---|
| Owner / license | Tencent Cloud's IaaS Frontier Technology team. Apache-2.0. **[V]** [README](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/README.md), [about-us](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/about-us.md) |
| Deployment model | Open-source software that you run yourself. It runs on a single node, multi-node clusters, Kubernetes (preview), or Tencent Cloud via Terraform. **[V]** [introduction](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/introduction.md), [README](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/README.md). I found no hosted CubeSandbox service with published pricing. **[I]**, based on an absence in the docs and a web search |
| Host requirements | The README says "x86_64 Linux with KVM". v0.5 added native ARM64. PVM mode deploys on ordinary cloud VMs without nested virtualization. **[V]** [README](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/README.md), [v0.5.0](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/changelog/v0.5.0.md), [pvm-deploy](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/pvm-deploy.md) |
| Isolation tech | Each sandbox is a KVM MicroVM with its own kernel, run by CubeHypervisor (RustVMM) and integrated through a containerd Shim v2 (CubeShim). Networking is eBPF (CubeVS). Egress goes through an L7 MITM proxy (CubeEgress, OpenResty). Storage is XFS-reflink CoW (CubeCoW). **[V]** [architecture overview](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/architecture/overview.md) |
| Control plane | CubeAPI is a Rust/Axum REST API that is E2B-compatible. CubeMaster is the Go scheduler. Redis holds all state. CubeProxy (OpenResty) routes requests into sandboxes. cube-lifecycle-manager handles auto-pause and auto-resume. **[V]** [architecture overview](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/architecture/overview.md) |
| In-guest agent | `envd` on port 49983, the same daemon E2B uses. Commands, files, and PTY all go through it. A template needs envd for those APIs. **[V]** [templates](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/templates.md), [bring-your-own-image](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/tutorials/bring-your-own-image.md) |
| Images | An OCI image is built into a *template* (rootfs plus memory snapshot) inside the cluster. Sandboxes boot by CoW-cloning the template. **[V]** [architecture overview § Storage](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/architecture/overview.md) |
| SDKs | Python `cubesandbox`, Node/TS `@cubesandbox/sdk` (Node ≥ 18, ESM+CJS), and Go. **[V]** [Node SDK](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/tutorials/sdk/nodejs.md), [`sdk/`](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/sdk) |
| Auth | Off by default ("allows all requests"). When `--auth-callback-url` is set, CubeAPI forwards the Bearer/X-API-Key credential plus path and method to your callback, and HTTP 200 allows the request. There are no built-in users, teams, or quotas. **[V]** [authentication](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/authentication.md) |

### E2B compatibility

- **[V]** The stock E2B SDKs work by setting `E2B_API_URL` to CubeAPI and `E2B_API_KEY` to any placeholder. [quickstart](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/quickstart.md)
- **[V]** The compatibility is incomplete. The roadmap lists "close the remaining gaps … full drop-in compatibility" as future work. [roadmap](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/roadmap.md)
- **[V]** Known divergences:
  - Cube timeouts are in seconds, while E2B JS uses `timeoutMs`. [lifecycle](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/lifecycle.md)
  - The official E2B Python SDK cannot be used for Volumes ("hardcoded to the e2b.cloud backend"). [volume-plugin](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/volume-plugin.md)
  - Wildcard DNS for `<port>-<id>.<domain>` defaults to `cube.app`. Remote clients often need `CUBE_PROXY_NODE_IP`, which only the Cube SDKs honor. [https-and-domain](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/https-and-domain.md), [Node SDK README](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/sdk/node/README.md)
  - Several Cube-only extensions exist: host mounts, read-only volume attachments, and `restartPolicy`.
- **[I]** A driver written against "the E2B API" would reach both E2B Cloud and Cube for create, connect, commands, files, and PTY. Cube-specific features such as host mounts and the proxy-IP override need Cube-aware code or the Cube SDK.

### Lifecycle and identity (both services)

- **[V] Cube states:** `running`, `pausing`, `paused`, `resuming`, `terminated`, `unknown`. [lifecycle](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/lifecycle.md)
  - `unknown` covers three cases: still creating, exited, or unreadable. Only `POST /sandboxes/{id}/connect` tells them apart: it returns 409 when the sandbox is permanently gone.
  - A killed sandbox returns 410 on later requests.
  - An auto-resume failure returns 503 + `Retry-After`.
- **[V] Cube timeouts:**
  - `timeout` is an *idle* timeout. `on_timeout` is `kill` (the default) or `pause`.
  - With `auto_resume`, any request to a paused sandbox wakes it.
  - A manual `pause()` does not cancel a later idle kill.
  - The shipped cluster default has no idle TTL (`default_timeout_insec: -1`). [lifecycle](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/lifecycle.md)
- **[V] Pause fidelity:**
  - Memory, processes, and the filesystem survive a pause.
  - "Outbound sockets the sandbox itself opened are dropped on pause."
  - E2B: "connected clients are disconnected" while paused, and they must reconnect. [Cube lifecycle](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/lifecycle.md), [E2B persistence](https://docs.e2b.dev/sandbox/persistence)
- **[V] Identity:**
  - The *service* assigns the sandbox ID (for example `iiny0783cype8gmoawzmx-ce30bc46`). The client cannot choose it.
  - Both services accept arbitrary `metadata` key/value pairs at create time and can filter list results by metadata. E2B also filters by state, template, and `startedAfter`, with pages of at most 100 results.
  - Reconnecting by ID uses `Sandbox.connect(id)`, which also resumes a paused sandbox.
  - Sources: [Cube lifecycle](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/lifecycle.md), [Cube openapi.yml `metadata` query](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/openapi.yml), [E2B list](https://docs.e2b.dev/sandbox/list), [E2B connect](https://docs.e2b.dev/sandbox/connect)
- **[V] Restarts:** Cube `restartPolicy` restarts keep the disk, network, and ID, but not memory. [lifecycle § Restart after exit](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/lifecycle.md)

## 2. Workspace: can it be mounted?

| Option | Facts |
|---|---|
| Cube host mount | **[V]** Bind-mounts a directory *on the Cubelet node* into the guest, read-write or read-only. The `host-mount` metadata key is a Cube extension. [persistent-storage](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/persistent-storage.md) <br>**[V]** Paths must sit under an operator allowlist (default `/data/shared/`). Ownership and mode are preserved, and the sandbox user is UID 1000, so a UID mismatch causes `Permission denied`. <br>**[V]** "`hostPath` refers to the filesystem of the sandbox host node … not the machine where your SDK script executes." <br>**[V]** A host-mounted sandbox is pinned to its origin node for pause, resume, and restore. Rollback does not roll back the mounted data. |
| Cube volumes | **[V]** E2B-compatible `/volumes` API backed by operator-installed plugins (COS, S3 via s3fs, JuiceFS, NFS-style). A volume's lifecycle is independent of any sandbox, and one volume can be shared by several sandboxes. Requires Cube ≥ 0.6. [volume-plugin](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/volume-plugin.md) <br>**[V]** Snapshots keep external-reference semantics: restore and rollback show the volume's *current* data. |
| Cube's own Claude Code integration | **[V]** Keeps Claude Code on the host and redirects only `Bash` into the VM through a `PreToolUse` hook. The project is mounted *read-only*, and only when it is co-located with the Cubelet. Read/Write/Edit stay on the host. "The hook does not upload or synchronize a local project to a remote deployment." Output is buffered, not streamed. [integrations/claude-code](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/integrations/claude-code.md) |
| E2B Cloud | **[V]** There is no host mount. The workspace gets in by `git clone` inside the sandbox (the documented Claude Code pattern) or by `files.write` (single, batch, or streamed uploads, plus pre-signed upload URLs). [E2B claude-code](https://docs.e2b.dev/agents/claude-code.md), [E2B upload](https://docs.e2b.dev/filesystem/upload) <br>**[V]** Volumes are in "private beta". [E2B volumes](https://docs.e2b.dev/volumes) <br>**[V]** The `sandbox.git` helpers are deprecated in favor of running `git` via `commands.run`. [git-deprecation](https://docs.e2b.dev/migration/git-deprecation.md) |
| E2B's OpenCode workspace plugin (prior art) | **[V]** OpenCode stays local. A remote OpenCode server runs in the sandbox on a *copy* of the project, which is uploaded in batches of 100 files without `.git`, `node_modules`, `dist`, and so on. The project sits at the same absolute path on both sides. Changes come back as a patch applied to the local git repo, and commits made in the sandbox do not come back. Conflict handling is not documented. [workspace-plugin](https://docs.e2b.dev/agents/opencode/workspace-plugin.md) |

**[I]** Whether a driver can mount the workspace depends on the *deployment*, not on the driver type:

- A Cube cluster on the same machine as T3, with the worktree under an allowlisted prefix and matching UIDs, can mount it read-write, just like BoxLite.
- A remote Cube node cannot, unless T3's host and the Cube nodes share storage at the same path (NFS or JuiceFS).
- E2B Cloud never can.

## 3. Reaching the T3 MCP endpoint from the guest

- **[V] T3 today:**
  - `McpSessionRegistry` builds the endpoint as `http://<bind address>:<port>/mcp`, and as `127.0.0.1` when the server binds the unspecified address.
  - Auth is a per-session bearer token with a 24 h liveness window.
  - `/mcp` is outside environment auth.
  - [`apps/server/src/mcp/McpSessionRegistry.ts`](../apps/server/src/mcp/McpSessionRegistry.ts)
- **[V] Cube default egress:**
  - Internet is allowed by default.
  - CubeVS still denies `10/8`, `127/8`, `169.254/16`, `172.16/12`, and `192.168/16` "to prevent workloads … reach[ing] Cube infrastructure".
  - Allow rules take precedence over deny rules, so `allow_out` can open an explicit hole.
  - [network-policy](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/network-policy.md)
- **[V] E2B egress:**
  - Open by default, with `allowOut`/`denyOut` (IP, CIDR, or domain) and `updateNetwork` on a running sandbox.
  - Domain filtering only works for HTTP on port 80 and TLS on port 443.
  - [E2B internet-access](https://docs.e2b.dev/network/internet-access)
- **[V] Inbound to the guest:**
  - Cube routes HTTP through CubeProxy, by host (`<port>-<id>.<domain>`) or by path (`/sandbox/<id>/<port>/…`). WebSocket upgrades are supported. [https-and-domain](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/https-and-domain.md)
  - E2B gives `https://<port>-<id>.e2b.app` for "any service listening on a sandbox port", with TLS terminated at its proxy. [E2B public-url](https://docs.e2b.dev/network/public-url)
  - Both services leave the URL public by default and support `allowPublicTraffic=false` with a per-sandbox `e2b-traffic-access-token` header. [restrict-public-access](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/restrict-public-access.md)
- **[I]** On Cube, cross-host routing appears to need a per-port host-port mapping (`sandbox_backend.lua` errors on "misses host port mapping"). That suggests only ports exposed by the template are routable from another node. Not confirmed in the docs. [CubeProxy/lua/sandbox_backend.lua](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/CubeProxy/lua/sandbox_backend.lua)
- **[V]** Both services have an MCP *gateway* concept for MCP servers running *inside* the guest. It points the other way from T3's need. [Cube mcp-gateway](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/mcp-gateway.md), [E2B Claude Code MCP](https://docs.e2b.dev/agents/claude-code.md)
- **[I]** A loopback URL never works from any guest, including BoxLite. A remote guest also cannot reach a LAN or private address on Cube without an explicit allow hole, and it cannot reach a NAT'd T3 host from E2B Cloud at all. That leaves two options:
  - T3's MCP endpoint is reachable from the guest's network (public, a tunnel, or the tailnet), or
  - the guest-to-T3 path is carried over a T3-initiated channel, such as a T3 → CubeProxy/envd connection that a guest-side shim bridges to a local port.
- **[I]** Pausing drops guest-opened sockets, so a long-lived MCP streaming connection will be cut whenever the guest idles out.
- **[I]** Tailscale's `100.64.0.0/10` is not in Cube's default deny list. A guest on a tailnet-attached node might reach a tailnet T3. Untested.

## 4. Running the agent process remotely

- **[V]** `AgentScope.wrap()` returns a rewritten local `{ command, args }` (plus an optional systemd unit), which T3 then spawns and talks to over stdio. [`packages/shared/src/AgentScope.ts`](../packages/shared/src/AgentScope.ts)
- **[V]** In both services, process I/O goes over envd's HTTP/Connect process API:
  - streaming stdout through `onStdout`
  - `sendStdin(pid, data)`
  - PTY `create`, `resize`, and `connect(pid)` to reattach, which survives a client disconnect
  - [Cube Node SDK README § PTY](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/sdk/node/README.md), [E2B claude-code](https://docs.e2b.dev/agents/claude-code.md)
- **[I]** For a remote driver there is no local process. Stdio JSON-RPC providers (Codex app-server, Claude) and terminals have two options:
  - run through a local proxy command that bridges to envd, which keeps the `wrap()` shape, or
  - use a driver-level "exec with streams" interface.
- **[I]** Either way, the host loses the systemd scope, PID-based kill, and OOM signals that `AgentScope` provides today.
- **[I]** Each stdin write becomes an HTTP request, which adds per-message latency.

## 5. Tenancy, pricing, limits

| | CubeSandbox (self-hosted) | E2B Cloud |
|---|---|---|
| Cost | Your own infrastructure. **[V]** No published hosted price found. **[I]** | **[V]** Per second while running: vCPU $0.000014/s, RAM $0.0000045/GiB-s. Hobby is $0. Pro is $150/mo + usage. Enterprise starts at $3,000/mo. Paused sandboxes are not billed for compute. [pricing](https://e2b.dev/pricing), [billing](https://docs.e2b.dev/billing.md) |
| Max runtime | No hard wall-clock ceiling. The idle TTL is the operator's choice. **[V]** [lifecycle](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/lifecycle.md) | **[V]** 1 h (Hobby) or 24 h (Pro) of continuous runtime. Pause/resume resets the window. Paused sandboxes are kept "indefinitely". [sandbox](https://docs.e2b.dev/sandbox), [persistence](https://docs.e2b.dev/sandbox/persistence) |
| Concurrency | Node capacity. **[V]** Paused sandboxes keep their full scheduler quota by default (`paused_resource_release_ratio = 0`). With release enabled, resume can be rejected with 409. Snapshots consume disk. [lifecycle § Paused Resource Release](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/lifecycle.md) | **[V]** Hobby: 20 running. Pro: 100, up to 1,100 with add-ons. Paused sandboxes do not count. [billing](https://docs.e2b.dev/billing.md), [paused FAQ](https://docs.e2b.dev/faq/paused-sandboxes-concurrency.md) |
| Rate limits | None documented. | **[V]** Hobby: 1 create/s and 10 API req/s per endpoint. Pro: 5 create/s and 20 req/s. Exceeding a limit returns 429 + `Retry-After`. [billing](https://docs.e2b.dev/billing.md) |
| Sizes | Template and request define them. | **[V]** 1–8 vCPU and 1–8 GiB RAM, default 2 vCPU / 4 GiB. Disk is 10 or 20 GiB. CPU only. [pricing](https://e2b.dev/pricing) |
| Tenancy | **[V]** One shared credential space through an optional auth callback. | **[V]** API key per team or project. Limits are shared across all keys in a project. |
| Self-host | The whole thing. | **[V]** "E2B Embed" is single-machine and Apache-2.0. BYOC (AWS, GCP, Azure) is Enterprise-only and "is not self-hosting". [byoc](https://docs.e2b.dev/byoc.md) |

## 6. Latency

- **[V] Cube**, on bare metal, from Cube's own benchmarks:
  - create ≈ 48–60 ms serial, p95 ~90 ms at 50 concurrent creates (README) or 508 ms (benchmark page, different load)
  - resume ≈ 42 ms
  - pause ≈ 558 ms at 2 GiB (full memory copy today)
  - [README](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/README.md), [performance-benchmark](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/performance-benchmark.md)
- **[V] E2B:** pause ≈ 4 s per GiB of RAM, resume ≈ 1 s. [persistence](https://docs.e2b.dev/sandbox/persistence)
- **[I]** For a remote driver, boot time is small next to (a) the network round trip on every exec, stdin write, and file operation, and (b) moving the workspace in and out. A multi-GB repo or `node_modules` upload costs minutes, while a boot costs milliseconds.

## 7. Other relevant facts

- **[V] Egress policy and credentials:** both services support `allow_internet_access` plus allow/deny lists. Cube adds L7 rules with header-based credential injection, so "secrets never enter the sandbox". [security-proxy](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/security-proxy.md)
- **[V] Snapshots:** both offer VM-level snapshot, rollback, and fork. Cube: [snapshot-rollback-clone](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/snapshot-rollback-clone.md). E2B: [snapshots](https://docs.e2b.dev/sandbox/snapshots.md).
- **[I] Checkpoints:** VM-level snapshots are *not* T3 checkpoints, which are host git refs. With an unmountable workspace, host-side git, checkpoint, and file APIs have nothing to read until changes are synced back.
- **[V] Telemetry:** Cube exposes per-sandbox CPU and memory through an unauthenticated Cubelet Prometheus endpoint (`:9998`) on the operator network, not through CubeAPI. [resource-metrics](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/resource-metrics.md)
- **[V] Images:** a guest image must be built as a template *inside each cluster or account*, and the template ID is per cluster. It must include envd. [templates](https://github.com/TencentCloud/CubeSandbox/blob/39491adc42065d746e8046f1b85643dd12d89e30/docs/guide/templates.md)

## Constraints the sandbox driver interface must not rule out

All of these are **[I]**, derived from the facts above.

1. **Unmountable workspaces.** Mounting is a capability that each driver and deployment declares, not something the interface assumes. The interface must allow a workspace that is *copied in* (upload or clone) and *copied back* (diff, patch, or sync). It must also tolerate that, while a turn runs, the authoritative files live in the guest. That affects when host-side git, checkpoints, and file APIs can run.
2. **No local process.** Agent, terminal, and setup-script execution must work when there is no host PID: stdio is streamed over a network API, and kill, wait, and exit code come from the driver. The interface must not require a systemd scope or a host OOM signal.
3. **Service-assigned, persisted identity.** The driver hands back an opaque ID that T3 must store per thread and use to reconnect after a T3 restart. Tagging guests with metadata (environment ID and thread ID) allows orphan discovery and avoids collisions when several T3 environments share one account or cluster.
4. **The guest can vanish or change state on its own.** Idle timeouts, 1 h/24 h ceilings, auto-pause, kill-on-timeout, and 409/410/503 responses must all map to a visible sandbox status and a "recreate or fail visibly" path, with no host fallback.
5. **Pause breaks connections.** Any long-lived stream (agent stdio, MCP, terminals, forwarded ports) must survive a reconnect or a restart after resume.
6. **Guest-to-T3 reachability is not given.** The MCP endpoint (and any other guest-to-host callback) cannot assume loopback or LAN access. The interface must let a driver supply the URL the guest should use, or carry the traffic over a T3-initiated channel.
7. **Host-to-guest ports go through a proxy.** Preview port forwarding may be an HTTPS URL per port (sometimes only template-declared ports) with an access token, not a local TCP forward.
8. **Per-driver credentials and quotas.** A sandbox instance needs driver credentials (an API key or a URL plus token), and the driver may surface 429s, concurrency caps, and per-second cost. Status polling across many threads must respect rate limits.
9. **Images are per-provider artifacts.** The "guest image" is a template ID built inside the remote service and must contain envd. It is not a local OCI image the driver pulls.
10. **Latency is in I/O, not boot.** The interface should not make every file operation or command a separate round trip to the guest when a batched or streamed alternative exists.
