# njs — agent instructions

njs is a JavaScript engine integrated with NGINX. It ships as:

- a standalone CLI (`build/njs`) for testing and scripting,
- two NGINX modules: `ngx_http_js_module` and `ngx_stream_js_module`,
- two interchangeable JS engines selectable per location/server via the
  `js_engine` directive:
  - **njs** — the built-in engine, deprecated since 1.0.0.
  - **QuickJS** — recommended. Set `js_engine qjs;` in nginx.conf.

This file is the index. Detailed instructions live under [`docs/agent/`](docs/agent/).

## Repositories

- `origin` = `https://github.com/freeunitorg/njs.git` — fork, work happens here
- `upstream` = `https://github.com/nginx/njs.git` — official, read-only

## Key directories

| Path | Purpose |
|---|---|
| `src/` | Engine core — VM, parser, generator, builtins |
| `external/` | External modules — crypto, webcrypto, fs, query_string, xml, zlib |
| `nginx/` | NGINX module source + integration tests (`nginx/t/*.t`) |
| `test/` | JS unit tests (`.t.js`, `.t.mjs`), shell tests, test262 |
| `ts/` | TypeScript declarations for NGINX API (`ts/ngx_http_js_module.d.ts`) |
| `auto/` | Build system (configure, CC detection) |

## Git and branch convention

- Feature branches: `bump-YYYY-MM-DD` (sync from upstream + new features)
- Commit style: past tense, subject ≤ 67 chars, module prefixes: `Tests:`, `HTTP:`, `Core:`, `Stream:`, etc.
- `origin/master` = fork's master; `origin/bump-YYYY-MM-DD` = feature branch
- **Never commit or push without an explicit user instruction.**

## Pick your task

| If you are doing... | Read |
|---|---|
| Editing C in `src/`, `external/`, `nginx/` — engine, modules, build system | [docs/agent/engine-dev.md](docs/agent/engine-dev.md) |
| Writing JavaScript that runs in njs (CLI or NGINX), targeting either engine | [docs/agent/js-dev.md](docs/agent/js-dev.md) |
| Writing JavaScript that must run on the deprecated njs engine | [docs/agent/js-dev-njs.md](docs/agent/js-dev-njs.md) |

## 1. Engine and module development (C)

You are extending or fixing the engine, the QuickJS integration, or the
nginx modules.

Quick facts:

- **Build (CLI):** `./configure && make -j$(nproc) njs` → `build/njs`. Rebuild: `make njs` (no reconfigure).
- **Build (CLI with QuickJS):**
  ```bash
  (cd <QUICKJS_SRC> && CFLAGS=-fPIC make libquickjs.a)
  make clean && ./configure --cc-opt='-I<QUICKJS_SRC>' --ld-opt='-L<QUICKJS_SRC>' && make njs
  ```
- **Build (NGINX module, dynamic):**
  ```bash
  cd <NGINX_SRC>
  ./auto/configure --add-dynamic-module=<NJS_SRC>/nginx --with-stream
  make -j$(nproc) modules
  ```
- **Dual engine = dual code.** Modules ship both `njs_*.c` and `qjs_*.c`
  (also in `external/`). If you change behavior on one side, change the other.
- **Tests:** `make unit_test`, `make lib_test`, `make test262`. NGINX
  integration tests: `prove -I nginx/t/lib nginx/t/*.t`.
- **Code style and commits:** follow
  [docs/agent/engine-dev.md](docs/agent/engine-dev.md#code-style-and-commits)
  for formatting, warnings, and commit-log style.

Full details, sanitizer builds, VM architecture, and object model:
[docs/agent/engine-dev.md](docs/agent/engine-dev.md).

## 2. Writing JavaScript for njs (CLI or NGINX)

You are writing `.js` modules that run inside `js_content` / `js_filter` /
`js_set` / `js_access` / `js_preread` handlers, or under the standalone CLI.

Orientation:

- **Default to the QuickJS engine** (`js_engine qjs;`). The built-in njs
  engine is deprecated since 1.0.0; write new code for QuickJS.
- **Language baseline.** QuickJS is ES2023; the njs engine is ES5.1 strict
  with a curated ES6+ subset. See the
  [compatibility page](https://nginx.org/en/docs/njs/compatibility.html).
- **Nginx drives the engine, not the JS.** Code only runs from
  directive-bound entry points (HTTP: `js_content`, `js_access`,
  `js_header_filter`, `js_body_filter`, `js_set`, `js_periodic`).
- **Quick test (CLI):** `./build/njs -c '<code>'` or `./build/njs file.js`.
- **Test inside NGINX:** `prove -I <tests-lib> nginx/t/<your>.t` with
  `TEST_NGINX_GLOBALS_HTTP='js_engine qjs;'` (and the same for `_STREAM`).

Everything else — full integration-point semantics, `nginx.conf` wiring
(`js_shared_dict_zone`, `resolver` + `js_fetch_*`, `js_import` /
`js_path` / `js_engine`), bindings (`r`, `s`, `ngx.fetch`, `ngx.shared`,
`crypto`, …), engine-only features, do/don't recipes:
[docs/agent/js-dev.md](docs/agent/js-dev.md). For code that must run on
the deprecated njs engine, also see
[docs/agent/js-dev-njs.md](docs/agent/js-dev-njs.md).

## TypeScript

`ts/ngx_http_js_module.d.ts` — types for HTTP request API.
Compile check: `test/ts/test.ts` — exercises types, must compile cleanly.

## Important constraints

- `src/njs.h` defines `NJS_HAVE_QUICKJS` — guards all QuickJS code paths.
- NGINX version guards: `#if defined(nginx_version) && (nginx_version >= 1023000)` for newer APIs.
- All external modules must be registered in `nginx/ngx_js_modules.h` to be loaded.
- Compile with warnings as errors (`-Werror`).

## Resources

- [njs official documentation](https://nginx.org/en/docs/njs/)
- [Reference (API surface)](https://nginx.org/en/docs/njs/reference.html)
- [Compatibility](https://nginx.org/en/docs/njs/compatibility.html)
- [Engine selection](https://nginx.org/en/docs/njs/engine.html)
- [njs-examples repo](https://github.com/nginx/njs-examples/)
