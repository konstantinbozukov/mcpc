# AGENTS.md

Rules for any Cursor agent working in this checkout. Read this file before editing. Deeper design notes live in `CLAUDE.md` and `CONTRIBUTING.md`. When those disagree with the code, trust the code and `mcpc --help`.

## What this project is

`mcpc` (`@apify/mcpc`) is a universal command-line client for the [Model Context Protocol](https://modelcontextprotocol.io/). It maps MCP operations to shell commands (`tools-list`, `tools-call`, `resources-read`, and so on), keeps persistent sessions in a background bridge, and speaks Streamable HTTP and stdio.

This GitHub repo is **Konstantin Bozukov's fork**: `konstantinbozukov/mcpc`, forked from `apify/mcpc`. Checked on 13 November 2026:

- Fork `main` is commit `ab880f909662893accebe92c2b76408f2dfdf387` (10 September 2026). `package.json` version is `0.6.0`, with an `[Unreleased]` changelog.
- Upstream `apify/mcpc` `main` is **66 commits ahead** and `package.json` there is `0.8.0`, matching the npm package published 4 November 2026.
- The fork has no tags, no GitHub Pages, and no open issues. Upstream has Pages, releases, and the npm publish pipeline.
- `package.json` still names the author as Jan Curn and the repository as `apify/mcpc`. That metadata describes the upstream product.

Do not treat this tree as the current published CLI. Sync with upstream before fixing bugs that may already be released. TODO: confirm with Konstantin whether this fork should track `apify/mcpc` or diverge.

## Stack

- Language: TypeScript, ESM (`"type": "module"`), compiled by `tsc` to `dist/`.
- Runtime: Node.js `>=22.12.0` (verified here: Node `v22.23.4`). Bun `>=1` is a supported alternative runtime; it was **not** installed in this environment.
- Package manager: pnpm `10.33.4`, pinned by `packageManager` in `package.json`. Enable with `corepack enable && corepack prepare pnpm@10.33.4 --activate` if needed.
- CLI parsing: Commander. Human output uses chalk and ora. `--json` is the machine interface.
- MCP: `@modelcontextprotocol/client` and `@modelcontextprotocol/core` at `2.0.0` (exact pins). `@modelcontextprotocol/sdk` `1.30.0` is still used by `connect --proxy` and the e2e server.
- Credentials: `@napi-rs/keyring` (OS keychain), with a `0600` file fallback.
- Tests: Vitest 5 for unit tests, bash suites under `test/e2e/` for end-to-end.
- Lint: ESLint 10 (flat config, `src/**/*.ts` only) and Prettier 3 (`src` and `test` TypeScript).
- x402 payments: viem is a devDependency, bundled at build time by `scripts/bundle-viem.mjs` into `dist/lib/x402/viem.js`. Import viem only from `src/lib/x402/viem.ts`.

`pnpm-workspace.yaml` sets `minimumReleaseAge: 4320` (3 days). pnpm 10 enforces that only when resolving new versions. `pnpm run check:deps-age` checks the lockfile and is a release gate. Renovate (`renovate.json`) uses a 7-day `minimumReleaseAge`.

## Folder layout

```
bin/                 # mcpc and mcpc-bridge shims
src/cli/             # Commander program, commands, output, argument parsing
src/core/            # MCP client wrapper, transports, protocol versions
src/bridge/          # Long-lived bridge process and the local MCP proxy
src/lib/             # Sessions, config, logging, IPC, auth, x402
skills/mcpc/         # Agent skill printed by `mcpc help --skill`
test/unit/           # Vitest
test/e2e/            # Shell e2e suites and the local MCP test servers
test/conformance/    # Adapter for the upstream MCP conformance client
docs/                # REFERENCE.md (generated), examples, this audit
scripts/             # Release wrapper, reference generator, viem bundler
.github/workflows/   # CI, e2e matrices, conformance, npm release
```

User data is **not** in the repo. It lives in `~/.mcpc/` (`MCPC_HOME_DIR` overrides that): `sessions.json`, `profiles.json`, `bridges/`, `logs/`, and the keychain fallback files.

## Install, run, test, lint, build

Commands below were run in this environment on 13 November 2026 unless marked otherwise.

```bash
pnpm install --frozen-lockfile   # verified: 539 packages added
pnpm run build                    # verified: tsc, then viem bundle (686 KiB, 331 modules)
pnpm run test:unit                # verified after build: 43 files, 1081 tests, all passed
pnpm run lint                     # eslint src && prettier --check (not run in this audit)
pnpm run lint:fix                 # eslint --fix and prettier --write
pnpm run format                   # prettier --write only
```

`pnpm run test:unit` without a prior build fails one test: `test/unit/packaging.test.ts` expects `dist/cli/commands/help.js` inside `npm pack --dry-run`. CI builds first (`.github/workflows/ci.yml`). On the first attempt here, before `pnpm run build`: 1080 passed, 1 failed, for that reason only. After `pnpm run build`: 1081 passed, 0 failed.

End-to-end (build first, or pass `--no-build` when `dist/` is current):

```bash
pnpm run test:e2e                 # ./test/e2e/run.sh --keep  (full legacy matrix; not run here)
pnpm run test:e2e:modern          # same suites against the 2026-07-28 test server
pnpm run test:e2e:bun             # requires Bun; `bun` was not installed here
pnpm test                         # build + unit + legacy e2e + modern e2e + bun e2e
./test/e2e/run.sh --no-build --parallel 6 basic/help.test.sh basic/errors.test.sh basic/session-commands.test.sh
```

That last command was run here. All 3 suites passed (help 6.0s, errors 3.7s, session-commands 5.0s) on Node v22.23.4, legacy protocol server. The other 49 e2e suites, the modern-protocol matrix, the Bun matrix, coverage, and conformance were not run.

Local CLI after a build:

```bash
node bin/mcpc --help
pnpm link --global    # optional; documented in CONTRIBUTING.md, not run here
```

Other documented commands (not run in this audit): `pnpm run test:coverage`, `pnpm run test:conformance`, `pnpm run check:reference`, `pnpm run build:reference`, `pnpm run check:deps-age`, `pnpm run release`.

`pnpm install` printed that esbuild's build script was ignored (`onlyBuiltDependencies` lists only `@napi-rs/keyring`). The esbuild package still imported as `0.28.2`, and `pnpm run build` completed.

Some e2e suites log in to the real server `mcp.apify.com` and need profiles `e2e-test1` and `e2e-test2`. See `CONTRIBUTING.md`. Do not create those profiles against a personal account unless Konstantin asked.

## Deploy targets and how deploys happen

There is no application server, database, or container deploy in this repo.

| Target | How it happens |
| --- | --- |
| npm package `@apify/mcpc` | `pnpm run release` (or `:minor` / `:major` / `:pre`) runs `scripts/publish.sh`, which only checks preconditions and triggers `.github/workflows/release.yml` via `workflow_dispatch`. The workflow runs e2e (Windows, macOS, Linux), conformance, dependency-age, lint, build, and unit tests, then bumps the version, tags, and publishes to npm with provenance. Stable releases must come from `main`. Direct `npm publish` is blocked by `prepublishOnly` unless `MCPC_RELEASE` is set. |
| GitHub release | Same `release.yml` job, using `secrets.APIFY_SERVICE_ACCOUNT_GITHUB_TOKEN` and `secrets.NPM_TOKEN`. |
| Homebrew | Documented in `CLAUDE.md` as a **manual** `workflow_dispatch` in `apify/homebrew-tap`, not a job in this repo. TODO: confirm with Konstantin that the tap still tracks npm `@apify/mcpc` and not this fork. |
| GitHub Pages | `_config.yml` is Jekyll config for `https://apify.github.io/mcpc/` (README, `client-metadata.json` OAuth client identity, logo, LICENSE, `docs/`). Upstream `apify/mcpc` has Pages enabled. This fork's GitHub metadata says Pages is **off**. There is no Pages workflow file in the repo; publishing is the GitHub Pages Jekyll integration. TODO: confirm with Konstantin that he does not expect this fork to serve the CIMD document. |

Do not run `pnpm run release` from this fork. The workflow expects Apify's npm and GitHub tokens and publishes the `@apify/mcpc` name, which this fork does not own.

## External services

No Supabase, Vercel, or AWS application config exists in this repository (searched source, workflows, and docs).

Services the code and the release process actually use:

- **npm registry** for the published CLI.
- **GitHub Actions** for CI and release. CI (`.github/workflows/ci.yml`) runs on push to `main` and on pull requests: lint, build, unit tests on Node 22, 24, and 26; e2e on Node 24; a modern-protocol e2e job; Bun 1.3.x and 1.4.x e2e.
- **GitHub Pages** on the upstream repo, for the public README and the OAuth client-metadata document.
- **OS keychain** (Secret Service / libsecret on Linux, with a file fallback when no daemon is available).
- **Whatever MCP server the user connects to**, including that server's OAuth authorization server. Examples and some e2e tests use `mcp.apify.com`.
- **Base** public RPCs for experimental x402: `https://mainnet.base.org` and `https://sepolia.base.org` (`src/lib/x402/signer.ts`). A session with x402 enabled can sign and send real payments.
- **Homebrew tap** `apify/homebrew-tap` (documented, not verified in this audit).

## Environment variables

Names only. Never print values.

Process env the CLI and bridge read:

- `MCPC_HOME_DIR` — state directory, default `~/.mcpc`
- `MCPC_VERBOSE` — verbose logs when `1`, `true`, or `yes`
- `MCPC_JSON` — JSON output, same truthy values
- `HTTPS_PROXY`, `https_proxy`, `HTTP_PROXY`, `http_proxy`, `NO_PROXY` (and the lowercase proxy names the proxy module also reads)
- `NODE_TLS_REJECT_UNAUTHORIZED` — set to `0` inside the bridge when a session was connected with `--insecure`
- `APPDATA` — Windows path used while discovering Claude Desktop / VS Code config locations
- `MCPC_RELEASE` — must be set for `prepublishOnly` to allow npm publish; the release workflow sets it

Config files may contain `${VAR_NAME}` placeholders. Those names are chosen by the config author and are expanded from the process environment (`src/lib/config.ts`). That includes any secret the shell happens to have. See `docs/AUDIT.md`.

CI-only names: `GITHUB_STEP_SUMMARY`, `GITHUB_OUTPUT`, `GITHUB_ACTIONS`, `NODE_AUTH_TOKEN` (set from `secrets.NPM_TOKEN` in the release workflow).

Test servers (not product config): `PORT`, `PAGINATION_SIZE`, `LATENCY_MS`, `REQUIRE_AUTH`, `NO_TOOLS`, `NO_TASKS`, `NO_RESOURCES`, `NO_PROMPTS`, `WITH_SKILLS`, `SKILLS_NO_INDEX`, `WITH_OAUTH`, `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, `OAUTH_NO_METADATA`, `LEGACY_MODE`, `TARGET_URL`, `E2E_SERVER_PROTOCOL`, `E2E_PER_TEST_TIMEOUT`, `MCP_CONFORMANCE_SCENARIO`, `MCP_CONFORMANCE_CONTEXT`.

## Coding conventions inferred from the code

- TypeScript `strict` is on, including `noUnusedLocals`, `noUnusedParameters`, and `exactOptionalPropertyTypes` (`tsconfig.json`). CI typechecks with `pnpm run build`.
- ESLint: unused vars must be prefixed with `_`; explicit return types and `any` are warnings; `no-console` is off.
- ESM with `.js` specifiers in relative imports (NodeNext).
- CLI commands are hyphenated (`tools-list`, `resources-read`). Session commands are `mcpc @name <command>`.
- Duration names include units (`timeoutSecs`, `KEEPALIVE_INTERVAL_MILLIS`). The CLI flag `--timeout` and the mcp.json `timeout` field stay unsuffixed because they are external names.
- Large or rarely used dependencies are loaded with `await import(...)`. Do not re-export them from a barrel file.
- Do not log credential values. Log `present` or `MISSING`. Headers go to the bridge over the socket, not on the command line.
- Create `~/.mcpc` directories with `ensureDir()` (`0700`). Secret files use mode `0600`.
- Spawn processes with `execFile` / `spawn` and argument arrays.
- Human output may include a dim next-step hint. `--json` output stays machine-readable: no hints, errors as `{ error, code }` on stderr.
- Help text is the documentation. One short line per `.description()` and `.option()`. JSON shape goes in a `jsonHelp()` section. After help-text changes, run `pnpm run build:reference` and commit `docs/REFERENCE.md`. CI runs `pnpm run check:reference`.
- User-facing CLI changes also need a look at `skills/mcpc/SKILL.md` and, when a user would care, a short entry under `[Unreleased]` in `CHANGELOG.md`.
- Do not advertise a command that the negotiated protocol version cannot perform. Use `isModernProtocolVersion()` from `src/core/protocol.ts`.
- Examples in docs should use `mcp.apify.com` when they need a real remote server (`CONTRIBUTING.md`).
- Section titles in README and code use sentence case.

## Never touch

- Do not print, commit, or paste secrets, tokens, private keys, or `.env` values. `.env` and `.env.local` are gitignored. Do not add them.
- Do not read or modify a real `~/.mcpc` directory, the OS keychain, `credentials.json`, or `wallets.json`. Tests must use `MCPC_HOME_DIR` pointed at a throwaway directory. The e2e runner already does this.
- There is no database and no migration directory. Do not add a data store or a migration as part of an unrelated change.
- Do not edit, as a drive-by: `_config.yml`, `client-metadata.json` (published OAuth client identity on upstream Pages), `.github/workflows/release.yml`, `scripts/publish.sh`, or `pnpm-lock.yaml`.
- Do not run `pnpm run release`, `pnpm run release:minor`, `pnpm run release:major`, or `pnpm run release:pre` unless Konstantin explicitly asked to publish, and not from this fork.
- Do not put tokens, client secrets, or x402 private keys on a command line in examples. `mcpc x402 import <private-key>` takes the key as an argument today; do not copy that pattern into new docs.
- `docs/REFERENCE.md` is generated. Change the Commander help text, then regenerate. Do not hand-edit the captured help.
- An existing deep review, `docs/security-review-2026-09.md`, is a snapshot of commit `833e584`. Do not "fix" it by rewriting history. New findings belong in a new note, or in code, when a change was requested.

## Before you finish a code change

This audit PR is documentation only. For later code work, the repo's own rule is: `pnpm run lint`, then `pnpm run build`, then `pnpm run test:unit`, and fix failures before committing. Help-text changes also need `pnpm run build:reference`.
