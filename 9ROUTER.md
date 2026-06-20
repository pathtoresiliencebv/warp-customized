# Warp-Customized + 9router

This is a fork of [`warpdotdev/warp`](https://github.com/warpdotdev/warp) patched so
the open-source (`Oss`) build honors the server URL override flags / env vars. That
makes it possible to point Warp at a self-hosted **9router** instead of `warp.dev`
without rebuilding from the internal `warp-channel-config` tool.

## TL;DR

```bash
# 1. One-time setup
cp 9router.toml.example 9router.toml
$EDITOR 9router.toml           # set server_root_url / ws_server_url

# 2. Build & run
./script/run-9router
```

Warp will then talk to your 9router for every GraphQL / RTC / agent-mode request.

## Why this fork exists

Warp's official OSS build hard-codes the production server URLs
(`https://app.warp.dev`, `wss://rtc.app.warp.dev/graphql/v2`) and the
`Channel::allows_server_url_overrides()` check explicitly excludes the `Oss`
channel, so the existing `--server-root-url` / `WARP_SERVER_ROOT_URL` knobs are
silently ignored. This fork flips that one check so OSS users (you) can redirect
Warp at any compatible backend.

The actual backend contract that Warp expects is the GraphQL schema in
`crates/warp_graphql_schema/api/schema.graphql`. Anything that speaks that schema
(and Firebase auth, or `skip_login`) will work. **9router** is whatever proxy /
warp-server replacement you point at.

## Files added by this fork

| Path | Purpose |
| --- | --- |
| `9router.toml.example` | Template config. Copy to `9router.toml` (gitignored). |
| `9router.toml`         | Your local 9router config (created by the script). |
| `script/run-9router`   | Reads `9router.toml` and runs `cargo run` with the right env vars. |
| `9ROUTER.md`           | This document. |

## Files modified by this fork

- `crates/warp_core/src/channel/mod.rs` — `Channel::allows_server_url_overrides()`
  now returns `true` for `Channel::Oss` (still `false` for `Stable` / `Preview`).
- `app/src/lib.rs` — comment updated to reflect the new behaviour.

## Configuration

`9router.toml` is parsed by `script/run-9router`. Every field has a default;
the only required ones are `server_root_url` and `ws_server_url`.

```toml
server_root_url = "http://localhost:8080"
ws_server_url   = "ws://localhost:8080/graphql/v2"
# session_sharing_server_url = "ws://localhost:8081"   # optional

profile  = "debug"
features = ["skip_login"]                             # comma-free, space-separated

[env]
# WARP_FIREBASE_API_KEY = "..."      # if your 9router uses a different Firebase project
# WARP_DISABLE_TELEMETRY = "1"       # recommended for self-hosted setups
```

CLI overrides win over the file:

```bash
./script/run-9router --profile release --features "skip_login fast_dev"
./script/run-9router --server-root-url http://10.0.0.5:8080
./script/run-9router -- --no-default-features      # passed straight to cargo
./script/run-9router -- --some-warp-flag           # passed straight to warp after `--`
```

## Environment variables

`script/run-9router` exports these for the Warp process:

| Variable | Effect |
| --- | --- |
| `WARP_SERVER_ROOT_URL`            | HTTP root URL (graphql queries, REST). |
| `WARP_WS_SERVER_URL`              | WebSocket endpoint for the GraphQL subscription stream. |
| `WARP_SESSION_SHARING_SERVER_URL` | Only set if `session_sharing_server_url` is in the config. |
| `WARP_FIREBASE_API_KEY`           | Only set if `env.WARP_FIREBASE_API_KEY` is in the config. |
| `WARP_DISABLE_TELEMETRY`          | Only set if `env.WARP_DISABLE_TELEMETRY` is in the config. |

These are read by `warp_cli::Args` (see `crates/warp_cli/src/lib.rs`) and applied
via `ChannelState::override_server_root_url` / `override_ws_server_url` /
`override_session_sharing_server_url`.

## Authentication

Warp's login flow hits Firebase (`app.warp.dev`'s project by default). Two ways
to deal with that:

1. **`features = ["skip_login"]`** (recommended for self-hosted setups). Warp
   uses a stub auth identity; your 9router can ignore auth entirely.
2. Point Firebase at your own project by exporting `WARP_FIREBASE_API_KEY` in
   the `[env]` table. You'll also need to register your 9router's domain with
   that Firebase project.

## Build / run from scratch

```bash
git clone https://github.com/<you>/warp-customized.git
cd warp-customized
./script/install_rust   # installs the pinned toolchain
cp 9router.toml.example 9router.toml
$EDITOR 9router.toml
./script/run-9router
```

The first build pulls a large dependency tree (wgpu, axum, cynic, ...) and
takes 15–45 minutes on a clean machine. Use `./script/run-9router --profile
release` for an optimized build.

## Troubleshooting

- **"Invalid server root URL"** — the URL didn't parse. Make sure it includes a
  scheme (`http://` or `https://`).
- **Warp silently still talks to warp.dev** — make sure you ran
  `./script/run-9router` (which exports the env vars) rather than `cargo run`
  directly. The overrides are applied at startup; they cannot be changed
  afterwards without restarting Warp.
- **GraphQL errors mentioning `app.warp.dev`** — your 9router is forwarding the
  request upstream. Either block that in your 9router config, or run with
  `features = ["skip_login"]` so the auth layer doesn't add the upstream
  reference.
- **WebSocket keeps disconnecting** — the `ws_server_url` path must end in
  `/graphql/v2` (e.g. `ws://localhost:8080/graphql/v2`). 9router must speak the
  `graphql-ws` subprotocol on that path.

## Limitations

- `Stable` and `Preview` channels still ignore overrides (intentional — those
  are first-party release builds).
- Warp's CLI subcommands (`oz`, `warpctrl`, ...) may still hit the production
  backend directly. Set the same `WARP_*` env vars in your shell before
  invoking them.
- The internal `release_bundle` build path (using `warp-channel-config`) still
  embeds URLs at compile time; this fork targets the regular `cargo run` /
  `cargo build` path used by OSS contributors.