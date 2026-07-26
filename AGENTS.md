# AGENTS.md — UnblockNeteaseMusic Revive

## Project

A Node.js MITM proxy that **replaces greyed-out song URLs** in Netease Cloud Music with matches from other Chinese music platforms (QQ, Kugou, Kuwo, Migu, Bilibili, YouTube, etc.).

Published as `@unblockneteasemusic-revive/server` on npm.

## Quick commands

```bash
pnpm install            # install dependencies
pnpm test               # run all Jest tests
pnpm test -- --testPathPattern=cache  # single test file
pnpm build              # webpack → precompiled/app.js + precompiled/bridge.js
pnpm prettier -w .      # format everything (tabWidth:4, useTabs, singleQuote)
pnpm start:dev          # run from source with debug logs
DEVELOPMENT=true node app.js   # run from source (no build needed)
node app.js             # run precompiled bundle
```

**Format before committing.** Prettier CI auto-fixes formatting on the `master` branch.

## Package manager

- **pnpm 10** (`packageManager` field in `package.json`).
- Lockfile: `pnpm-lock.yaml` (commit it).
- Config: `.npmrc`.

## Key structure

| Entry | Purpose |
|---|---|
| `app.js` | CLI bootstrap → `src/bootstrap/index.js` → loads `.env` → requires `precompiled/app.js` (or `src/app.js` if `DEVELOPMENT=true`) |
| `bridge.js` | Same bootstrap → `src/bridge.js` (HTTP bridge for cnrelay, listens on port 9000) |
| `nw.js` | Windows service installer/uninstaller (via `node-windows`) |
| `src/provider/match.js` | **Library entry point** (`package.json` `"main"`). Exports `match(id, source?)`. |

## Source layout

```
src/
  app.js          — CLI arg parsing & server startup
  bridge.js       — cnrelay bridge server (HTTP → provider calls)
  server.js       — HTTP/HTTPS proxy core (MITM + tunnel)
  hook.js         — Netease request interception & modification (~844 lines)
  consts.js       — provider registry & default source list
  cli.js          — custom CLI parser (no commander/yargs)
  request.js      — wrapped http/https with proxy support
  cache.js        — in-memory cache with 30min TTL + cleanup every 15min
  logger.js       — pino-based logger
  provider/
    match.js      — source matching orchestrator (default: kugou, bodian, migu, ytdlp)
    find.js       — fetch song metadata from Netease
    select.js     — pick best match by duration (±5s)
    insure.js     — cnrelay proxy for CN-hosted providers
    {qq,kugou,...}.js — individual provider implementations
  exceptions/
    SongNotAvailable, RequestFailed, IncompleteAudioData, ...
```

## Default sources

```js
['kugou', 'bodian', 'migu', 'ytdlp']
```

Override with `-o` flag or `global.source` / `FOLLOW_SOURCE_ORDER=true` env.

## Development mode

`DEVELOPMENT=true` makes the bootstrap `require()` source files directly instead of the webpack bundle. Useful for iteration without running `pnpm build`.

```bash
DEVELOPMENT=true node app.js -o bilibili kugou
```

## Testing

- **Jest v30**. Test files are `*.test.js` alongside source files.
- Integration tests that hit real APIs are in `match.disabled_test.js` (rename to enable).
- Some tests have extended timeouts (15s). Tests that cancel requests or test basic lib functions run quickly.
- Tests requiring network access (e.g. `request.test.js`) use real URLs (example.com).

```bash
pnpm test                              # all tests
pnpm test -- --testPathPattern=request  # single file
```

## Build & bundle

- **Webpack 5** + **swc-loader** + **TerserPlugin** (swc minify, target Node 12).
- Output: `precompiled/app.js` and `precompiled/bridge.js`.
- CI auto-builds `precompiled/` and opens a PR on the `enhanced` branch.
- `pnpm build` to re-bundle manually.

## Important env vars

| Var | Notes |
|---|---|
| `DEVELOPMENT=true` | Use source files; skip precompiled |
| `LOG_LEVEL=debug` | Verbose logs (default: `info`) |
| `JSON_LOG=true` | Machine-readable JSON log output |
| `ENABLE_FLAC=true` | Enable lossless audio quality |
| `ENABLE_LOCAL_VIP=cvip\|svip` | Spoof VIP status |
| `STORE_EXT` | Set to `1` / `true` / `0` for UWP store extension |
| `QQ_COOKIE`, `MIGU_COOKIE`, `JOOX_COOKIE` | Required by specific providers |
| `MIN_BR=320000` | Minimum bitrate to keep; below gets replaced |
| `NO_CACHE=true` | Disable caching |
| `SIGN_CERT`, `SIGN_KEY` | Custom TLS cert paths |

Run via `.env` file (auto-loaded) or inline: `LOG_LEVEL=debug node app.js`

## CI quirks

- **Default branch is `master`**.
- Prettier CI auto-fixes formatting and opens a PR labelled `ci:style`.
- Build CI auto-creates PRs for `precompiled/` updates.
- Docker images published as `unm-revive/unblock-netease-music-revive` with multi-arch support.
- Binary releases via `pkg` for linux-arm64, linux-x64, win-arm64, win-x64.
- Publish triggers: `pnpm publish` when `package.json` version changes on `master`.

## Docker

```bash
docker run unm-revive/unblock-netease-music-revive -o kuwo -p 1234
# or with env:
docker run -e LOG_LEVEL=debug unm-revive/unblock-netease-music-revive
```

## Architecture notes

- **No framework** for the CLI; `src/cli.js` is a hand-rolled arg parser.
- **No TypeScript compilation** in the build step. `.swcrc` transpiles ES2020+ → CommonJS for Node 12. TypeScript is installed only for editor intellisense (via `@types/node` etc.).
- **Provider interface**: Each provider exports `check(info)` that returns `{ url, br, size, md5 }` or throws. Providers can use `insure()` as a cnrelay bridge for CN-restricted APIs.
- **Cache**: In-memory `Map` with 30 min TTL. Cleanup runs every 15 min. Use `NO_CACHE=true` to disable.
- **Certificate**: MITM TLS requires a CA cert. `generate-cert.sh` produces it. The repo ships default `server.crt`/`server.key`.

## VSCode

Settings in `.vscode/settings.json` — uses Prettier and TypeScript from `node_modules`. Recommended extensions: `esbenp.prettier-vscode`.
