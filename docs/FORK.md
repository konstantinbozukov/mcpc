# Fork operations (konstantinbozukov/mcpc)

This GitHub repository is a **tracking fork** of [apify/mcpc](https://github.com/apify/mcpc). It is maintained for Konstantin Bozukov's x402 / agent-economy work, not as a separate product release.

## Publishing

- **Never publish** `@apify/mcpc` to npm from this fork. The package name, release workflow secrets (`MCPC_RELEASE`, `NPM_TOKEN`, Apify's GitHub token), and Homebrew tap all belong to the upstream repo.
- **Never run** `pnpm run release` or dispatch `apify/mcpc`'s release workflow from here unless Konstantin explicitly requests a one-off experiment—and even then expect it to fail without Apify credentials.

## Contributor docs

- **Agent and architecture guide:** upstream `AGENTS.md` in this tree (synced from apify/mcpc). Do not replace it with fork-specific notes; use this file instead.
- **Historical audit:** `docs/AUDIT.md` (November 2026 snapshot of the pre-sync tree, updated with post-sync status).

## Open upstream work tied to this fork

| PR | Topic |
| --- | --- |
| [apify/mcpc#396](https://github.com/apify/mcpc/pull/396) | Fail closed when x402 `resource.url` is unreachable (branch `fix/x402-refuse-unreachable-resource` on this fork). **Do not modify that branch in a sync-only PR.** |
| [apify/mcpc#372](https://github.com/apify/mcpc/pull/372) | Local `--x402-max-amount` spend cap (upstream; addresses audit finding 3). |

Sync `main` from `apify/mcpc` `main` periodically; rebase the #396 branch onto the updated base when refreshing that PR.