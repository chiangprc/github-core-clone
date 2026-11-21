# github-core-clone-01 — a GitHub-compatible forge

A standalone GitHub-compatible git forge in TypeScript for Node 22, standard library only,
no dependencies, no `git` binary, no network, no model calls. It serves the four surfaces
the contest scores — git smart HTTP, REST v3, GraphQL v4 and MCP over streamable HTTP —
behind the lifecycle `main.ts` drives (bind an ephemeral port, register it with the env,
poll `/done`, exit 0).

## Run it

```sh
node --experimental-strip-types main.ts
```

`MAKEFUN_ENV_URL` must name the env (the harness sets it); `MAKEFUN_SHOP_TOKEN` is the
bearer token every API call presents. The platform runs `node /bot/main.ts` on Node 22.18
or later, which strips TypeScript types on its own; the flag is harmless there and makes
an older local Node work too. There is nothing to install.

## Layout

- `main.ts`: the entry <-> env lifecycle. Keep it as it is.
- `app.ts`: the server. It owns routing, auth and the wire discipline, and assembles the
  four surfaces:
  - `GET /healthz` → 200 `{"ok":true}`, no token (the env's boot probe, called twice).
  - `GET /api/v3` and `GET /api/v3/` → the starter's API root, with
    `X-Oauth-Scopes: repo, read:org`.
  - `POST /api/graphql` → `graphql.ts` (GraphQL v4).
  - `POST /mcp` → `mcp.ts` (MCP over streamable HTTP: initialize, tools/list, tools/call).
  - `/api/v3/*` → `rest.ts` (REST v3: repos, branches, commits, contents, issues,
    comments, labels, pull requests, files, merges).
  - `/<owner>/<repo>.git/*` → `git.ts` (git smart HTTP: `info/refs`,
    `git-upload-pack`, `git-receive-pack`).
  - Anything else → 404 `{"message":"Not Found"}`.
- `git.ts`, `rest.ts`, `graphql.ts`, `mcp.ts`: one module per surface. Each exports a
  handler (`handle`, `default`, or `handleRest` / `handleGit` / `handleGraphql` /
  `handleMcp` / `*Handler`) called as `await handler(req, res, ctx)`; `ctx` carries the
  shared store, the expected token, a deterministic clock and the JSON sender, so state
  created on one surface is visible on the others (a repository created over REST is
  cloneable over git; an issue created over GraphQL or MCP is listed by REST). A module
  that is missing, fails to load, or exports no recognised handler does not stop the
  server: its surface answers 501 `{"message":"Not Implemented"}` and the reason is
  written to stderr.
- `package.json`, `tsconfig.json`, `LICENSE` (MIT): the entry's metadata and licence.

## Wire discipline (enforced in app.ts, not left to the modules)

- Every reply carries a non-empty `X-GitHub-Request-Id` (a per-process counter).
- Every JSON reply carries `content-type: application/json`.
- Every route except `GET /healthz` requires `Authorization: Bearer $MAKEFUN_SHOP_TOKEN`;
  a missing or wrong token is 401 `{"message":"Bad credentials"}`.
- Determinism: served state never uses `Date.now()`, `new Date()` or `Math.random()`.
  Timestamps come from `trialNow()` (`MAKEFUN_TRIAL_NOW`, fallback
  `2026-11-21T06:30:00.000Z`).

## Verify it locally (no docker)

The kit's own conformance runner grades a running server over HTTP:

```sh
node --experimental-strip-types <kit>/sdk/run.ts --url http://127.0.0.1:<port> --token "$TOKEN" --json [--perf]
```

`main.ts` exits 1 without `MAKEFUN_ENV_URL`, so a local run needs a stub env that
answers `/register` and `/done`; `../github-core-clone-01-harness/run-local.mjs` does
the whole thing in one command (stub env + entry + preflight + the suite):

```sh
node ../github-core-clone-01-harness/run-local.mjs [--perf]
```

## Status (2026-11-21, integration pass)

Routing, auth, the request-id / JSON discipline and the module assembly in `app.ts`
are in place and the entry boots. The four surface modules were still being written
when this README was updated, so until they land their surfaces answer 501; the
conformance result for each surface depends on its module. See the integration report
for the per-surface result.
