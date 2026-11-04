# BoxLite capabilities for a thread sandbox

Answers [#11](https://github.com/jonaswre-labs/t3code/issues/11) for the sandbox map ([#10](https://github.com/jonaswre-labs/t3code/issues/10)).

- **Sources read:** BoxLite at commit [`3f8e7d2`](https://github.com/boxlite-ai/boxlite/tree/3f8e7d2dedd338895399f45ff597145a53eca9b3) (2026-11-04, a few days after v0.11.0), its docs, Node SDK source, issues and releases. Also BoxLite's libkrun fork at the pinned submodule commit [`e12b9b3`](https://github.com/boxlite-ai/libkrun/tree/e12b9b3780ffa8df9f3e1797b217d13453479167).
- **Live probe:** I ran `@boxlite-ai/boxlite@0.11.0` from npm on this Linux x86_64 KVM host (kernel 6.8, unprivileged user in the `kvm` group, Node 24) with `alpine:latest` and `python:3-alpine` images.
- **Labels:** **[V]** means verified in source, docs or the probe. **[I]** means my inference.


## Answer in brief

BoxLite fits the plan's shape. It is an embeddable library with no daemon, ships a first-party Node SDK (napi-rs), and runs one micro-VM per box. Boxes are persistent and can be stopped and restarted, and they share a host directory read-write over virtio-fs. It has exec with a PTY, signals, env, cwd and user, a hostname for reaching host loopback, egress allowlists, and OCI images.

The risks are concentrated in five places:

1. **Hosts:** macOS on Intel and native Windows are not supported.
2. **Volumes:** a hard limit of two directory volumes on Linux x86_64.
3. **Guest services:** a service bound to the guest's `127.0.0.1` cannot be forwarded to the host.
4. **Telemetry:** OOM and metrics reporting is weak.
5. **Churn:** the project is moving fast and is about to replace its VMM.

## Threats to the plan

| # | Threat | Status | Impact on the map |
|---|---|---|---|
| T1 | **No macOS on Intel, no native Windows.** Supported hosts are macOS on Apple Silicon, Linux x86_64 and arm64, and WSL2. The npm package ships only `darwin-arm64`, `linux-x64-gnu` and `linux-arm64-gnu`, with no musl build. Native Windows is planned only as milestone M10 of the new VMM. Intel Macs are explicitly out of scope for that VMM, despite issue [#356](https://github.com/boxlite-ai/boxlite/issues/356) promising them. | [V] | Sandboxes must show as disabled with a reason on Windows desktop, Intel Macs and musl Linux. The "Windows hosts" open item resolves to "WSL2 only, or nothing". |
| T2 | **At most 2 directory volumes on Linux x86_64.** With 3 volumes the VM fails to start (`RegisterNetDevice(IrqsExhausted)`, issue [#935](https://github.com/boxlite-ai/boxlite/issues/935), still open). I reproduced this on 0.11.0: 2 volumes work, 3 give "The VM failed to start". The fix is planned for the new VMM, with one virtio-fs device for all shares ([#2274](https://github.com/boxlite-ai/boxlite/issues/2274), milestone M3). | [V] | A worktree needs the worktree directory plus the main repo's `.git`, which is already 2 of the 2 slots. Credential and cache mounts have to be copied in rather than mounted. |
| T3 | **Guest services bound to `127.0.0.1` can't be forwarded.** Published ports and tunnels both dial the guest IP `192.168.127.2`. In the probe, a service on `0.0.0.0` returned 200 and the same service on `127.0.0.1` returned `ECONNRESET`, through both paths. | [V] | The preview browser breaks for dev servers that default to `localhost` (Vite does). You need a relay inside the guest (such as socat) or a forced `--host 0.0.0.0`. |
| T4 | **Host loopback is reachable by default.** `host.boxlite.internal` (192.168.127.254) is NATed to the host's `127.0.0.1`, on any port, whenever `allow_net` is empty. An allowlist can only add rules. There is no deny rule, so you can't keep open internet while blocking the host. | [V] | This is the route the plan needs, and it works. But with egress open by default, the T3 server port (and every other dev server on the host) is reachable from the agent. Auth on those services is then the only barrier. |
| T5 | **Mount ownership is fixed to two guest uids.** The virtio-fs server runs as the host user. A guest process can write only as guest uid 0 or as the host user's uid; any other uid gets `EPERM`. `chown` fails with `EINVAL`. BoxLite idmaps the host owner to the container user, so the default root user works. | [V] | Guest images must run the agent as root, or as a user whose uid equals the host owner's. A `USER node` (uid 1000) image fails when the host uid isn't 1000. |
| T6 | **Host-side writes can look stale or truncated in the guest for up to 5 s.** The guest caches attributes with a 5 s timeout, and the cache policy is `Auto`. In the probe, the guest read a host-rewritten file truncated to its old length. Guest-side writes are visible on the host immediately, because writeback caching is off. | [V] | This matters when T3's host-side git restores a checkpoint, checks out, or applies a diff while the guest is running. The guest may see torn files for a few seconds. Stopping the guest, or waiting, works around it. |
| T7 | **OOM and metrics are weak.** The memory limit is the VM's RAM. The guest kernel OOM-killer kills a process inside the VM: the exec returns `137`, the box stays `running`, and no event fires. `metrics().memoryBytes` reported about 2 MB for a busy box. The issue tracker admits a shim crash isn't detected until the next call ([#586](https://github.com/boxlite-ai/boxlite/issues/586)). | [V] | The "Resource telemetry" item needs its own detection, for example reading guest `dmesg` or `/proc/pressure`, or treating exit 137 as a heuristic. |
| T8 | **One runtime per `BOXLITE_HOME`.** BoxLite takes an exclusive `flock` on `$BOXLITE_HOME/.lock`. | [V] | The T3 server must own a private home directory. The `boxlite` CLI can't inspect it while the server runs, and each dev worktree needs its own home. |
| T9 | **Execs can't be reattached locally.** On the local backend, `attach()` supports only the main command, and the Node SDK doesn't expose `attach` at all. There is no way to list running execs ([#2165](https://github.com/boxlite-ai/boxlite/issues/2165), open). | [V] | Provider processes inside a guest can't survive a T3 server restart. Either the box stops with the server (the default), or with `detach` it keeps orphaned execs nobody can reach. |
| T10 | **Fast churn.** Version 0.x, about 30 releases between January and October 2026, a repo created in December 2025, and 424 open issues. libkrun is being replaced by an in-house VMM ([#1512](https://github.com/boxlite-ai/boxlite/issues/1512)). When it lands (milestone M5), macOS 12–14 support ends and macOS 15 becomes the minimum. | [V] | Pin an exact version and wrap BoxLite behind the sandbox driver interface. Expect breaking changes. |

## Embedding and process model

- **[V] There are three interfaces:**
  - an embedded library with SDKs for Python, Node, Go, Rust and C;
  - the `boxlite` CLI;
  - `boxlite serve`, a REST server on 127.0.0.1:8100.

  The README describes the library as "Daemonless: embed as a library — no root, no background service" ([README](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/README.md)).
- **[V] The Node SDK** is `@boxlite-ai/boxlite`, built with napi-rs over the Rust core, for Node 18 and later. Its targets are `aarch64-apple-darwin`, `x86_64-unknown-linux-gnu` and `aarch64-unknown-linux-gnu` ([sdks/node/package.json](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/sdks/node/package.json)). The Linux x64 native package is about 107 MB unpacked, and the runtime binaries are embedded in it. They're extracted to a cache such as `~/.local/share/boxlite/runtimes/v{VERSION}-…`, and `BOXLITE_RUNTIME_DIR` overrides the location ([src/boxlite/src/util/binary_finder.rs:1-11](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/src/boxlite/src/util/binary_finder.rs#L1-L11)).
- **[V] Process model:**
  - The runtime runs in the host process.
  - Each box gets its own `boxlite-shim` subprocess. It runs inside a jailer (bubblewrap and seccomp on Linux, `sandbox-exec` on macOS) and hosts the VM through libkrun, plus gvproxy for networking in the same process.
  - The guest agent talks gRPC over vsock ([docs/concepts/README.md](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/docs/concepts/README.md)).
  - With `detach: false` (the default), the shim watches a parent pipe and shuts down when the host process dies ([src/shim/src/main.rs:222-228](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/src/shim/src/main.rs#L222-L228)).
- **[V] Node API shape:**
  - `new JsBoxlite({ homeDir })`, then `create(opts, name)`, `get`, `getInfo`, `listInfo` and `remove(id, force)`.
  - On a box: `exec`, `start`, `stop`, `info`, `metrics`, `copyIn`, `copyOut`, `cloneBox`, `export` and `snapshot`, plus `network` and `ssh` handles.

  Sources: [docs/reference/nodejs/README.md](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/docs/reference/nodejs/README.md) and [sdks/node/src/box_handle.rs](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/sdks/node/src/box_handle.rs).
- **[V] The box "agents" API is a stub.** Every operation returns Unsupported ([docs/reference/agents.md](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/docs/reference/agents.md)). Issue [#2322](https://github.com/boxlite-ai/boxlite/issues/2322), opened 2026-11-04, plans to mount pinned Claude Code, Codex and OpenCode binaries into unchanged images. That is worth watching, but it isn't usable today.
- **[I] Server integration:** an Effect service would hold one `JsBoxlite` per environment with a private `homeDir` under T3 home (see T8). It would wrap the napi promises with `Effect.tryPromise`. The native module has to load lazily, because on unsupported hosts the optional dependency is missing and the import fails.

## Supported hosts and detection

- **[V] Support matrix** (README "Supported Platforms"):

  | Host | Status |
  |---|---|
  | macOS on Apple Silicon, 12+ | ✅ |
  | Linux x86_64 and ARM64 with `/dev/kvm` | ✅ |
  | Windows via WSL2 with KVM, user in the `kvm` group | ✅ |
  | macOS on Intel | "Coming soon" |

  Python wheels are tagged `macosx_14_0_arm64`, and CI runs on macOS 15. [I] So in practice, macOS 14 or later.
- **[V] The new VMM drops macOS 12–14** at milestone M5, keeps Intel Macs out of scope, and plans Windows on WHP for M10 ([docs/contributing/architecture/vmm/README.md:30-49](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/docs/contributing/architecture/vmm/README.md#L30-L49)). The native Windows request, [#69](https://github.com/boxlite-ai/boxlite/issues/69), was triaged P4, "very high difficulty", with WSL2 as the recommended path.
- **[V] Detection:** creating a runtime runs `SystemCheck::run()` ([src/boxlite/src/runtime/rt_impl.rs:233](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/src/boxlite/src/runtime/rt_impl.rs#L233)). It opens and smoke-tests `/dev/kvm` on Linux, checks `sysctl kern.hv_support` and the arm64 architecture on macOS, and returns `Unsupported` with an operator-readable diagnostic for KVM missing, permission denied, and similar ([src/boxlite/src/system_check.rs:44-70](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/src/boxlite/src/system_check.rs#L44-L70)). In Node, the `JsBoxlite` constructor throws.
- **[I] Detection order for T3:** (1) the optional native package fails to import, (2) the constructor throws `Unsupported`, (3) a smoke `create` plus `exec("true")` at first use. Map each to the "disabled with reason" UI.
- **[V] macOS signing:** the shim is ad-hoc signed with the `com.apple.security.hypervisor` and `disable-library-validation` entitlements ([scripts/build/sign.sh](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/scripts/build/sign.sh)). [I] Whether a notarized Electron app can run that shim as extracted from the npm package, or must re-sign it, is untested and open.
- **[V] Nested virtualization in the cloud is fragile.** For example, issue [#2239](https://github.com/boxlite-ai/boxlite/issues/2239) (open) reports that 0.10.8 can't start on EC2 nested KVM. The guest-ready timeout is a hard-coded 30 s with no override ([#1111](https://github.com/boxlite-ai/boxlite/issues/1111)).

## Mounts

- **[V] Mounts are virtio-fs host binds:** `volumes: [{ hostPath, guestPath, readOnly? }]`, local runtime only. Each directory volume becomes its own virtio-fs device ([src/boxlite/src/volumes/container_volume.rs](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/src/boxlite/src/volumes/container_volume.rs)). The volume set is fixed at create time, and hot-plug is a later VMM milestone. [I] So changing a thread's workspace, for example in a worktree handoff, means recreating the box.
- **[V] Volume limit:** see T2. Two directory volumes is the ceiling on Linux x86_64 today.
- **[V] Ownership on Linux:**
  - libkrun's passthrough runs unprivileged. `set_creds` allows guest uid and gid 0, or the host's own uid and gid, and returns `EPERM` for anything else ([src/devices/src/virtio/fs/linux/passthrough.rs:833-866](https://github.com/boxlite-ai/libkrun/blob/e12b9b3780ffa8df9f3e1797b217d13453479167/src/devices/src/virtio/fs/linux/passthrough.rs#L833-L866)).
  - BoxLite stats the host path and applies an idmapped mount that swaps the host owner's uid with the container user's uid on read-write mounts ([src/guest/src/container/lifecycle.rs:151-176](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/src/guest/src/container/lifecycle.rs#L151-L176), [src/guest/src/storage/idmap.rs](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/src/guest/src/storage/idmap.rs)).
  - In the probe, root in the guest saw host-owned files as `0:0`. Files the guest created were owned by `1500:988` (the host user) on the host, with the requested modes. As uid 1000 the guest got `Permission denied`, and `chown` returned `Invalid argument`.
- **[V] Ownership on macOS:** the macOS passthrough records the guest's uid, gid and mode in a `user.containers.override_stat` xattr on each host file it creates ([src/devices/src/virtio/fs/macos/passthrough.rs:35](https://github.com/boxlite-ai/libkrun/blob/e12b9b3780ffa8df9f3e1797b217d13453479167/src/devices/src/virtio/fs/macos/passthrough.rs#L35)). [I] So workspace files the guest creates gain that xattr on the host. That's harmless to git, but visible to `xattr -l`.
- **[V] Symlinks** pass through unchanged as symlinks, both relative and absolute (probe: `abs-link -> /etc/hostname` on the host). [I] They resolve in whichever namespace reads them. An absolute link created in the guest points at a host path when host git or the host's file APIs follow it. T3's host-side file APIs must not follow links out of the workspace. That's consistent with the plan's "treat the sandboxed repo as untrusted".
- **[V] Consistency:** see T6. The guest uses `CachePolicy::Auto` with a 5 s entry and attribute timeout and writeback off ([passthrough.rs:336-339](https://github.com/boxlite-ai/libkrun/blob/e12b9b3780ffa8df9f3e1797b217d13453479167/src/devices/src/virtio/fs/linux/passthrough.rs#L336-L339); BoxLite passes the defaults). Guest writes reach the host file before the syscall returns, so host git sees them immediately. Host rewrites can look stale or truncated in the guest for up to about 5 s.
- **[I] Speed:** I found no published numbers for large repos. The docs say only "fast for small files, slight overhead for large files" ([docs/guides/volumes.md](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/docs/guides/volumes.md)). Today every virtio-fs request goes through one in-process FUSE worker, and the new VMM design notes that a slow request stalls the other devices (vmm/README.md, "Threads"). This needs a benchmark (`git status` on a large worktree) before committing.

## Networking

- **[V] Guest to host loopback:** use `host.boxlite.internal` or `192.168.127.254`, which gvproxy NATs to the host's `127.0.0.1` ([src/boxlite/src/net/constants.rs:15-32](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/src/boxlite/src/net/constants.rs#L15-L32), [docs/guides/networking.md](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/docs/guides/networking.md), "From Box to Host Loopback"). In the probe, `wget http://host.boxlite.internal:<port>` reached a host server bound to `127.0.0.1`. Under a non-empty allowlist, `192.168.127.254` has to be listed explicitly.
- **[V] Host to guest, two mechanisms:**
  - `ports: [{ guestPort, hostPort?, hostIp? }]` is TCP only, fixed at create time, and defaults to `hostIp` `0.0.0.0`. [I] Set it to `127.0.0.1`.
  - `box.network.tunnel(SocketAddress)` followed by `forward({ type: "tcp", host, port: 0 })` binds a local listener on a running box dynamically.

  Both reach only services listening on `0.0.0.0` or the guest IP ([src/boxlite/src/litebox/network.rs:369-385](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/src/boxlite/src/litebox/network.rs#L369-L385); threat T3).
- **[V] Outbound policy:**
  - `network: { mode: "enabled" | "disabled", allowNet?: string[] }`. Rules are exact or `*.` wildcard hostnames (matched on HTTP Host or TLS SNI), IPv4 addresses, or IPv4 CIDRs; UDP needs IP rules.
  - The empty list (the default) means unrestricted. `disabled` at create time means no network card at all.
  - `box.network.updateOutbound` replaces the policy on a running box. New flows use the new policy and existing flows keep theirs, and the policy persists across stop and start.
  - Inbound policy is enforced only by BoxLite's cloud proxy; it is unsupported locally ([docs/guides/networking.md](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/docs/guides/networking.md)).
- **[V] Secret placeholders:** `secrets: [{ name, value, hosts }]` substitutes a placeholder in outbound HTTPS traffic through a MITM proxy with a per-box CA, so the real value never enters the VM ([docs/concepts/networking.md](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/docs/concepts/networking.md)). [I] This could be an option for "credentials are injected per provider instance", for example API keys, but not for OAuth flows that refresh tokens inside the guest.

## Lifecycle

- **[V] Persistence:** `autoDelete: 0` keeps the box and its copy-on-write QCOW2 disk after stop. The default removes it locally. `stop()` followed by `start()` cold-boots the VM on the kept disk; there is no memory snapshot. The probe confirmed that a file under `/root` survives stop and start.
- **[V] Outliving the host process:** with `detach: true`, a box survives the parent process exiting, and a new runtime can call `get(id)` to reach it again ([src/boxlite/src/runtime/options.rs:444-455](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/src/boxlite/src/runtime/options.rs#L444-L455)). On runtime start, `recover_boxes` resets boxes after a host reboot and cleans up orphans. AutoStop and AutoResume exist only on the REST server (options.rs:424), so stopping an idle guest is T3's job.
- **[V] Latency:**

  | Measurement | Time | Source |
  |---|---|---|
  | `stop` | 71 ms | Probe, Linux |
  | Restart of a stopped box plus the first exec | 0.94 s | Probe, Linux |
  | Fresh box from a cached image | 1.9 s | Probe, Linux |
  | First box including the image pull | 2.8 s (1.8 s of it pulling) | Probe, Linux |
  | `remove` | 24 ms | Probe, Linux |
  | `start` with the jailer on | about 2.1 s | [docs/contributing/investigations/boot-latency-analysis.md](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/docs/contributing/investigations/boot-latency-analysis.md) (Apple Silicon, February 2026) |

  About 1.4 s of the macOS time is code-signing validation of shim and dylib copies made per box.
- **[V] Snapshots and copies:** `box.snapshot.create/list/restore/remove`, `cloneBox` and `export` to a `.boxlite` archive ([sdks/node/src/snapshots.rs](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/sdks/node/src/snapshots.rs)). [I] These could back "reset", but recreating the box from its image is simpler.

## Running commands

- **[V] Exec:** `box.exec(cmd, args?, env?, tty?, user?, timeoutSecs?, workingDir?)`. The execution handle offers `stdin()` (with `write`, `writeString` and `close`), `stdout()`, `stderr()`, `wait()` returning `{ exitCode }`, `kill()` (SIGKILL), `signal(n)` and `resizeTty(rows, cols)` ([sdks/node/src/box_handle.rs:60-110](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/sdks/node/src/box_handle.rs#L60-L110), [sdks/node/src/exec.rs](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/sdks/node/src/exec.rs)). In the probe, env and cwd were applied, `exit 7` returned 7, a SIGTERM trap returned 42, `kill` returned -9, and a PTY honored `resizeTty(40,120)` (`stty size` printed `40 120`).
- **[V] Streams are lossy UTF-8 strings in arbitrary chunks, not lines.** The docs say "line", but the source decodes gRPC chunks into a `String` with a decoder that preserves partial code points ([src/boxlite/src/portal/interfaces/exec.rs:341-350](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/src/boxlite/src/portal/interfaces/exec.rs#L341-L350)). The probe got `"…partial"` and `"-rest\n"` as separate chunks. [I] JSON-RPC over stdio still needs T3's own line framing, which it already has. Binary or non-UTF-8 output gets corrupted with U+FFFD replacement characters, which is fine for terminals and provider protocols.
- **[V] Executions are tied to the handle:** see T9. A lost handle means a lost process on the local backend.

## Images

- **[V] OCI images:** BoxLite pulls OCI images from any registry. Registry auth, plain-HTTP or insecure registries, and a pull proxy are configured on the runtime (`imageRegistries`; [docs/guides/image-registry-configuration.md](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/docs/guides/image-registry-configuration.md)). Layers are cached by digest under `$BOXLITE_HOME/images` and shared across boxes, and each box gets a copy-on-write disk. A tag is resolved once, when the box is created; restarts reuse the box's disk. When the registry is unreachable, BoxLite falls back to the cached build ([docs/concepts/images.md](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/docs/concepts/images.md)).
- **[V] No image build.** The rootfs is either `Image(ref)` or `RootfsPath(dir)` ([src/boxlite/src/runtime/options.rs:812-817](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/src/boxlite/src/runtime/options.rs#L812-L817)). [I] The T3 default image is built with Docker or Buildah in CI and pushed to a registry, or unpacked to a rootfs directory. The cache is per `BOXLITE_HOME`, so it isn't shared with Docker or with other homes.

## Resource limits and metrics

- **[V] Limits:**
  - `cpus` is 1–16; the bundled kernel has `CONFIG_NR_CPUS=16`.
  - `memoryMib` is 128–65536, default 512 MiB, and is a hard limit.
  - `diskSizeGb` is sparse.
  - The host shim also runs under cgroups and rlimits.

  Sources: [docs/guides/resource-limits.md](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/docs/guides/resource-limits.md) and [src/boxlite/src/jailer/cgroup.rs](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/src/boxlite/src/jailer/cgroup.rs).
- **[V] Metrics** come from host-side `sysinfo` on the shim process ([src/boxlite/src/vmm/controller/shim.rs:206-207](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/src/boxlite/src/vmm/controller/shim.rs#L206-L207)). They include CPU percent, memory, network bytes and TCP counts, and boot-stage timings. Exec and byte counters are also available. See T7 for OOM.

## License, cadence and maturity

- **[V] License:** Apache-2.0. Contributions are under a CLA ([docs/legal/CLA.md](https://github.com/boxlite-ai/boxlite/blob/3f8e7d2dedd338895399f45ff597145a53eca9b3/docs/legal/CLA.md)).
- **[V] Cadence and size:** releases from v0.5.8 (2026-01-29) to v0.11.0 (2026-10-30), roughly every 1–2 weeks. The repo has about 2.7k stars. It's a monorepo that also contains BoxLite's hosted cloud (dashboard, runner, proxy), and much of its recent work targets the cloud.
- **[V] Adoption:** the README lists Databricks Omnigent, Alibaba AgentScope Runtime and ByteDance deer-flow as frameworks that use it.
- **[V] Known limitations not covered above:**
  - A dropped attach can't detach cleanly ([#2045](https://github.com/boxlite-ai/boxlite/issues/2045)).
  - The 30 s guest-ready timeout ([#1111](https://github.com/boxlite-ai/boxlite/issues/1111)).
  - The macOS boot cost from code signing.
  - Inbound policy is enforced by the cloud only.

## Open questions this research did not settle

1. Large-repo virtio-fs performance (`git status`, `pnpm install` on a worktree) on Linux and macOS.
2. Whether a notarized T3 desktop on macOS can run the npm-embedded shim as it is.
3. The guest network address from a WSL2 host, and whether the T3 server and the guest can see each other there.
4. Whether `detach: true` plus the missing local exec attach is acceptable, or whether guests should simply die with the server.

## Probe transcript (abridged)

```text
first exec (includes boot): 2763 ms   metrics {"create":2718,"pull":1777,"mem":2174976}
guest: uid=0(root)  kernel 6.12.87  uservol0 on /workspace type virtiofs (rw,relatime,idmapped)
host sees fromguest.txt: uid=1500 gid=988 mode=755
guest chown -> "Invalid argument";  touch as 1000:1000 -> "Permission denied"
host rewrite then guest cat -> "a much" (stale size); after 6 s -> full content
host loopback via host.boxlite.internal: hello-from-host-loopback
env/cwd/exit: code=7 chunks=["bar\n/workspace\npartial","-rest\n"]
SIGTERM exit=42   kill exit=-9   pty: "40 120\r\n/dev/pts/0\r\n"
published/tunnel -> guest 0.0.0.0:8001: HTTP 200;  guest 127.0.0.1:8002: ECONNRESET
OOM (256 MiB box): exec rc=137, dmesg "tail invoked oom-killer", box still running
stop 71 ms; restart+exec 940 ms; fresh box cached image 1884 ms; remove 24 ms
three directory volumes: "The VM failed to start."  two volumes: ok
```
