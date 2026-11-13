# Repository audit

Audit date: 13 November 2026. Tree: `konstantinbozukov/mcpc` commit `ab880f909662893accebe92c2b76408f2dfdf387` (`package.json` version `0.6.0`). No application code was changed for this audit. Line numbers refer to that commit.

## Summary

This is Konstantin's public fork of [apify/mcpc](https://github.com/apify/mcpc), a production command-line client for the Model Context Protocol: persistent sessions, OAuth 2.1, stdio and Streamable HTTP, a local proxy, and experimental x402 payments. The implementation is mature (strict TypeScript, CI on three Node versions plus OS e2e matrices, a conformance gate, a dependency-age release gate, and a September 2026 security review). The fork itself is a stale snapshot. It was created 13 September 2026, last pushed the same day, and on 13 November 2026 it is 66 commits behind `apify/mcpc` `main`. npm `@apify/mcpc` is `0.8.0` (published 4 November 2026). Upstream `package.json` on `main` is also `0.8.0` and pins newer MCP SDK packages than this tree. Treat the product as production software and this checkout as production code that is two published releases behind.

## Stack and hosting

- TypeScript ESM, Node.js `>=22.12.0`, pnpm `10.33.4`. Optional runtime: Bun `>=1`.
- Libraries: Commander, chalk, ora, `@napi-rs/keyring`, `proper-lockfile`, `undici` 7.x, `@modelcontextprotocol/client` and `core` `2.0.0`, `@modelcontextprotocol/sdk` `1.30.0` (proxy and e2e server only). viem is bundled into the published CLI for x402 and is not a runtime dependency.
- Nothing in this repo is hosted as a web app. There is no Supabase project, Vercel project, AWS stack, Docker image, or database.
- The published artifact is the npm package `@apify/mcpc`. Upstream also serves a Jekyll GitHub Pages site at `https://apify.github.io/mcpc/` (`_config.yml`), including the OAuth client-metadata document. GitHub reports Pages **enabled** on `apify/mcpc` and **disabled** on this fork.
- State on a user's machine is `~/.mcpc/` plus the OS keychain.

## Deploy path

1. A maintainer on the upstream repo runs `pnpm run release` (patch), `pnpm run release:minor`, `pnpm run release:major`, or `pnpm run release:pre`.
2. `scripts/publish.sh` checks a clean branch and a logged-in `gh` CLI, then dispatches `.github/workflows/release.yml`.
3. That workflow runs Windows, macOS, and Linux e2e plus MCP conformance, then on Ubuntu: dependency-age check, lint, build, unit tests, README and REFERENCE regeneration for stable releases, version bump, git tag, npm publish with provenance, and a GitHub release. Stable releases are refused unless the ref is `main`. Checkout and the git push use `secrets.APIFY_SERVICE_ACCOUNT_GITHUB_TOKEN`. Publish steps set `NODE_AUTH_TOKEN` from `secrets.NPM_TOKEN`.
4. `prepublishOnly` exits unless `MCPC_RELEASE` is set, so a plain `npm publish` from a laptop fails.
5. Homebrew is a separate manual workflow in `apify/homebrew-tap`, described in `CLAUDE.md`. TODO: confirm with Konstantin that the tap still follows npm and not this fork.

This fork cannot honestly run that path: the package name is `@apify/mcpc`, the secrets belong to the Apify repo, and this `main` is not the commit npm `0.8.0` was built from. The `[Unreleased]` section in `CHANGELOG.md` describes fixes that upstream has since shipped inside `0.7.0` (23 September 2026) and `0.8.0`.

Pull-request CI (`.github/workflows/ci.yml`) is lint, build, reference check (Node 24 only), and unit tests on Node 22/24/26, plus Linux e2e for legacy and 2026-07-28 servers and Bun 1.3/1.4. It does not deploy.

## Environment variable names

Values were not printed. No `.env` file is committed. `.gitignore` ignores `.env` and `.env.local`.

Product:

- `MCPC_HOME_DIR`, `MCPC_VERBOSE`, `MCPC_JSON`
- `HTTPS_PROXY`, `https_proxy`, `HTTP_PROXY`, `http_proxy`, `NO_PROXY`
- `NODE_TLS_REJECT_UNAUTHORIZED` (forced to `0` in the bridge for `--insecure` sessions)
- `APPDATA` (Windows config discovery)
- `MCPC_RELEASE` (release workflow only)

GitHub Actions secret **names** referenced by `release.yml` (values are not in git): `APIFY_SERVICE_ACCOUNT_GITHUB_TOKEN`, `NPM_TOKEN`. The workflow also reads `GITHUB_STEP_SUMMARY`, `GITHUB_OUTPUT`, `GITHUB_ACTIONS`, and sets `NODE_AUTH_TOKEN`.

Config files expand arbitrary `${VAR_NAME}` strings from the environment (`src/lib/config.ts`). Those names are not a fixed list.

Test-only: `PORT`, `PAGINATION_SIZE`, `LATENCY_MS`, `REQUIRE_AUTH`, `NO_TOOLS`, `NO_TASKS`, `NO_RESOURCES`, `NO_PROMPTS`, `WITH_SKILLS`, `SKILLS_NO_INDEX`, `WITH_OAUTH`, `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, `OAUTH_NO_METADATA`, `LEGACY_MODE`, `TARGET_URL`, `E2E_SERVER_PROTOCOL`, `E2E_PER_TEST_TIMEOUT`, `MCP_CONFORMANCE_SCENARIO`, `MCP_CONFORMANCE_CONTEXT`.

## Test coverage and what was run

Layout: 43 Vitest files under `test/unit/` (threshold 70% branches/functions/lines/statements, enforced only when coverage is on). 52 shell suites under `test/e2e/suites/`. A conformance adapter in `test/conformance/`. Coverage and conformance were not run.

Environment: Node `v22.23.4`, pnpm `10.33.4`, Linux. `bun` is not installed. `pnpm install --frozen-lockfile` added 539 packages. pnpm warned that the esbuild install script was ignored; `import('esbuild')` still resolved `0.28.2`.

| Command | Result |
| --- | --- |
| `pnpm run test:unit` before any build | **Failed.** 43 files, 1080 passed, 1 failed. `test/unit/packaging.test.ts` expected `dist/cli/commands/help.js` from `npm pack --dry-run`. `dist/` did not exist. CI builds before unit tests. |
| `pnpm run build` | **Passed.** `tsc`, then viem bundle `686 KiB`, `331` modules. |
| `pnpm run test:unit` after that build | **Passed.** 43 files, 1081 tests, 0 failed. About 3.6s. |
| `./test/e2e/run.sh --no-build --parallel 6 basic/help.test.sh basic/errors.test.sh basic/session-commands.test.sh` | **Passed.** 3/3 on the legacy (2025-11-25) test server. Node v22.23.4. |
| `pnpm test` (full unit + legacy e2e + modern e2e + Bun e2e) | **Not run.** 49 e2e suites were not executed. Modern-protocol and Bun jobs were not executed. |
| `pnpm run lint` | **Not run.** |

`pnpm audit --prod` on this lockfile reported **37 vulnerabilities: 1 critical, 11 high, 22 moderate, 3 low**. Almost every path is inside `@modelcontextprotocol/sdk@1.30.0` (Express, ajv, hono, and so on) or the pinned `@modelcontextprotocol/client@2.0.0`. Notable advisories:

- Critical: `proxy-addr` via `@modelcontextprotocol/sdk` → `express` ([GHSA-jqcg-44mw-7w3h](https://github.com/advisories/GHSA-jqcg-44mw-7w3h)).
- High: `@modelcontextprotocol/client` `>=2.0.0 <2.2.0` can send OAuth credentials to an authorization server chosen by the MCP server ([GHSA-6qxp-vccf-f47h](https://github.com/advisories/GHSA-6qxp-vccf-f47h)). This tree pins `2.0.0`. The same advisory applies to `@modelcontextprotocol/sdk` `<1.31.0`.
- High, also via the v1 SDK: task results not bound to the creating session ([GHSA-22jm-h49p-29qw](https://github.com/advisories/GHSA-22jm-h49p-29qw), patched in `>=1.32.0`); request-size memory exhaustion ([GHSA-46jr-4wx8-v453](https://github.com/advisories/GHSA-46jr-4wx8-v453)).

Upstream `package.json` read on 13 November 2026 pins `@modelcontextprotocol/client` and `core` at `2.3.1` and `@modelcontextprotocol/sdk` at `^1.32.1`. Those ranges are past the patched versions named above. This audit did not re-run `pnpm audit` on upstream.

## Bugs and likely bugs

`docs/security-review-2026-09.md` reviewed commit `833e584` (2 September 2026). Several of its high findings are fixed in this tree and recorded in `CHANGELOG.md` `[Unreleased]`: Windows `cmd.exe` browser launch, and token refresh that stayed on the authorization server recorded at login. The items below were re-read in the current files. Upstream commit subjects after this fork include "Auto-discovery: skip project configs that read environment variables (#383)" and "Recover OAuth sessions from HTTP 401 and send the resource indicator on refresh (#398)". Those two are still present here and may already be fixed on `apify/mcpc` `main`. The other items were not re-checked upstream.

1. **Project configs expand every environment variable, and bulk connect sends HTTP entries with no extra confirmation.** `substituteString` replaces `${NAME}` with `process.env[NAME]` (`src/lib/config.ts:161-172`) in `url`, `command`, `args`, `env`, and `headers` (`src/lib/config.ts:117-143`). Auto-discovery reads `.mcp.json`, `mcp.json`, `.cursor/mcp.json`, `.vscode/mcp.json`, and others from the current directory (`src/lib/config.ts:311-317`). Bulk connect skips stdio entries unless `--stdio` is passed, and connects every remaining HTTP entry (`src/cli/commands/connect.ts:911-922`). A repo-local config can therefore put a shell secret into a request header or a hostname. The fully substituted object is also debug-logged (`src/lib/config.ts:97`), and `debug()` always appends to the bridge log file even when stderr is quiet (`src/lib/logger.ts:259-264`).

2. **Stdio `env` and `args` still ride on the bridge command line.** Only `headers` are stripped before the config is JSON-encoded into argv (`src/lib/bridge-manager.ts:184-191`). The bridge then writes `process.argv.slice(1)` into the log banner (`src/bridge/index.ts:477-479`). README says process arguments show no secrets (`README.md:1345`) and that credentials are never logged (`README.md:1354`). Those sentences oversell what the code does for stdio env values.

3. **x402 signs whatever the server asks and can approve an unlimited allowance.** `MAX_UINT256` is the Permit2 approval amount when the allowance is short (`src/lib/x402/signer.ts:45`, `src/lib/x402/signer.ts:469-483`). `amount`, `payTo`, `asset`, and `network` come from the server. There is no spending cap in `src/lib/x402/`. The RPC is hard-coded to `https://mainnet.base.org` (`src/lib/x402/signer.ts:119`). `mcpc x402 import <private-key>` takes the key as a positional argument (`src/cli/commands/x402.ts:428-431`), so it lands in shell history.

4. **OAuth `state` is generated and then deliberately not checked.** `src/lib/auth/oauth-flow.ts:797-803` adds a random `state` and the comment says PKCE makes verification unnecessary. The callback ports are the fixed set 13316, 31613, 16133. The same login deletes stored client registration before the new login succeeds (`src/lib/auth/oauth-flow.ts:730-735`). A cancelled login can leave a profile that can no longer refresh.

5. **Refresh still omits the RFC 8707 `resource` parameter, and non-401 failures are auth errors.** The refresh body is `grant_type` and `refresh_token` plus client authentication (`src/lib/auth/oauth-utils.ts:402-410`). Any status other than 400/401, including 429 and 5xx, throws `AuthError` (`src/lib/auth/oauth-utils.ts:425-432`). Upstream #398's subject says the resource indicator was added after this commit.

6. **The local MCP proxy is unauthenticated unless a token was configured, and a non-loopback bind skips the Host check.** Bearer comparison runs only when a token exists, and it uses `!==` (`src/bridge/proxy-server.ts:170-181`). Host validation runs only for loopback binds (`src/bridge/proxy-server.ts:113-126`). A missing `Origin` is allowed.

7. **"Is this PID alive?" is `process.kill(pid, 0)` on Unix, and stop sends SIGKILL after one second.** `src/lib/utils.ts:541-546`. `stopBridge` SIGTERMs, waits 1000 ms, then SIGKILLs (`src/lib/bridge-manager.ts:389-400`). After reboot, a recycled PID can point at a different process of the same user. The one-second wait is shorter than the bridge's own HTTP and stdio shutdown budget, so stdio children can be orphaned.

8. **`--schema` validation lists only the first page of tools.** `callTool` uses `client.listTools()` (`src/cli/commands/tools.ts:299`). The bridge method returns one page (`src/bridge/index.ts:1451-1454`). A tool on a later page is reported as not found and the call is refused. `tools-get` uses the all-pages path.

9. **Any non-TTY stdin is treated as piped arguments.** `hasStdinData` is `!process.stdin.isTTY` (`src/cli/parser.ts:632-633`). `tools-call` with no positional args then reads stdin to EOF (`src/cli/commands/tools.ts:281-283`). Under an agent harness that leaves stdin open, the command waits.

10. **`--json` is syntax-coloured when stdout is a TTY.** `formatJson` highlights whenever `process.stdout.isTTY` (`src/cli/output.ts:134-142`). A pseudo-terminal then receives ANSI codes inside JSON.

11. **`mcpc clean sessions` help text does not match the command.** Help says "Remove stale/crashed session records" (`src/cli/index.ts:824`). `cleanSessions` stops and deletes every session (`src/cli/commands/clean.ts:75-90`). The README example at `README.md:1321` matches the code. The agent skill allows `Bash(mcpc:*)` with no extra prompt (`skills/mcpc/SKILL.md:4`), so an agent can wipe live sessions from the help text alone. The skill also recommends `npx -y @apify/mcpc@latest` (`skills/mcpc/SKILL.md:17`), which is the unpinned install the dependency-age policy exists to avoid.

12. **`--insecure` is detected by a raw argv scan before Commander parses.** `process.argv.includes('--insecure')` (`src/cli/index.ts:63`) disables TLS verification for the process if that exact token appears anywhere on the command line, including as a tool-call value. It is persisted on the session and reapplied as `NODE_TLS_REJECT_UNAUTHORIZED=0` for the whole bridge (`src/lib/bridge-manager.ts:265`).

13. **Bridge log files are created with the process umask.** `createWriteStream` without mode `0600` (`src/lib/file-logger.ts:60`). Debug lines, including the ones in finding 1, are written there unconditionally.

`test/README.md` tells contributors to run `npm test` / `npm run test:unit`. The repo scripts are pnpm. That is a docs mismatch, not a runtime bug.

## Security risks

No production API key, cloud credential, or `.env` was found in git. Secret scanning is enabled on the fork (GitHub metadata). Two committed test fixtures contain key material. Paths only:

- `test/e2e/server/https-wrapper.ts` — private key for a self-signed **localhost** certificate used to exercise `--insecure`. It is a test fixture, and it is still a private key in the repository.
- `test/e2e/server/index.ts` and `test/e2e/server/index-v2.ts` — a default OAuth client-secret literal for the local test authorization server. The same fixture is repeated in `test/e2e/suites/basic/client-credentials.test.sh` and `test/e2e/suites/basic/client-credentials-token-endpoint.test.sh`.

Other risks, in the order they matter on this tree:

- Running this checkout as if it were `@apify/mcpc@0.8.0`. The pinned client `2.0.0` is inside GHSA-6qxp-vccf-f47h. The v1 SDK `1.30.0` pulls the critical `proxy-addr` advisory and the high SDK advisories listed above. Upstream `0.8.0` has moved the pins.
- Project-config environment expansion (finding 1). This is the realistic "cloned repo steals `GITHUB_TOKEN`" path, and it matches a class of bug upstream #383's title says was later restricted.
- x402 with no cap (finding 3) moves real funds on Base once a session was started with x402.
- Proxy bound to a non-loopback address with no token (finding 6) exposes the upstream authenticated session on the network over HTTP.
- PID reuse before `SIGKILL` (finding 7) can signal an unrelated process of the same user.
- README claims that HTTPS is enforced by auto-upgrade (`README.md:1352`). `normalizeServerUrl` adds `https://` only when the user omitted a scheme. An explicit `http://` URL to a non-local host is a separate path. The September review called this out; it was not re-tested here beyond reading that the claim is still in the README.
- `release.yml` checks out with the service-account token and the default credential persistence, so a token that can push to `main` is available while `pnpm install` and the build run third-party code. That is an upstream workflow property copied into this fork. Actions are pinned to moving tags (`actions/checkout@v6`, `actions/setup-node@v6`, `oven-sh/setup-bun@v2`, `apify/actions/pnpm-install@v1.1.2`), not commit SHAs. CI workflow files set no `permissions:` block.

## Dependency and staleness issues

`pnpm outdated` on 13 November 2026, this lockfile:

- Direct product pins that upstream has already moved: `@modelcontextprotocol/client` and `core` `2.0.0` (latest seen by outdated: `2.4.0`; upstream main: `2.3.1`), `@modelcontextprotocol/sdk` `1.30.0` (outdated offered `1.32.2`; upstream: `^1.32.1`), `@napi-rs/keyring` `2.0.0` (outdated: `2.1.2`; upstream: `2.1.2`).
- `undici` `7.29.1` versus an offered `8.12.4`. Stay on 7.x unless the comment in `src/lib/proxy.ts` is revisited: undici 8's dispatcher is incompatible with Node 22/24's built-in fetch. Upstream is `^7.30.1`, a 7.x bump, not 8.
- `qrcode-terminal` `0.12.0`. npm `time.modified` for that package is 25 June 2022. It is a tiny QR printer, and it is stale.
- Majors that upstream has also not taken: TypeScript `6.0.3` (offered `7.0.2`), ora `9.4.1` (offered `10.0.0`). pnpm itself printed that `12.18.0` exists; the repo pins `10.33.4` because pnpm 10's `minimumReleaseAge` behaviour is what `scripts/check-dependency-age.mjs` compensates for. Do not bump pnpm casually.
- Both `c8` and `nyc` are devDependencies next to `@vitest/coverage-v8`. Coverage of unit tests goes through Vitest.
- `package.json` has no `files` allowlist. `.npmignore` excludes `src/`, `test/`, `docs/`, and `scripts/`, so the npm README's relative links into `docs/` break for people reading the package. `skills/` is not ignored, which is required: `mcpc help --skill` reads `skills/mcpc/SKILL.md` from the install tree (`test/unit/packaging.test.ts`).
- Fork metadata: created and last pushed 13 September 2026, 0 stars, 0 open issues, no git tags. Upstream pushed 6 November 2026, 1101 stars, 34 open issues, 109 forks, package `0.8.0`.

## Top 5 improvements

Ranked by value to Konstantin against the effort of doing it on this fork.

1. **Fast-forward the fork to `apify/mcpc` `main`.** Value: highest. Effort: low if there are no local commits to keep (there are none on `main` today). That single update picks up 66 commits, the `0.7.0` and `0.8.0` releases, the project-config restriction in #383, the refresh `resource` fix in #398, and MCP SDK pins that are past the high advisories above. Re-fixing those bugs on this snapshot would duplicate work that is already published.

2. **Re-run `pnpm audit` and the September review's open items on upstream `0.8.0` before writing any security patch.** Value: high. Effort: low. This audit's bug list is true for `ab880f9`. Shipping a patch against a two-release-old tree, then merging upstream, will conflict and may revert upstream fixes.

3. **Put a cap and a receipt on x402, and stop taking the private key as an argument.** Value: high if anyone enables x402. Effort: medium. Still true in this tree (`src/lib/x402/signer.ts:469-483`, `src/cli/commands/x402.ts:428-431`). Upstream has since added settlement receipts (#399); a cap was not visible from the commit subject alone. TODO: confirm with Konstantin whether `0.8.0` already has a spend limit before designing one.

4. **Make `mcpc clean sessions` and the agent skill match the destructive behaviour.** Value: medium for anyone pointing an agent at `skills/mcpc/SKILL.md`. Effort: low. Either change the help line at `src/cli/index.ts:824` to say every session is deleted, or make the command delete only stale ones. Narrow `allowed-tools` in `skills/mcpc/SKILL.md:4` so `clean`, `x402`, and `connect --stdio` are not pre-approved. Check upstream first; the skill may already have changed.

5. **Pin GitHub Actions to commit SHAs and drop the long-lived checkout token during install.** Value: medium for whoever actually publishes. Effort: low, and it belongs on `apify/mcpc`, not only here. `release.yml` keeps `secrets.APIFY_SERVICE_ACCOUNT_GITHUB_TOKEN` in the git credentials from checkout through `pnpm install`. Set `persist-credentials: false` and pass the token only to the push and release steps. Add a `permissions:` block to `ci.yml`.

## Open questions for the owner

- Should `konstantinbozukov/mcpc` track `apify/mcpc`, or is a divergence intended? There are no local commits on `main`, and the fork is 66 commits behind.
- Do you want this fork to publish anything? `pnpm run release` targets `@apify/mcpc` and Apify's GitHub secrets. This fork has Pages turned off, so it does not serve `client-metadata.json`.
- Is the npm package you actually run `0.8.0` from the registry, this tree, or a global install from an older `npm install -g`?
- Are the localhost test private key (`test/e2e/server/https-wrapper.ts`) and the test OAuth client secret (`test/e2e/server/index.ts`, `test/e2e/server/index-v2.ts`) acceptable to keep in git? They are fixtures. Say if you want them rotated or generated at test start.
- Do you use x402 against Base mainnet? If you do, the missing spend cap on this tree matters immediately. If you only use upstream `0.8.0`, confirm whether that release added a cap.
- Who updates the Homebrew formula, and does it point at npm `@apify/mcpc`? `CLAUDE.md` says the bump is manual in `apify/homebrew-tap`. TODO: confirm with Konstantin.
- The September review (`docs/security-review-2026-09.md`) is still the long-form security note in this fork, and several of its line numbers have drifted. Do you want a follow-up review of upstream `0.8.0`, or is tracking upstream enough?
- `test/README.md` says `npm test`. Development is pnpm. Worth fixing only if you keep a divergent fork.
