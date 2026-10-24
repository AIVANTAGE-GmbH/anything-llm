# Node 18 → Node 24 migration report (AnythingLLM v1.16.1 fork)

Branch: `node24-v1.16.1`. It starts at upstream commit `35c58d89` (the upstream `v1.16.1` tag, which our fork does not carry).
Date of analysis: 2026-10-24. All testing was local. No images were pushed, and no remote environment or real credentials were used.

> **Applied to `master` (v1.19.0).** The Dockerfile commit was cherry-picked onto `master` (`a5bb8b9c`, v1.19.0) and the full verification was repeated there. See §10. Sections 1–9 describe the original v1.16.1 run.

## 1. Summary and recommendation

**GO for Node 24.22.1.**

- **Only the Dockerfile changes.** No application code or dependency changes were needed. Nothing was found broken on Node 24.
- **Encryption is unaffected.** `EncryptionManager` (`scryptSync` + `createCipheriv`/`createDecipheriv`, AES-256-CBC) uses no removed API.
  - Ciphertexts and single-user JWTs created under Node 18 decrypt under Node 24.
  - Ciphertexts written under Node 24 decrypt under Node 18, so a rollback is data-safe.
- **Flows and tests match the Node 18 baseline.** All 25 end-to-end flows pass on Node 24 arm64 and amd64, the same as the Node 18 baseline. The existing Jest suite passes 572/572 on both versions.
- **No runtime warnings.** `--trace-deprecation --trace-warnings` produced zero warnings during boot and all flows, on both Node 18 and Node 24.
- **Node 22.23.4 fallback was not needed.** Node 24 broke nothing, so the conditional Node 22 run (Part 3.5) was not triggered. Node 22.23.4 contains the same fix for CVE-2026-111497. NodeSource publishes `22.23.4-1nodesource1`, and the static findings below apply to Node 22 as well.

**Separate security issue, not caused by this migration:** the image ships **Chromium 119.0.6045.105** (Nov 2023) for Puppeteer link scraping. That is much older than the Node issue being fixed here. Fixing it needs a Puppeteer upgrade, which this task excludes. See §7.

### Corrections to the task brief

| Brief said | What was verified |
|---|---|
| Branch from tag `v1.16.1` | The tag exists upstream only. `git ls-remote` on Mintplex-Labs shows `v1.16.1 → 35c58d89`, which is already in our history. The branch was created from that commit. |
| "CVE-2026-111497 ('Node2Shell')" | The CVE is real: *"Node.js Brotli decoder out-of-bounds write permits remote code execution"*, high, affects 26.x/24.x/22.x, fixed in 24.22.1 / 22.23.4 ([official advisory](https://nodejs.org/en/blog/vulnerability/october-2026-security-releases)). The name **"Node2Shell" does not appear** in the advisory. The advisory is dated Oct 21. nodejs.org's `index.json` dates v24.22.1 to 2026-10-20. |
| `apt-get upgrade` gets `libbrotli1` security updates | Correct for `libbrotli1` itself, but **it does not fix the CVE**. Node statically bundles its own Brotli (1.2.0 in 24.22.1). `ldd /usr/bin/node` shows no libbrotli. System `libbrotli1` is used by Chromium (via libbrotlidec), libfreetype6, libcurl and libjxl. The upgrade was kept as instructed. `libbrotli1 1.1.0-2build2` was already the newest candidate. |
| `util.is*` removed | `util.isBoolean`/`isBuffer`/`isDate`/… (DEP0045–DEP0058) and `util.log` (DEP0059) reached **End-of-Life in v23.0.0**. They were runtime-deprecated in v22.0.0. `util.isArray` (DEP0044) and `util._extend` (DEP0060) are **only runtime-deprecated** on 24, not removed. |
| `punycode`, `new Buffer()`, `url.parse()` deprecated | On v24 these are **"Application (non-`node_modules` code only)"** deprecations (DEP0040, DEP0005, DEP0169). Calls from inside `node_modules` print no warning and still work. |
| "upstream 1.16.1 already fixed one undici v7 issue for Gemini" | **Unverified.** No such commit exists between `v1.16.0` and `v1.16.1`, and nothing in the history mentions undici. The Gemini custom `fetch` wrapper (`server/utils/agents/aibitat/providers/gemini.js:22`) is error logging from commit `315391c0` ("better Gemini errors"). Possible siblings of that bug class were still checked and tested (F-05). |
| Embed widget returns "a streamed response with citations" | In v1.16.1 the public embed stream **always sends `sources: []`** (`server/utils/chats/embed.js:205-224`). Sources are deliberately stripped from public embed responses (`server/models/embedChats.js:41-50`, so private document sources are not exposed to anonymous widget users). This is the same on Node 18 and 24. Retrieval and citations were verified server-side in `embed_chats.response`. |

## 2. Changes on the branch

| Commit | Change | Why |
|---|---|---|
| `33e29c11` | `docker/Dockerfile`: NodeSource `node_18.x` → `node_24.x` (both arch stages). `nodejs=24.22.1-1nodesource1` pinned. `node:18-slim` → `node:24.22.1-slim` (frontend stage). `apt-get upgrade -y` after the first `apt-get update` (both arch stages). `RUN node -v \| grep -qx 'v24.22.1'` in the common build stage. | Task Part 1. Diff: 9 insertions, 5 deletions. Users/UID, entrypoint, healthcheck and ports are unchanged. |
| *(this file)* | `NODE24_MIGRATION.md` | Report |

**Install method: NodeSource apt.** `24.22.1-1nodesource1` exists in `https://deb.nodesource.com/node_24.x` for both `binary-amd64` and `binary-arm64`. It was checked in the repo's `Packages` index and installed in both builds (`dpkg-query` → `24.22.1-1nodesource1`). The tarball + SHASUMS256 fallback was not needed.

**No compatibility fixes were required, so none were made.** `engines.node` in `server/`, `collector/` and the root (`>=18.12.1` / `>=18`) already allows 24, so it was left unchanged.

## 3. Findings

Severity applies to the Node 18 → 24 move only. "Verified" means checked against the official Node docs or changelog **and/or** reproduced in the built image.

| # | Sev | Location | Issue | Broke in | Evidence | Fix | Status |
|---|---|---|---|---|---|---|---|
| F-01 | INFO (checked CRITICAL item: clean) | `server/utils/EncryptionManager/index.js:14,56,72`; `collector/utils/EncryptionWorker/index.js:48,64` | Encryption uses `scryptSync` + `createCipheriv`/`createDecipheriv` (aes-256-cbc, random IV, `ct:iv` hex). `createCipher`/`createDecipher` (DEP0106, EOL v22.0.0) are **not used anywhere** in app code. | – | grep across server/collector/embed. Runtime: Node 24 derives the identical key (fingerprint `e716b5b8…` on both), decrypts Node 18 `EncryptionManager` + Telegram `encryptToken` ciphertexts and the Node 18 single-user JWT `p` claim. Node 18 decrypts Node 24 ciphertexts. | None. Ciphertext format unchanged. | Verified |
| F-02 | INFO | `server/utils/comKey/index.js:81` (`privateEncrypt`), `collector/utils/comKey/index.js:49` (`publicDecrypt`) | RSA PKCS#1 v1.5 padding. The Marvin-attack hardening (CVE-2023-46809) disabled `RSA_PKCS1_PADDING` **only for `privateDecrypt`**, already backported to v18.19.1/v20.11.1/v21.6.2. `publicDecrypt` is unaffected. | n/a (same on 18.20.8) | v24 `crypto.json` change history. Runtime round-trip server→collector passes on 24. Keys rotate every boot, so nothing is persisted. | None | Verified |
| F-03 | INFO | `exceljs@4.4.0` (`dist/exceljs.bare.js`) | `createDecipher` hit is the bundled *browserify* polyfill in the browser bundle. The Node entry (`main: ./excel.js`) does not use it. | – | `package.json` `main`/`browser` fields | None | Verified |
| F-04 | INFO | `youtubei.js@9.4.0` (collector, direct) `dist/src/platform/{deno,web}.js` | `import … assert { type: 'json' }` is a SyntaxError on Node ≥22 (import assertions dropped in v22.0.0, nodejs/node#52104). Under the `node` export condition, Node loads `platform/node.js` (ESM) or `bundle/node.cjs` (CJS), and neither file contains it. | v22.0.0 | `require("youtubei.js")` from `/app/collector` (as `collector/utils/extensions/YoutubeTranscript/YoutubeLoader/index.js:67` does) and `import()` both load `Innertube` on 24 | None | Verified |
| F-05 | LOW | `server/utils/boot/patchSdkTimeouts.js:46-48` (`setGlobalDispatcher(new Agent())`), `server/utils/collectorApi/index.js:2,27,131` (`dispatcher: new Agent()` on every collector call), `server/utils/AiProviders/ollama/index.js:153` | **The undici-v7 sibling risk.** Agents from the npm `undici@6.21.1` (transitive) are passed to Node's **bundled** fetch. That is undici 5.x on Node 18 and **7.30.1** on Node 24. Cross-major dispatcher mixing is the known undici v7 failure class. | v24 bundles undici 7 | Probe in both 24.22.1 images (arm64 + amd64): native fetch, per-request npm Agent, and global npm Agent each pass Brotli-encoded JSON (auto-decompressed) and SSE streaming bodies. Document upload, which goes through the collector `dispatcher` path, passes end-to-end. | None needed now. **Re-test if the transitive `undici` 6.x changes or Node bumps undici.** | Verified (runtime) |
| F-06 | INFO | `server/utils/agents/aibitat/plugins/create-files/docx/utils.js:212` | Reads `content-length` before downloading an image. On fetch auto-decompression the header is the *compressed* size. | – | The code re-checks `imageBuffer.length` after download (l.222), so the limit can't be bypassed | None | Verified (code review) |
| F-07 | INFO | `server/endpoints/utils/dockerModelRunnerUtils.js:46`, `lemonadeUtilsEndpoints.js:52` | `response.body.getReader()` stream consumption. The API is unchanged in undici 7. | – | SSE body iteration verified in the F-05 probe | None | Verified (equivalent path) |
| F-08 | LOW | `@langchain/core@0.1.61/0.2.36` `utils/env` | Node 21+ defines global `navigator` (userAgent `Node.js/24`), so `isJsDom()` flips **false → true**. `getEnv()` still returns `"node"` because `isNode()` is checked first. | v21.0.0 | Ran `getEnv()`/`isJsDom()` in both images: 18 → `node/false`, 24 → `node/true` | None | Verified |
| F-09 | INFO | `axios@1.6.8`, `@anthropic-ai/sdk`, `openai@4`, `cohere-ai`, `elevenlabs`, `@azure/core-util`, `jose`, `pdf-parse`, `officeparser`, `xlsx`, `mammoth` | `typeof navigator` detection. These checks combine it with `typeof window`/`document`, test `navigator.product === "ReactNative"` or `msSaveBlob`, or only parse the userAgent, so none switches to browser mode under Node. No app code or dependency feature-detects global `WebSocket`. | v21 (`navigator`), v22 (`WebSocket`) | Scan hit lines reviewed. PDF/DOCX ingestion and chat via OpenAI-compatible SDKs pass on 24. | None | Verified (code + runtime) |
| F-10 | LOW | `langsmith@0.1.21/0.1.68`, `@langchain/core@0.2.36` (callbacks/dispatch) | `AsyncLocalStorage`: since v24.0.0 it uses `AsyncContextFrame` by default, which can change context propagation in edge cases. Only used for LangChain tracing/callback context. No app code uses it. | v24.0.0 | `CHANGELOG_V24.md` (24.0.0 notable changes) | None. Watch if LangSmith tracing is ever enabled. | Verified (docs); not exercised at runtime |
| F-11 | INFO | `prisma@5.3.1`, `@prisma/client@5.3.1` (`generator-build/index.js:20444`) | `fs.promises.rmdir(dir, {recursive:true})`: DEP0147, runtime deprecation (warning only on 24, not removal) | v16 (runtime) | Not triggered by `prisma generate` at boot (zero warnings captured) | None | Verified |
| F-12 | INFO | `safer-buffer`, `debug@2.6.9`, `lodash`, `exceljs`, `@prisma/engines` | `process.binding()` (DEP0111, documentation-only). All uses are inside `try` or behind a feature check. | – | Source review. No warnings at runtime. | None | Verified |
| F-13 | INFO | `send@0.19.x` `index.js:1014` | `res._headers` (DEP0066, EOL v24.0.0) is only read when `res.getHeaderNames` is missing, which it never is on 24 | v24.0.0 | Code review. Static file/widget serving passes. | None | Verified |
| F-14 | INFO | bluebird, protobufjs, zod, onnx-proto, encoding-japanese, undici 5/6 | Scan hits for `util.isX(` are calls on each package's **own** `util` object, not `node:util` | – | e.g. `undici/lib/dispatcher/client-h1.js:6 const util = require('../core/util.js')`, `bluebird/js/release/catch_filter.js:3 var util = require("./util")` | None | Verified |
| F-15 | INFO | `tr46@0.0.3`/`whatwg-url@5` (via `node-fetch@2`) | `require("punycode")`: DEP0040 is application-only on 24, so no warning from `node_modules`. The module still exists. | – | deprecations.json v24 | None | Verified |
| F-16 | MEDIUM (config-dependent) | `server/utils/agents/aibitat/plugins/sql-agent/SQLConnectors/MSSQL.js:19-20,51`; `pg@8`, `mysql2@3` (no custom TLS in app code; `mysql2/lib/connection.js:350-353` passes user-provided `ssl.ciphers`/`minVersion`) | TLS: the Oct-2026 release fixes CVE-2026-106069 / -105483 / -106072 (hostname-verification bypasses, incl. "DNS SANs authenticate IP endpoints"). SQL-agent connections **by IP address** whose server certificate has only DNS SANs may now fail verification where they previously passed. Node bundled OpenSSL 3.0.x on 18 → 3.5.9 on 24. **Whether OpenSSL 3.5's default security level rejects legacy ciphers or keys on old DB servers is unverified.** | 24.22.1 / 22.23.4 (CVE fixes); OpenSSL 3.5 (24.x) | Advisory text. Not tested: no legacy DB servers are available locally. | None. **Test the actual SQL-agent targets.** Prefer hostnames that match the certificate. | Partly verified (advisory), OpenSSL default unverified |
| F-17 | LOW | `collector/utils/extensions/Confluence/ConfluenceLoader/index.js:72,84` | Sets `process.env.NODE_TLS_REJECT_UNAUTHORIZED = "0"` process-wide when "bypass SSL" is chosen. Same behavior on 18 and 24. Node still prints its insecure-TLS warning when this is active. | – | Code review | None (pre-existing behavior) | Verified |
| F-18 | INFO | `docker/Dockerfile` frontend stage | Yarn 1.22.22 (inside `node:24.22.1-slim`) prints `DEP0169 url.parse()` during `yarn install`. Build-time only, not in the runtime image. | v24 (application deprecation) | build log `#12 0.307` | None | Verified |
| F-19 | INFO | `server/package.json:13`, `collector/package.json:13`, entrypoint, Dockerfile | Only flag in use is `--trace-warnings` (dev scripts). No `NODE_OPTIONS`, `--experimental-*`, `--openssl-legacy-provider` or `--security-revert` anywhere. | – | grep | None | Verified |
| F-20 | INFO | `require(esm)` (on by default since v22.12.0) | No installed package uses the `module-sync` exports condition, and the only `ERR_REQUIRE_ESM` handler is in `marked/bin/main.js` (CLI, not used at runtime). The app is CJS. | v22.12.0 | Scan of all `package.json` `exports`. Boot is clean. | None | Verified |
| F-21 | INFO | Streams | Default `highWaterMark` raised 16 KiB → 64 KiB (v22.0.0, nodejs/node#52037). App code never sets or depends on chunk sizes (`server/utils/files/index.js:813` reads line-by-line). SSE writes are explicit `response.write` per chunk. | v22.0.0 | Streaming chat (34 chunks) and embed stream identical on 18 and 24 | None | Verified |
| F-22 | INFO | `engines` | 1048 (server) + 556 (collector) installed packages checked with `semver.satisfies("24.22.1", engines.node)`. **0 exclude Node 24.** Only `pnpm`/`bare` engine-name warnings from yarn. | – | scan output | None | Verified |
| F-23 | INFO | Prisma 5.3.1 | `engines.node >=16.13`. No `binaryTargets` in `schema.prisma`, so the native engine is used. OpenSSL detection picks `linux-arm64-openssl-3.0.x` / `debian-openssl-3.0.x` (system libssl3t64 3.0.13). Prisma's official support matrix for 5.3.1 on Node 24 is **unverified**. Empirically, `generate`, `migrate deploy` (20 pending migrations on an older DB) and all queries work. | – | runtime | None | Empirically verified |

Checklist items with **no hits in app code** (server, collector, embed `src`, docker): import assertions, `util.is*`, `punycode`, `url.parse`, `new Buffer`, `process.binding`, `tls.createSecurePair`, `fs.rmdir` recursive, `fs.Stats` constructor, `AsyncLocalStorage`, `typeof WebSocket/navigator`, `highWaterMark`, `zlib.*`, `child_process` with `shell:true` + args (DEP0190), and non-string/Buffer `fs.writeFile*` data (DEP0162).

## 4. Native module inventory

All addons are **Node-API (ABI-stable)**. None is nan or raw-V8, and nothing compiles from source in the image. Every addon was loaded **and exercised** on Node 24.22.1 (modules 137, N-API 10) on linux-arm64 and linux-x64.

| Package (where) | Version | Binding type | Binary source | arm64 | x64 |
|---|---|---|---|---|---|
| `@prisma/engines` / `prisma` / `@prisma/client` (server) | 5.3.1 | Node-API library engine (`libquery_engine-*.so.node`) | downloaded at postinstall | `linux-arm64-openssl-3.0.x` ✔ | `debian-openssl-3.0.x` ✔ |
| `@lancedb/lancedb` (server) | 0.15.0 | napi-rs (per-platform package `@lancedb/lancedb-linux-*-gnu`) | npm prebuilt | ✔ create/search table | ✔ |
| `@mintplex-labs/mdpdf` (server) | 0.1.9 | napi-rs (`@mintplex-labs/mdpdf-linux-*-gnu`) | npm prebuilt | ✔ (also used to generate the test PDF) | ✔ |
| `canvas` (server) | 3.2.1 | node-addon-api, `binary.napi_versions [7]` | `prebuild-install -r napi` prebuilt (built upstream on Node 21.7.3, napi 9). No `.o` files in the image. | ✔ draw + PNG | ✔ |
| `sharp` (server; collector via `@xenova/transformers`) | 0.32.6 | node-addon-api, napi 7 + bundled libvips 8.14.5 | prebuild-install prebuilt | ✔ resize | ✔ |
| `sharp` → `@img/sharp-linux-*` (collector) | 0.33.5 | Node-API, libvips 8.15.3 | npm optional prebuilt | ✔ | ✔ |
| `onnxruntime-node` (server + collector, via `@xenova/transformers` 2.17.2) | 1.14.0 | Node-API v3 (`bin/napi-v3/linux/{arm64,x64}`) | shipped in the npm tarball | ✔ binding loads (model inference not run) | ✔ |
| `bcrypt` | – | not used (`bcryptjs`, pure JS) | – | – | – |

## 5. Runtime warnings captured

`NODE_OPTIONS="--trace-deprecation --trace-warnings"` was set on the container and confirmed in `/proc/<pid>/environ` for both `node /app/server/index.js` and `node /app/collector/index.js`. A probe `process.emitWarning` printed with a stack trace, so warnings would have been visible.

| Run | Warnings |
|---|---|
| Node 18.20.8 arm64: boot + all flows | **none** |
| Node 24.22.1 arm64: boot (Node 18 data) + all flows | **none** |
| Node 24.22.1 arm64: boot on older DB (20 pending migrations) | **none** |
| Node 24.22.1 amd64 (QEMU): boot + all flows | **none** |
| Jest, Node 18 and Node 24 | `MaxListenersExceededWarning: 11 abort listeners` ×2 on **both** versions, from a test fixture. Not Node-24 specific. |
| Image build, frontend stage | `DEP0169 url.parse()` from yarn 1.22.22 (build-time only, F-18) |

## 6. Test and flow results

### Existing test suites (Jest 29.7.0, the locked version, run inside each image over `/app/server/__tests__` + `/app/collector/__tests__`)

| | Suites | Tests | Time |
|---|---|---|---|
| Node 18.20.8 (baseline) | 44/44 | 572/572 | 4.9 s |
| Node 24.22.1 | 44/44 | 572/572 | 3.5 s |

### End-to-end flows

**Test setup:**
- **Images:** `anythingllm:node18-arm64` (unmodified v1.16.1 Dockerfile) as the baseline, and `anythingllm:node24-{arm64,amd64}` from this branch.
- **Vector DB:** local `pgvector/pgvector:pg16` (pgvector 0.8.7).
- **LLM and embedder:** a dependency-free OpenAI-compatible mock serving `/models`, streaming `/chat/completions` and deterministic 384-dim `/embeddings`. It Brotli-compresses JSON when the client advertises `br`.
- **Config:** a dummy `.env` (generic-openai LLM + embedder, `VECTOR_DB=pgvector`).
- **Encryption keys:** `SIG_KEY`/`SIG_SALT` were left unset, so the **Node 18 image generated them**, as in production.

| Flow | Node 18 arm64 | Node 24 arm64 | Node 24 amd64 |
|---|---|---|---|
| Boot, `prisma generate` + `migrate deploy` on Node 18 data | ✔ | ✔ (no pending) | ✔ |
| Boot on **older SQLite DB** (first 20 of 40 migrations, up to `20240510032311`, created by Node 18 + seeded row) | – | ✔ 20 migrations applied, 40/40 recorded, `legacy-ws` row intact | – |
| Single-user login (AUTH_TOKEN) → enable multi-user → admin login → create API key | ✔ (seed) | – | – |
| Admin login with password hashed under Node 18 | ✔ | ✔ | ✔ |
| API key created under Node 18 still authenticates | ✔ | ✔ | ✔ |
| Create API key / authenticate with it | ✔ | ✔ | ✔ |
| **Encryption:** scrypt key identical / decrypt Node 18 `EncryptionManager` ciphertext / decrypt Node 18 Telegram token / decrypt Node 18 single-user JWT `p` | ✔ | ✔ | ✔ |
| Encryption rollback: Node 18 decrypts ciphertexts written by Node 24 | ✔ | – | – |
| comKey RSA `privateEncrypt`→`publicDecrypt`→`EncryptionWorker`, sign→verify | ✔ | ✔ | ✔ |
| Upload PDF, DOCX, TXT (collector) | ✔ | ✔ | ✔ |
| Embed into workspace → rows in PGVector (pdf 1, docx 1, txt 3) | ✔ | ✔ | ✔ |
| Workspace streaming chat (SSE, 34 chunks) + citations (`sample.pdf, sample.docx, sample.txt`) | ✔ | ✔ | ✔ |
| Embed widget script `/embed/anythingllm-chat-widget.min.js` served (650 KB) | ✔ | ✔ | ✔ |
| Create embed → `POST /api/embed/:uuid/stream-chat` streamed reply | ✔ | ✔ | ✔ |
| Embed citations: none sent to the public client (by design); sources persisted in `embed_chats` | ✔ | ✔ | ✔ |
| Un-embed DOCX → its PGVector rows **deleted** (1 → 0), other docs' rows untouched | ✔ | ✔ | ✔ |
| Delete DOCX from document storage | ✔ | ✔ | ✔ |
| Delete workspace → **all** its PGVector rows removed (`count = 0`) | ✔ | ✔ | ✔ |
| Collector link scrape (Puppeteer → `https://example.com`), run once, scraped doc removed | ✔ | ✔ | ✔ |
| **Total** | **25/25** | **25/25** | **25/25** |

**Probes run in the Node 24 images (both arches):**
- **Native addons:** every module in §4 loaded and exercised.
- **undici:** the F-05 matrix (6/6 pass).
- **`youtubei.js`:** loads via both `require` and `import`.

**Not tested (limitations):**
- **Real LLM/embedding providers.** The production embedding model was not used, by design, so the embedding model and vector schema stay untouched. Only the mock's 384-dim vectors were stored, in a throwaway pgvector container.
- **ONNX inference.** The `onnxruntime-node` binding was loaded, but no model was run.
- **The widget in a real browser.** This is browser-side JS, unaffected by the server's Node version. The served script and the exact endpoint it calls were tested.
- **Node 22.23.4.** Not run, since Node 24 had no failures.
- **Native amd64 hardware.** amd64 was tested under QEMU on an arm64 Mac, not on native hardware.
- **Link scraping:** there is no persistent toggle for `upload-link`. It was run once and nothing was enabled, so there was nothing to switch off afterwards.

## 7. Security inventory

| Component | arm64 image | amd64 image |
|---|---|---|
| Node.js | 24.22.1 (`nodejs 24.22.1-1nodesource1`) | 24.22.1 (`nodejs 24.22.1-1nodesource1`) |
| Bundled Brotli (`process.versions.brotli`, the CVE-2026-111497 component) | 1.2.0 | 1.2.0 |
| Bundled OpenSSL | 3.5.9 | 3.5.9 |
| Bundled undici / zlib | 7.30.1 / 1.3.2.1-motley-285e94b | same |
| `modules` / `napi` | 137 / 10 | 137 / 10 |
| V8 / libuv / c-ares / nghttp2 | 13.6.233.17-node.58 / 1.52.1 / 1.34.9 / 1.70.0 | same |
| System `libbrotli1` | 1.1.0-2build2 (newest available after `apt-get upgrade`) | 1.1.0-2build2 |
| System OpenSSL (`libssl3t64`) | 3.0.13-0ubuntu3.16 | 3.0.13-0ubuntu3.16 |
| Chromium for Puppeteer | **Chromium 119.0.6045.105**, prebuilt `chromium-1088-linux-arm64.zip` from `webassets.anythingllm.com` (`PUPPETEER_EXECUTABLE_PATH=/app/chrome-linux/chrome`) | **Chrome for Testing 119.0.6045.105**, downloaded by `puppeteer@21.5.2` to `/app/.cache/puppeteer/chrome/linux-119.0.6045.105` |
| Base images | `ubuntu:noble-20251013` (`sha256:c35e29c9…`), `node:24.22.1-slim` (`sha256:55fb5f5a…`) | same |

**Before (Node 18 baseline):** Node 18.20.8 has no fix for the Oct-2026 CVEs. The advisory lists the 26/24/22 lines only.

**Brotli decode paths (item 15):**

| Path | Direction | Library / Brotli implementation | Reachable how |
|---|---|---|---|
| Global `fetch` (bundled undici 7.30.1) auto-decompression | outbound responses | Node `zlib` (bundled Brotli) | Native fetch sends `accept-encoding: br` by default. The mock logged `ua=node accept-encoding=br` from the server's model-list fetch. Any upstream (LLM provider, model-map/pricing sync, web-browsing agent) can send `br`. **This is the main reachable path for CVE-2026-111497, and 24.22.1 fixes it.** |
| npm `undici@6.21.1` (server), `undici@5.28.4` (`@qdrant/js-client-rest`), `undici@5.29.0` (collector) fetch | outbound responses | Node `zlib` | Used when those packages' own fetch is called |
| `node-fetch@2.7.0` (OpenAI SDK v4 transport, langchain, etc.) | outbound responses | Node `zlib.createBrotliDecompress` | Advertises only `gzip,deflate` (observed `ua=OpenAI/JS`), but decodes `br` if a server sends it anyway |
| `axios@1.6.8` | outbound responses | Node `zlib` | Advertises `br` |
| `decompress-response@6.0.0` | outbound responses | Node `zlib` | via `simple-get` (prebuild-install, build-time) |
| `body-parser` | **inbound** request bodies | – | The main server and collector resolve **body-parser 1.20.4** (`server/index.js:66-69`, `collector/index.js:37-39`), which only inflates `gzip`/`deflate` and rejects `br`. The public embed endpoints therefore do **not** decode inbound Brotli. `body-parser@2.2.1` (supports `br`) exists only nested under `@modelcontextprotocol/sdk` (Express 5 server transport), which AnythingLLM does not use. |
| `sharp@0.32.6` install script, Prisma engine download | build-time | Node `zlib` | Image build only |
| Chromium 119 (Puppeteer link scraping) | outbound responses | Chromium network stack + system `libbrotlidec` (linked on both arches) | Every scraped page. **Not covered by the Node fix. See below.** |

**Separate issue, out of scope:** Chromium 119 dates from November 2023. It renders attacker-controlled pages during link scraping and is far behind current Chrome security fixes. Updating it means upgrading `puppeteer` in `collector/` (and the arm64 Chromium zip), which is a dependency upgrade. **Not done. Needs your approval.**

## 8. Commands

```bash
# branch (upstream v1.16.1 = 35c58d89; the fork has no tags)
git fetch origin && git checkout node24-v1.16.1
git submodule update embed          # embed pinned at 7e5c6af for v1.16.1

# build (repo root)
docker build --platform linux/arm64 -f docker/Dockerfile -t anythingllm:node24-arm64 .
docker build --platform linux/amd64 -f docker/Dockerfile -t anythingllm:node24-amd64 .
# (or both at once with buildx: --platform linux/amd64,linux/arm64)

# verify the runtime
docker run --rm --entrypoint node anythingllm:node24-amd64 -e 'console.log(process.versions)'

# local test stack (dummy values only)
docker network create allm-test
docker run -d --name allm-pg --network allm-test \
  -e POSTGRES_USER=allm -e POSTGRES_PASSWORD=dummy-pass -e POSTGRES_DB=allm pgvector/pgvector:pg16
docker run -d --name allm-mock --network allm-test -v "$PWD/mock:/mock:ro" -w /mock node:24.22.1-slim node mock-llm.js

# run (same as the official docker run, plus trace flags)
mkdir -p ./storage && touch ./storage/.env   # dummy env: generic-openai -> http://allm-mock:8089/v1,
                                             # VECTOR_DB=pgvector, PGVECTOR_CONNECTION_STRING=postgresql://allm:dummy-pass@allm-pg:5432/allm
docker run -d --name anythingllm --network allm-test -p 3001:3001 --cap-add SYS_ADMIN \
  -v "$PWD/storage:/app/server/storage" -v "$PWD/storage/.env:/app/server/.env" \
  -e STORAGE_DIR=/app/server/storage \
  -e NODE_OPTIONS="--trace-deprecation --trace-warnings" \
  anythingllm:node24-amd64
curl -f http://localhost:3001/api/ping
```

The test harness (mock LLM, flow script, encryption canary, native/undici probes, dependency scanner) was kept outside the repo, in the session scratchpad. It can be added under `extras/` on request.

## 9. Open items needing a decision (nothing done)

1. **Chromium 119 / `puppeteer@21.5.2`.** This is the outstanding security exposure in the image (§7), and fixing it requires a dependency upgrade.
2. **Pin the frontend base image by digest.** `node:24.22.1-slim@sha256:55fb5f5a527ea2c5c4a95f963e48ef90aa38c7b346054fdc8ee9957a2b55d205` is optional hardening. The tag is mutable, but `nodejs` in the runtime stages is pinned exactly.
3. **SQL-agent TLS (F-16).** Test the real target databases, especially ones reached by IP address or older servers.
4. **Production rollout.** Back up `/app/server/storage` (SQLite + `.env` with `SIG_KEY`/`SIG_SALT`) first. Rollback to the Node 18 image is data-compatible: migrations are unchanged, and ciphertexts work in both directions.

## 10. Retest on `master` (v1.19.0)

The Dockerfile change cherry-picked cleanly onto `master` (`a5bb8b9c`, 265 commits after v1.16.1). Master still used Node 18 (`node_18.x`, `node:18-slim`). Every check from §5–§7 was repeated with images built from master: `m-node24-{arm64,amd64}`, plus an unmodified master Dockerfile as the `m-node18-arm64` baseline.

**What changed between v1.16.1 and master that matters here:**
- **App code static rescan:** still no hits for any checklist item.
  - New: `server/utils/files/index.js:766` calls `fs.rmdirSync` on an empty directory, without `recursive`. That's fine.
  - New: `server/utils/AiProviders/llmman/index.js:198-213` uses the same npm-undici `Agent` pattern as F-05.
- **Unchanged since v1.16.1:** EncryptionManager, EncryptionWorker and both comKey modules.
  - `patchSdkTimeouts.js` changed only its SDK retry seeding. The undici dispatcher code is unchanged.
- **Dependency changes:** `@mintplex-labs/mdpdf` 0.1.10 (napi-rs, loads and runs on 24), `@qdrant/js-client-rest` 1.18.0 (nested undici 6.28.1), and `weaviate-client` 3.9.0 (nice-grpc / grpc-js 1.14.6), which replaces `weaviate-ts-client`. Both new clients load on 24.
  - New scan hits are the same false-positive classes as F-09/F-12/F-14: package-local `util`, a guarded `new Buffer`, and a constant cipher list.
  - Still 0 `engines` violations and no `module-sync` packages.

| Check | Node 18 master (baseline) | Node 24 master arm64 | Node 24 master amd64 (QEMU) |
|---|---|---|---|
| Runtime versions | 18.20.8 | 24.22.1 / Brotli 1.2.0 / OpenSSL 3.5.9 / modules 137 / napi 10 / undici 7.30.1 | same. `libbrotli1` 1.1.0-2build2, Chrome for Testing 119.0.6045.105 |
| Native addon probe (§4 list) | – | 10/10 | 10/10 |
| undici dispatcher probe (F-05) | – | 6/6 | – |
| Encryption: Node 18 ciphertexts + Node 18 JWT decrypt on 24; comKey round-trip | seeded | 6/6 | 6/6 |
| Rollback: Node 18 decrypts Node 24 ciphertexts | 3/3 | – | – |
| Older SQLite DB (first 20 migrations, seeded row) → boot | – | 20 applied, 40/40 recorded, `legacy-ws` intact | – |
| End-to-end flows (§6) | 25/25 | 25/25 | 25/25 |
| Runtime warnings with `--trace-deprecation --trace-warnings` | 0 | 0 | 0 |
| Jest (88 suites / 2098 tests) | 2098/2098 | 2098/2098 (×2) | – |

**Jest note:** the first full Jest run on Node 24 timed out once, in `server/__tests__/utils/AiProviders/embeddedLemonade/engine.test.js` › "leaves lemond's pick alone without an NVIDIA GPU". That run overlapped with the amd64 QEMU build, which saturated the CPU.
- Rerunning that suite alone gave 5/5 passes on both Node 18 and Node 24.
- Two further full runs on Node 24 gave 2098/2098 each.
- Classified as a load-dependent flaky test, not a Node 24 regression.

## 11. Addendum: bundled root-CA store (found after the master push)

| Sev | Location | Issue | Evidence | Fix | Status |
|---|---|---|---|---|---|
| MEDIUM (deployment-dependent) | All outbound TLS from Node: global `fetch`, provider SDKs, and `pg` for PGVector with `sslmode=require` (`server/utils/vectorDbProviders/pgvector/index.js:115`) | Node's bundled root store shrank from **150 roots (18.20.8) to 118 (24.22.1)**. 42 roots were removed, among them DigiCert Global Root CA, DigiCert High Assurance EV, Baltimore CyberTrust, GlobalSign Root CA, Comodo AAA Certificate Services, Go Daddy / Starfield Class 2, the Entrust roots, GTS Root R2, and the emSign / eMudhra roots. A server whose chain can *only* reach one of the removed roots fails verification on Node 24. With `pg` 8.11.5, `sslmode=require` maps to `ssl: {}`, which means full verification. | `tls.rootCertificates` diffed between the two images. Handshake test from both images, 2026-10-24: OpenAI, Anthropic, Google Generative Language, Mistral, Groq, OpenRouter, Together, Cohere, DeepSeek, AWS Bedrock (eu-central-1) and northflank.com / api.northflank.com all verify on 24.22.1. Google-fronted APIs chain via GTS Root R4, Northflank via ISRG Root X1. | None needed for those endpoints. For private endpoints, e.g. a managed Postgres addon, confirm the issuer, or supply the CA via `NODE_EXTRA_CA_CERTS`. | Verified. Our Northflank Postgres addon (`primary.anythingllm-v2-db--….addon.code.run:5432`) presents a Let's Encrypt **YR1** certificate. Its SAN `*.anythingllm-v2-db--rq7jmmv9zl9d.addon.code.run` matches the host. The chain YR1 → Root YR → ISRG Root X1 is the same one verified on 24.22.1 for northflank.com. `sslmode=require` keeps working. |

The frontend build stage is now pinned to `node:24.22.1-slim@sha256:55fb5f5a527ea2c5c4a95f963e48ef90aa38c7b346054fdc8ee9957a2b55d205` (multi-arch index digest).
