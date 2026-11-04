# Claude Code and Codex inside a Linux guest

Research for [#13](https://github.com/jonaswre-labs/t3code/issues/13) (map: [#10](https://github.com/jonaswre-labs/t3code/issues/10)).
Question: what does each CLI need to run inside a T3-launched Linux guest: install, auth, token
refresh, state dirs, and how T3 launches it today.

Sources checked on 2026-11-04. T3 code references are `path:line` at commit `b227694a7`. Codex source
references are pinned to tag `rust-v0.156.1`, the version T3's managed installer pins. Labels:
**[V]** means verified in a primary source (docs, source, registry metadata). **[R]** means reported in a
first-party issue tracker but not documented. **[I]** means my inference and is not verified.

## TL;DR

- **Both ship as static Linux binaries.** No Node is needed in the guest for either one. Claude is
  about 325–340 MB unpacked. Codex is a 146 MB tarball, or about 387 MB unpacked from npm. T3 already
  pins Codex musl tarballs with SHA-256 checksums that an image build can reuse.
- **The token-refresh ownership risk is real for both CLI logins.** Codex ChatGPT refresh tokens
  are single use (`refresh_token_reused`). Claude subscription logins also appear to rotate
  [R]. If you copy a host credential file into a guest, the first side to refresh logs the other
  out. The CLI clears the stored login, so the user has to sign in again on the host. A read-only
  copy is worse: the refresh consumes the token and then the write of the new one fails [I].
- **Ways to keep refresh on the host:**
  - **Codex, managed ChatGPT**: already host-owned. Codex calls an auth *command*. The command asks
    T3's host token broker over a private Unix socket, and the broker returns only an access token.
    The guest needs a forwarded broker endpoint and a small helper, but no secrets on disk.
  - **Codex, existing CLI login**: the app-server protocol has `chatgptAuthTokens` login plus a
    server-to-client `account/chatgptAuthTokens/refresh` request. T3 is the guest app-server's
    client, so T3 can feed it host-minted access tokens and the guest never stores a refresh token.
    The protocol exists, but nobody has tested this end to end.
  - **Claude subscription**: there is no documented broker hook. The clean option is
    `CLAUDE_CODE_OAUTH_TOKEN` from `claude setup-token`. It lasts one year, has no refresh, is
    injected as an env var, and only works for model requests.
  - **API keys** (Anthropic, OpenAI): static. Injecting them read-only is fine.
- **Claude state must be visible to the host. Codex state need not be.** T3 reads Claude transcripts
  directly on the host for fork, resume checks, and history, and it tails task output in
  `/tmp/claude-<uid>`. So the guest's Claude config dir should be a host-visible mount, and the
  workspace path in the guest must equal the host path (or set `CLAUDE_CODE_PROJECT_DIR_NAME`).
  Codex resume, fork, and rollback all go through app-server RPC.

## 1. How T3 launches each one today

### Claude (`ClaudeAdapterV2`)

- T3 uses the Claude Agent SDK (`@anthropic-ai/claude-agent-sdk ^0.3.276`, `apps/server/package.json:36`).
  `query()` runs inside the T3 server process. The SDK spawns the `claude` CLI and speaks its
  stream-json protocol over stdio [V].
- T3 supplies its own `spawnClaudeCodeProcess` (`apps/server/src/orchestration-v2/Adapters/ClaudeAdapterV2.ts:1022`, passed at `:1045`).
  It wraps the binary through `AgentScope.wrap` (`:980`–`:1000`). The SDK documents this hook as the way
  "to run Claude Code in VMs, containers, or remote environments"
  ([TS reference](https://code.claude.com/docs/en/agent-sdk/typescript)) [V]. That makes it the natural seam
  for a sandbox driver's `exec` with piped stdio [I].
- The binary is the instance's `binaryPath`, defaulting to the user's `claude`. It is not the SDK's
  bundled binary. The env comes from the instance, and `homePath` becomes `CLAUDE_CONFIG_DIR`
  (`apps/server/src/provider/Drivers/ClaudeHome.ts:30`–`43`).
- T3's MCP server goes over stdin, not argv or env (`ClaudeAdapterV2.ts:969`). It is an HTTP server on
  the host with a bearer header (`:1732`). A guest needs a route back to that endpoint [I].
- There is one CLI process per thread (`supportsMultipleProviderThreadsPerSession: false`,
  `ClaudeAdapterV2.ts:368`).
- **Host-side reads of Claude's private files:**
  - `forkSession` and `getSessionMessages` read `<config>/projects/…` (`:1268`–`:1325`). A custom home
    is handled through a worker subprocess (`:876`).
  - `hasEmptyClaudeTranscript` scans `<config>/projects/*/<id>.jsonl` (`:10968`).
  - `ClaudeTaskOutputTail` tails `/tmp/claude-<uid>` or `CLAUDE_CODE_TMPDIR`
    (`ClaudeTaskOutputTail.ts:17`). The tail is documented as best effort.

  A guest breaks all of these unless those paths are mounted back to the host [I].
- Version coupling: SDK 0.3.276 declares `claudeCodeVersion = 2.1.276` in its npm metadata [V]. Pin the
  guest CLI near the SDK's version instead of "latest" [I].

### Codex (`CodexAdapterV2`)

- T3 spawns `codex app-server [launchArgs]` and speaks JSON-RPC over stdio
  (`apps/server/src/orchestration-v2/Adapters/CodexAdapterV2.ts:1376`–`1405`, `:1495`–`:1540`;
  `apps/server/src/provider/codexLaunchArgs.ts:12`). The env adds `CODEX_HOME` and
  `CODEX_INTERNAL_ORIGINATOR_OVERRIDE` (`apps/server/src/provider/CodexProvider.ts:447`–`452`).
- One app-server hosts many threads (`supportsMultipleProviderThreadsPerSession: true`,
  `CodexAdapterV2.ts:240`; `AgentScope.wrap({ shared: true })`, `:1392`; `thread/unsubscribe`, `:7172`).
  The map already says shared servers become one per guest.
- **Two setup modes** (`setupMode: "managed" | "existing"`, `packages/contracts/src/settings.ts:631`):
  - **existing**: the user's `codex` and `CODEX_HOME`, optionally with a *shadow home*. A shadow home
    keeps a private `auth.json` and symlinks shared state to the main home
    (`apps/server/src/provider/Drivers/CodexHomeLayout.ts:18`–`41`).
  - **managed**: `CodexManagedRuntime` uses T3's own pinned Codex install. `CODEX_HOME` lives under T3's
    state dir, and T3 strips `OPENAI_API_KEY`, `OPENAI_BASE_URL`, and `ACCESS_TOKEN`. T3 also injects
    `-c model_provider="openai_token_sharing"` with
    `auth={command=<t3>, args=["__managed-chatgpt-token", <socket>, <instance>, <env>, <profileKey>], refresh_interval_ms=0}`
    (`apps/server/src/provider/CodexManagedRuntime.ts:22`–`32`, `:61`–`:135`).
- Nothing in the adapter reads Codex rollout files directly. Resume, fork, and revert all go through RPC
  (`thread/resume`, `thread/fork`). This is based on a grep of the adapter and `CodexThreadRevert.ts` [V].

## 2. Install into an OCI image (pinned)

| | Claude Code | Codex |
| - | - | - |
| Pinned install | `curl -fsSL https://claude.ai/install.sh \| bash -s <ver>`; signed apt/dnf/apk repos; or `npm i -g @anthropic-ai/claude-code@<ver>` ([setup](https://code.claude.com/docs/en/setup)) [V] | GitHub release `codex-package-<target>.tar.gz`. T3 pins 0.156.1 with SHA-256 for `x86_64-unknown-linux-musl` and `aarch64-unknown-linux-musl` (`apps/server/src/provider/CodexInstallation.ts:59`–`100`) [V]. npm: `@openai/codex@<ver>` with per-platform optional deps |
| Runtime | Native binary that "does not itself invoke Node". npm needs Node 22+ only for the install [V]. Alpine/musl needs `libgcc libstdc++ ripgrep` and `USE_BUILTIN_RIPGREP=0` [V]. Git is optional [V] | Static musl binary [V]. The Linux sandbox wants `bwrap` on `PATH`, or else unprivileged user namespaces for the bundled helper ([sandboxing](https://learn.chatgpt.com/docs/sandboxing)) [V] |
| Size | npm `@anthropic-ai/claude-code-linux-x64@2.1.318` unpacks to 337.8 MB, the `-musl` package to 324.3 MB (registry `dist.unpackedSize`) [V] | The 0.156.1 x86_64 musl tarball is 146.0 MB (`content-length`). npm `@openai/codex@0.156.1-linux-x64` unpacks to 386.9 MB [V] |
| Integrity | Signed `manifest.json` with per-platform SHA-256 at `downloads.claude.ai/claude-code-releases/<ver>/`. Linux binaries themselves are not code-signed [V] | SHA-256 table in T3 [V] |
| Disable self-update | `DISABLE_UPDATES=1`. `DISABLE_AUTOUPDATER` only stops the background check [V] | No self-update in app-server mode that I found [I] |
| Guest sizing | Docs list "4 GB+ RAM" as a system requirement [V] | Not stated |

Guest sandbox note [I]: Codex's own `workspace-write` sandbox needs bwrap or userns inside the guest.
Since the guest is the boundary, T3 can either ship bwrap and allow userns, or map sandboxed threads to
`danger-full-access`. Which one is a runtime-mode decision. The map says that is independent of
sandboxing, so the decision belongs in the spec.

## 3. Authentication per account type

| T3 account type | Where the credential lives | Minimum to inject | Read-only OK? |
| - | - | - | - |
| Claude subscription (`/login`) | Linux: `<CLAUDE_CONFIG_DIR or ~/.claude>/.credentials.json` (0600). macOS: Keychain, keyed per config dir ([auth docs](https://code.claude.com/docs/en/authentication)) [V] | Prefer `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`; one year; "can only make model requests"; precedence #5) [V]. Do not copy `.credentials.json` (see §4) | Yes for the env token. No for the credentials file [I] |
| Claude API key / gateway | `ANTHROPIC_API_KEY` (precedence #3), `ANTHROPIC_AUTH_TOKEN` (#2), `apiKeyHelper` (#4), cloud provider env (#1) [V]. T3 sets these as per-instance env vars with secret references (`packages/contracts/src/settings.test.ts:169`–`190`; `docs/user/providers-claude.md:101`–`108`) | The env var | Yes |
| Codex managed ChatGPT (T3-owned) | T3's server secret store (`ProviderCredentialStore "codex-chatgpt"`, `apps/server/src/provider/CodexChatGptAuth.ts:159`). Never in `CODEX_HOME` | Nothing secret. The guest needs the auth-command helper and a reachable broker endpoint | n/a |
| Codex existing ChatGPT login | `$CODEX_HOME/auth.json` (0600) or the OS keyring, chosen by `cli_auth_credentials_store = file \| keyring \| auto \| ephemeral` ([auth docs](https://learn.chatgpt.com/docs/auth); [`storage.rs:206`](https://github.com/openai/codex/blob/rust-v0.156.1/codex-rs/login/src/auth/storage.rs#L206-L223)) [V] | Prefer external tokens through app-server (§4). Do not copy `auth.json` | No |
| Codex API key | `auth.json` `OPENAI_API_KEY` field (`codex login --with-api-key`) [V]. **app-server ignores the `CODEX_API_KEY` env var** (`enable_codex_api_key_env: false`, [`app-server/src/lib.rs:541`](https://github.com/openai/codex/blob/rust-v0.156.1/codex-rs/app-server/src/lib.rs#L541)) [V] | Write a guest `auth.json` holding only the key, or call `account/login/start {type:"apiKey"}` [V protocol, I persistence] | Yes, since the key is static [I] |

## 4. Token refresh and rotation

### Codex ChatGPT login (verified in source)

- Refresh tokens are single use. A reused one fails with `refresh_token_reused`, which Codex maps to
  `Exhausted`: "your refresh token was already used. Please log out and sign in again"
  ([`manager.rs:203`–`216`, `:1664`–`:1687`](https://github.com/openai/codex/blob/rust-v0.156.1/codex-rs/login/src/auth/manager.rs#L1664-L1687)) [V].
- Codex refreshes proactively when the access JWT is within 5 minutes of expiry, or when
  `last_refresh` is more than 8 days old (`:2955`–`:2977`). It also refreshes on 401 after a guarded reload
  from disk (`:2795`–`:2836`). The refresh lock is an in-process semaphore [V]. T3 also notes an
  `auth.lock` file in the Codex home (`CodexHomeLayout.ts:32`).
- The order is: POST to the token endpoint, *then* `persist_tokens` → `storage.save`
  (`:3043`–`:3060`, `:1584`–`:1606`) [V]. If the save fails (read-only file) or the process dies in between,
  the new refresh token is lost and the old one is already spent [I, follows from the order above].
- OpenAI's CI guide: "Do not share the same file across concurrent jobs or multiple machines". It lists
  "another machine or concurrent job rotated the token first" as a reason to reseed
  ([CI/CD auth](https://learn.chatgpt.com/docs/auth/ci-cd-auth)) [V].
- T3 already documents the same race for the primary handoff: "retaining a primary refresh session
  would race refresh-token rotation" (`docs/internals/providers.md:65`–`70`).

**Keeping refresh on the host:**

1. **Managed mode already does this.** Codex runs the auth command and caches its stdout as a bearer.
   It re-runs the command on 401 ([`external_bearer.rs:33`–`60`](https://github.com/openai/codex/blob/rust-v0.156.1/codex-rs/login/src/auth/external_bearer.rs#L33-L60)) [V].
   The command connects to `CodexManagedTokenBroker`. The broker listens on a 0600 Unix socket in a 0700
   private dir (`apps/server/src/provider/CodexManagedTokenBroker.ts:45`–`207`) and checks `environmentId`
   (`:146`). It calls `CodexChatGptAuth.access`, which refreshes under an in-process semaphore plus a
   cross-process file lock and returns only `{token}` (`CodexChatGptAuth.ts:739`–`801`;
   `CodexChatGptSessionLock.ts`). The wire format is one JSON line each way:
   `{version:1, instanceId, environmentId, profileKey}` → `{token}`
   (`CodexManagedTokenProtocol.ts:6`–`20`).

   In a guest, the helper today is `t3 __managed-chatgpt-token` (`apps/server/src/bin.ts:31`), a host
   binary, and the map rules out mounting host binaries. The guest therefore needs (a) the socket
   forwarded (vsock, or a forwarded Unix socket) and (b) a tiny helper in the image. The protocol is
   trivial enough for `socat` [I]. The broker has no per-caller auth beyond `environmentId` and
   `profileKey`. That is acceptable under the map's threat model, but anything in the guest can mint
   access tokens for that instance [I].
2. **Existing login through external tokens.** `LoginAccountParams` has
   `{type:"chatgptAuthTokens", accessToken, chatgptAccountId, chatgptPlanType?}`. The app-server sends the
   client `account/chatgptAuthTokens/refresh {reason:"unauthorized", previousAccountId}` and expects
   `{accessToken, chatgptAccountId, chatgptPlanType}` back. External tokens are stored `Ephemeral`,
   in memory only ([`manager.rs:1115`–`1131`, `:1485`–`:1519`](https://github.com/openai/codex/blob/rust-v0.156.1/codex-rs/login/src/auth/manager.rs#L1485-L1519)) [V].
   T3 can mint the access token on the host from a host-side app-server on the user's `CODEX_HOME` with
   `getAuthStatus {includeToken:true, refreshToken:true}`, which returns `authToken` [V schema].
   Both methods are already in T3's generated client
   (`packages/effect-codex-app-server/src/_generated/meta.gen.ts:109`, `:124`).

   Not verified: whether `getAuthStatus` (a v1 method) stays supported, and whether external-token mode
   loses any features compared with a full login [I].
3. **API key**: no refresh, so nothing to own.

### Claude subscription login (partly documented)

- Docs: the stored login refreshes automatically. Once it "can't be refreshed, each model request
  fails with `Login expired`" ([auth](https://code.claude.com/docs/en/authentication#renew-an-expiring-login)) [V].
  The docs do **not** say whether refresh tokens rotate.
- Reports in `anthropics/claude-code` describe rotation and a cross-process race on one
  `.credentials.json`. Examples: [#102985](https://github.com/anthropics/claude-code/issues/102985), where
  many processes sharing `~/.claude/.credentials.json` get logged out every 1–2 days, coordinated through
  `.oauth_refresh.lock`, with roughly 8 h access tokens; [#99314](https://github.com/anthropics/claude-code/issues/99314);
  and [#105645](https://github.com/anthropics/claude-code/issues/105645), "loses rotated tokens, then clears
  credentials" [R]. After an `invalid_grant` the CLI clears the stored tokens, which logs out every
  process that uses that file [R].
- So a *copy* of `.credentials.json` in the guest will eventually log out the host's Claude, and the
  other way round [I, from R]. Mounting the host's config dir read-write would share one file and lock.
  That only works if the guest's shared filesystem honours the lock across the VM boundary, which is
  unverified for BoxLite. It also exposes the whole `~/.claude` to the guest [I].
- **No documented host-broker hook exists for subscription OAuth.** `apiKeyHelper` returns an API
  credential and is re-run on a TTL or a 401. The docs do not say whether a subscription OAuth access
  token can be served through it [I, not verified].
- Options, best first [I]:
  - **`CLAUDE_CODE_OAUTH_TOKEN`**: from `claude setup-token`, a one-year token injected per instance as a
    sensitive env var, the same as an API key. It has no refresh and no file. Docs limit it to model
    requests, with no Remote Control and no claude.ai connectors. Locally configured MCP servers still
    work [V], which covers T3's MCP.
  - **API key or gateway token.**
  - **Shared mount of the host config dir.** Risky.

  Never copy `.credentials.json` into the guest.
- The SDK `SessionStore` mirror does not help with credentials. Its resume path also writes a temp
  `CLAUDE_CONFIG_DIR` on the **SDK host**, which a guest-spawned CLI would not see
  ([session storage](https://code.claude.com/docs/en/agent-sdk/session-storage)) [V doc, I for guest].

## 5. Config and state dirs; persistence across guest stops

### Claude (`CLAUDE_CONFIG_DIR`, default `~/.claude` plus `~/.claude.json`) [V, [claude-directory](https://code.claude.com/docs/en/claude-directory)]

- **Needed for resume**: `projects/<project>/<session>.jsonl`, plus `<session>/subagents/` and
  `tool-results/`. `<project>` is the cwd with every non-alphanumeric character replaced by `-`.
  `CLAUDE_CODE_PROJECT_DIR_NAME` overrides it.
- `file-history/` (rewind), `.claude.json` (account/app state, per-project trust, personal MCP servers;
  it moves inside the config dir when `CLAUDE_CONFIG_DIR` is set), `settings.json`, `sessions/`,
  `session-env/`, `shell-snapshots/`, `plans/`, `tasks/`, `backups/`.
- Temp (not in the config dir): `/tmp/claude-<uid>/<project>/<session>/` holds the scratchpad, images,
  and task output. `CLAUDE_CODE_TMPDIR` moves it.
- Retention: `cleanupPeriodDays` defaults to 30.
- **Persistence:** the map's guest is persistent across stop and resume, so guest-local state survives
  a stop. However, T3 reads `projects/` from the host (§1), and reset or delete would lose resume.
  Recommendation [I]: put a per-thread, credential-free Claude config dir in T3's host state dir and
  mount it into the guest at a fixed path. Mount the workspace at the **same absolute path** as on the
  host so `<project>` matches, or set `CLAUDE_CODE_PROJECT_DIR_NAME`. Set `CLAUDE_CODE_TMPDIR` into a
  mounted dir if live task-output tailing should keep working.

### Codex (`CODEX_HOME`, default `~/.codex`) [V]

- `config.toml`, `auth.json`/`auth.lock`, `models_cache.json`.
  `sessions/YYYY/MM/DD/rollout-<ts>-<id>.jsonl` and `archived_sessions/`
  ([`rollout/src/lib.rs:84`–`85`](https://github.com/openai/codex/blob/rust-v0.156.1/codex-rs/rollout/src/lib.rs#L84-L85)).
  Also `sqlite/` (state DB), `shell_snapshots/`, `worktrees/`, `skills/`, `plugins/`, `cache/`, `logs/`,
  `memories/`, `tmp/`, `mcp-oauth-locks/`, and `thread-writer-locks/` (T3's inventory,
  `CodexHomeLayout.ts:18`–`41`).
- **Persistence:** `thread/resume` needs the rollout. T3 does not read rollouts on the host, so
  guest-local storage works as long as the guest lives. A reset loses native resume, and T3 falls back
  to a handoff summary [I]. A host-mounted per-thread `CODEX_HOME` would survive reset, matching Claude
  [I]. Shadow-home sharing between instances does not carry over into per-thread guests [I].

### User configuration the guest won't have [I]

Users' own `~/.claude` (skills, `CLAUDE.md`, MCP servers) and `~/.codex/config.toml` (MCP servers,
profiles) are not in the guest. T3 currently loads `user` setting sources for non-custom Claude homes
(`ClaudeAdapterV2.ts:4532`). The spec needs to decide what, if anything, gets seeded. MCP servers that
are stdio commands also need their binaries in the image.

## Open questions for the spec

1. BoxLite's shared-filesystem semantics: do `flock`/`fcntl` locks, `O_EXCL`, and rename atomicity hold
   across the mount? That decides whether host-mounted config dirs are safe.
2. Host↔guest channel for the Codex token broker and T3's MCP endpoint (vsock, port forward). One
   channel could carry both.
3. Should T3 offer `claude setup-token` as a first-class "sandbox credential" for Claude subscription
   instances, given it's a separate one-year grant the user must mint and can revoke?
4. Codex sandbox mode inside the guest: ship bwrap and enable userns, or use `danger-full-access`?
5. Version policy: pin the guest CLIs to the SDK's `claudeCodeVersion` and T3's Codex pin, or follow
   the host's installed versions?
