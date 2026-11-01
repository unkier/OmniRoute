# What upstream requires of a PR that adds a per-node setting

Research for unkier/OmniRoute#5 (child of the map unkier/OmniRoute#2, "reported cost rate for
provider nodes").

- **Question:** besides the code, what must a `cheaper-inference/OmniRoute` PR that adds a
  provider-node setting contain to pass the gates and match maintainer conventions?
- **Measured on:** `release/v3.8.53` @ `139070680f` (2026-11-01). The upstream branch tip was the
  same SHA that day.
- **Sources:** the repo's gate scripts, workflows, baselines and contributor docs; the template
  commit `831ea040c3` (upstream PR #12590); and recent merged upstream PRs that add node fields or
  migrations (#16961, #16087, #16877, #14817, #13580, #14223, #16350, #16310). Everything upstream
  was read only.

## TL;DR

A per-node setting PR touches three gated surfaces besides the code. **(1) i18n:** every new
`en.json` key must be in all 66 other locale catalogs and actually translated. English fallback is
not accepted, and three blocking gates check it. **(2) Docs:** nothing checks the new API field
itself, but the new migration is gated. The migration count must be bumped in `README.md`,
`AGENTS.md`, `llm.txt`, the 66 translated READMEs and the 66 byte-identical `llm.txt` mirrors.
`AGENTS.md` and `llm.txt` are protected surfaces that need owner approval. **(3) Process:** add a
`changelog.d/` fragment and never edit `CHANGELOG.md`. Use the next free migration number (223
today); expect it to be renumbered at merge if the tip moves. Put tests under `tests/unit/`,
`tests/unit/db/` and `tests/unit/ui/*.test.tsx`. Of the four ratchets the ticket names, only
`cycles` has no headroom. Duplication and any-budget have plenty, and coverage was not measured.
The template PR actually tripped on i18n parity, the migration count, complexity, Stryker test
registration and one ESLint finding.

## 1. i18n: all 66 locales, really translated

**Is `en.json` plus runtime fallback accepted? No.** The runtime does fall back to English for an
absent key (`src/i18n/request.ts:28`, `:176` `deepMergeFallback`). CI does not accept that.
Job `i18n-ui-coverage` (`.github/workflows/ci.yml:594-658`) runs on any PR with code or catalog
changes. It feeds `Gate / CI` (`ci.yml:1663-1692`) and contains:

| Step (blocking unless noted)                             | What a new key must satisfy                                                                                                                                                                                                                                                                                                                                                                                                                            |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `check-key-completeness.mjs` (`ci.yml:652-653`)          | Absolute key-set parity: every locale carries every `en.json` key, none extra (script header `scripts/i18n/check-key-completeness.mjs:3-18`). At the base all 66 locales carry all 14,582 keys.                                                                                                                                                                                                                                                        |
| `check-new-key-coverage.mjs` (`ci.yml:641-644`)          | Diff-based. A key new in English must be a non-empty string in every locale that does not start with `__MISSING__:` (`scripts/i18n/check-new-key-coverage.mjs:89-99`). The marker has been rejected since 2026-09-17 (`:28-38`). A verbatim English copy **does** pass this gate.                                                                                                                                                                      |
| `check-translation-ratio.mjs` (`ci.yml:626-627`)         | Per-locale ratchet. A value identical to English, a marker or an absent key counts as untranslated (`scripts/i18n/check-translation-ratio.mjs:46-61`). The share may not exceed the baseline + 0.5 pp (`:63-70`, `config/quality/i18n-translation-baseline.json`). Measured headroom: `yo` fails at 9 verbatim leaves, `my` at 11, `ig` at 14, `si` at 15. So a 2–4-key PR with English copies would pass today, but only by eating the shared margin. |
| `check:ui-icu` (`ci.yml:619-620`)                        | Every catalog compiles as ICU (placeholders and tags intact).                                                                                                                                                                                                                                                                                                                                                                                          |
| `check-ui-value-drift.mjs` (`ci.yml:631-634`)            | Applies only if you **rewrite** an existing English value: you must then update or mark its translations.                                                                                                                                                                                                                                                                                                                                              |
| Glossary job `i18n-glossary-zhcn` (`ci.yml:665-683`)     | zh-CN/zh-TW must use the canonical terms. In #16961 the maintainer had to change "provider" to 提供者 (commit `7c92b155f7` message).                                                                                                                                                                                                                                                                                                                   |
| `i18n` job, `validate_translation.py` (`ci.yml:690-731`) | **Advisory** (`continue-on-error: true`, `ci.yml:698`).                                                                                                                                                                                                                                                                                                                                                                                                |

**Convention (measured):** recent upstream PRs translate every locale. For each commit, I counted
the new `en.json` keys and then the locale values identical to English. #14817 (`f5f698057f`) had
3 of 1,254 identical (19 keys × 66 locales). #16961, #14604, #13102, #16846, #17346, #14728 and
#17979 had 0 identical. The template #12590 copied English verbatim into the 40 locales of the
time (160 of 160). That predates the 2026-09-15 retranslation #13782 (`cde49c9372`), which
translated those leaves and made the ratio gate blocking. Do not copy the template's i18n.

**How:** add keys to `en.json`, then `npm run i18n:sync-ui` (adds markers). Then translate with
`node scripts/i18n/sync-ui-keys.mjs --locale=<codes> --translate-markers --batch-size=40` or
`bash scripts/i18n/translate-new-keys.sh`. Both need `OMNIROUTE_TRANSLATION_API_URL`/`_API_KEY`/
`_MODEL` (`scripts/i18n/translate-new-keys.sh:1-25,114`), and pointing them at our own OmniRoute
should work. Keys that must stay English (product or engine names) go in
`scripts/i18n/untranslatable-keys.json`, not behind a marker (`check-new-key-coverage.mjs:39-41`).
The golden path asks for surgical i18n patches and no external translation calls "unless the task
explicitly requires generated translations" (`docs/ops/CONTRIBUTION_GOLDEN_PATH.md:131`). The
new-key gate makes them required here, so review the generated diff.

## 2. Docs, OpenAPI and API reference

**Nothing checks the new field.** The OpenAPI entries for `/api/provider-nodes` and `/{id}` have no
request-body schema (`docs/openapi.yaml:2632-2671`). The OpenAPI gates check only paths and
security tiers: `check:openapi-coverage`, `check:api-docs-refs` (`scripts/check/check-api-docs-refs.mjs:1-30`),
`check:openapi-security-tiers` (advisory) and `check:openapi-breaking`. `docs-sync` checks only
`info.version` (`scripts/check/check-docs-sync.mjs:213-219`). `docs/reference/API_REFERENCE.md:1285`
has one row for `/api/provider-nodes*` and lists no fields. No doc in `docs/` (excluding
`docs/i18n/`) describes `iconUrl`, `customHeaders`, `chatPath`, `providerIdSuffix` or the
daily-reset fields.

**Optional convention:** #16961 documented its new create field in the operation `description`
and in a `400` response (`7c92b155f7` diff of `docs/openapi.yaml`). #12590 changed no docs at all.
If you edit `docs/openapi.yaml`, regenerate `skills/omni-providers/references/endpoints.md` with
`node scripts/skills/generate-agent-skills.mjs --apply`. That file is generated from the spec
(`src/lib/agentSkills/generator.ts:20,174,194`). `check:agent-skills-sync` fails a PR on drift
(quality.yml `merge-integrity`, `.github/workflows/quality.yml:487-512`).

**What a node-field PR does trip is the migration count.** `check:docs-all` (`package.json:174`)
blocks in `docs-gates` (`quality.yml:83-103`) and `docs-sync-strict` (`ci.yml:494-513`):

- `check-docs-counts-sync.mjs` checks the SQL file count strictly against `README.md`, `AGENTS.md`
  and `llm.txt` (`scripts/check/check-docs-counts-sync.mjs:132-137,777-787`). It also checks the
  database row of every `docs/i18n/<locale>/README.md`
  (`scripts/check/lib/readmeMigrationCount.mjs:14-53`).
- `check-docs-sync.mjs` requires each `docs/i18n/<locale>/llm.txt` to be byte-identical to the
  root `llm.txt` (`:247`, mirror check `:81-119`). One count bump therefore edits about 135 files:
  root README, AGENTS, llm.txt, 66 README rows and 66 llm.txt mirrors. #12590 needed a follow-up
  commit just to resync the llm.txt mirrors.
- `AGENTS.md`, `llm.txt` (and mirrors) and `skills/**/SKILL.md` are agent-instruction surfaces.
  No merge without explicit operator approval (`AGENTS.md:250-255`). Maintainers record these as
  "owner-approved count-only edits" (#16087 `08432419aa`, #16310 `852f18502d`, #13580 `238cb1b076`).
- Both styles have been merged. The #16087 author left the bump out "so it doesn't touch ~70
  translated files", and the maintainer added it at merge (upstream #16087 comment, 2026-10-12).
  The maintainer added it to #12590 too (upstream #12590 comment, 2026-09-03).
- **Base state:** at `139070680f` the README and its 66 mirrors say 219 (correct), but `AGENTS.md`
  and `llm.txt` say 213. That is inherited drift; a red base is tracked upstream in #17219.

Other doc gates apply only if you add matching content. `check:fabricated-docs` requires that
`/api/...` paths, env vars and file paths in `docs/` or `AGENTS.md` exist
(`scripts/check/check-fabricated-docs.mjs:1-24`). `check:env-doc-sync` applies only to new env vars.
English doc edits no longer need their 66 translated mirrors in the same PR: i18n docs drift is
advisory on PRs since 2026-10-28 (`ci.yml:527-558`). The README count row is still strict,
as above.

## 3. Process: changelog, migrations, tests

### CHANGELOG

- Never edit `CHANGELOG.md`. Add one fragment
  `changelog.d/{features|fixes|maintenance}/<PR-number>-<slug>.md` that starts with `- ` and uses
  the credit format `(#PR — thanks @user)` (`changelog.d/README.md:3-24`; PR checklist
  `CONTRIBUTING.md:410`). The release captain aggregates fragments.
- The gate (`check:changelog-integrity`, quality.yml `merge-integrity`) checks only well-formedness:
  section directory, leading bullet, no conflict markers
  (`scripts/check/check-changelog-integrity.mjs:242-276`). It does not check presence or filename.
  A fragment is still the norm: 77 of the last 80 `feat`/`fix` merges that touched `src/` or
  `open-sse/` since 2026-10-15 had one. #12590 had none.

### Migration numbering and renumbering

- Name files `NNN_description.sql`, zero-padded (`scripts/check/check-migration-numbering.mjs:25`).
  The gate fails on a duplicate version or any new gap; known gaps are frozen as 026/055/121
  (`:1-14,53`). Take the next free number on the tip. That was **223** at `139070680f`
  (highest is `222_entra_sso_identities.sql`).
- A migration is plain SQL: nullable `ALTER TABLE provider_nodes ADD COLUMN …`, as in the template
  `172_provider_node_daily_quota_reset.sql`.
- **Renumbering is routine and done at merge** when the tip took the number first: #16087 206→212,
  #16350 207→209, #13580 177→179, #14223 182→183/184, #16310 208→207 (commit messages of
  `08432419aa`, `b362310e41`, `238cb1b076`, `66688599b2`, `852f18502d`). Each renumber also moves:
  - the version-keyed `case "NNN":` in `isSchemaAlreadyApplied` (`src/lib/db/migrationRunner.ts:575-637`).
    A stale number "would answer for another migration's schema and skip it" (#14223; comment at
    `:596-606`).
  - test file names that encode the number (`tests/unit/migration-209-*`).
  - hard-coded migration totals in tests (#16350 switched these to counting the fixture).
  - the docs count from section 2.
- `RENAMED_MIGRATION_COMPATIBILITY` (`src/lib/db/migrationRunner/constants.ts:11`) maps old→new
  numbers for renumbers that may already be recorded in user databases. A not-yet-merged migration
  needs no entry. #16087 used an `isSchemaAlreadyApplied` guard for DBs that ran the old 206 file
  instead. _(Inference from the code and the commit messages; not stated as a rule anywhere.)_
- The template added a retroactive guard `case "172"` that checks both columns
  (`831ea040c3`, `migrationRunner.ts` +5). This is a convention, not a gate.

### Expected test layout

- Production changes in `src/`, `open-sse/`, `electron/` or `bin/` need test changes in the same
  PR. The `pr-test-policy` job (`ci.yml:732-764`) checks this with
  `scripts/check/check-pr-test-policy.mjs:5-21`. The same job also runs `check:test-masking` and
  `check:pr-evidence`. `src/i18n/messages/` is exempt (`:15-16`).
- Runner globs (enforced by `check:test-discovery`, `scripts/check/check-test-discovery.mjs:1-24`):
  - node:test collects `tests/unit/*.test.ts` and the listed subdirectories, including `db/`,
    `ui/`, `usage/` and `api/` (`package.json:142`).
  - Vitest UI (`vitest.config.ts:17-30`, `npm run test:vitest:ui`) collects `tests/unit/**/*.test.tsx`.
- Template #12590 had a schema test (`tests/unit/provider-node-daily-reset-schema.test.ts`) plus
  service tests. Later PRs on the same fields added:
  - a null-clearing regression test (`tests/unit/provider-node-null-quota-reset-13066.test.ts`, #13083)
  - an export/import case in `tests/unit/provider-transfer.test.ts` (#16877, which added the fields
    to `PORTABLE_NODE_FIELDS`, `src/lib/transfer/providerTransfer.ts:90-102`)
- #16961 (the closest recent node-field PR) had a route+schema+DB test
  `tests/unit/provider-nodes-custom-suffix.test.ts` and a modal test
  `tests/unit/ui/add-compatible-provider-id-suffix.test.tsx`. The Edit-modal precedent is
  `tests/unit/ui/edit-compatible-node-icon-url.test.tsx`: `// @vitest-environment jsdom`, mocked
  `next-intl`.
- Migration tests: `tests/unit/db/migration-<number>.test.ts`, covering **a fresh DB and an
  upgrade** from the prior schema (`docs/ops/CONTRIBUTION_GOLDEN_PATH.md:157-178`). #16087 used
  `tests/unit/db/usage-cost-migrations.test.ts`. DB tests must call `resetDbInstance()` and close
  handles (`AGENTS.md`, "Database Handles in Tests").
- A new unit test that imports a Stryker-mutated module must be listed in `stryker.conf.json`
  `tap.testFiles`. `check:mutation-test-coverage --strict` runs in Fast Quality Gates
  (`scripts/check/check-mutation-test-coverage.mjs:15-20`). Mutated modules near this feature:
  `open-sse/handlers/chatCore/{responseHeaders,semanticCache,nonStreamingSse}.ts`,
  `open-sse/services/accountFallback.ts` and `src/sse/services/auth.ts` (`stryker.conf.json`
  `mutate`). The template tripped this and added a test file to `stryker.conf.json`.

### PR body and timing

- Use `.github/PULL_REQUEST_TEMPLATE/feature.md`. Its Design section asks: is it opt-in? Is
  data-mutating behaviour off by default?
- `check:pr-evidence` fails a body that claims "tests pass" without command output
  (`scripts/check/check-pr-evidence.mjs:1-28`).
- `check:ai-attribution` rejects AI `Co-Authored-By` trailers **and** "Generated with Claude Code"
  footers in commits and in the PR title or body (`scripts/check/check-ai-attribution.mjs:1-17`).
  Strip any harness-added footer.
- If the base is still red, add the line `⚠️ base-red inherited: #<issue>` (`AGENTS.md`, Base-green
  check). #17219 was open for `release/v3.8.53` on 2026-11-01.
- From 3.8.55, feature PRs are held with the `v4-feature` label for the v4 channel, not closed
  (`.github/PULL_REQUEST_TEMPLATE/feature.md:16-17`).

## 4. Pitfalls: ratchets

Measured on the base unless marked. The velocity phase loosened every numeric baseline by 20%
until 4.0, and `--require-tighten` is off (`config/quality/quality-baseline.json` `_policy`).

| Ratchet asked about | Headroom at `139070680f`                                                                                                                                                                                                                                                                                                                         | Risk for this PR                                                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| Coverage            | **Not measured** (needs the 8-shard CI run). Gate: c8 floor 60/60/60/60 (`ci.yml:1263-1264`). Ratchet: 67.33 / 67.33 / 72.02 / 65.08 statements/lines/functions/branches, plus per-module lines for chatCore 60.38, accountFallback 80.65 and auth 77.13, each with eps 1.5 (`quality-baseline.json` `metrics`).                                 | Low with focused tests. Untested lines added to `chatCore.ts` would hit the per-module ratchet. Not seen tripping in the sampled PRs. |
| any-budget          | PASS. Zero explicit `any` in the listed files, which include `open-sse/handlers/chatCore.ts`, `open-sse/services/usage.ts`, `open-sse/utils/usageTracking.ts` and `src/lib/usage/usageHistory.ts` (`scripts/check/check-t11-any-budget.mjs:12-118`). ESLint `no-explicit-any` is an error in `open-sse/` and `tests/` (`AGENTS.md`, Code Style). | Low. Just do not use `any`.                                                                                                           |
| Duplication         | 3.02% vs baseline 6.86% (`config/quality/duplication-baseline.json`)                                                                                                                                                                                                                                                                             | None                                                                                                                                  |
| Cycles              | **14 of ceiling 14, zero headroom** (`npm run check:cycles:ratchet`)                                                                                                                                                                                                                                                                             | Any new import cycle fails. Watch new imports between `src/lib/db/providers/nodes.ts`, schemas and usage code.                        |

Ratchets the ticket did not name, but which trip PRs of this shape:

- **open-sse → src boundary:** 774 of 774 edges, zero headroom. It is base-relative on PRs: only
  edges the PR adds block (`quality.yml:280-290`). A conversion point in `open-sse/` that reads the
  node from `src/lib/db/` would add an edge. #16961's merge note says "no new open-sse -> src
  import".
- **Complexity / cognitive / max-lines (new-code mode):** blocks new violations in touched files.
  The limits are complexity 15, max-lines-per-function 80 and cognitive 15
  (`eslint.complexity-ratchets.config.mjs:57-70`, `scripts/check/newCodeMode.mjs:1-13`).
  #12590 had to extract helpers in `nodes.ts` and `moonshotQuotaFetcher.ts` (maintainer comment on
  #12590). Dead-code (knip) is also new-code mode: an unused new export in a touched file blocks.
- **Mutation-test-coverage** (Stryker `tap.testFiles`): see section 3. It tripped #12590.
- **ESLint:** `lint-guard` runs `--max-warnings 0` (`quality.yml:449-451`). #12590 got a red for a
  locally redefined `toNumber` helper and had to import the shared one. _(The exact rule was not
  identified.)_
- **File size:** cap 1440 for new or unfrozen files (`config/quality/file-size-baseline.json:285-286`).
  Frozen files may not grow. It is base-relative on PRs. #12590 needed a rebaseline for growth in
  frozen chokepoints. Today the likely touch points are far below the cap: `nodes.ts` 234 lines,
  the two `provider-nodes` routes 255/188, `EditCompatibleNodeModal.tsx` 445, `schemas/provider.ts`
  815, `providerTransfer.ts` 716. `chatCore.ts` is at 3450 of its frozen 3945.
- **Validation of a blank field:** the template shipped a Zod schema that rejected `null` for one
  of its two fields, though the modal sends `null` when cleared. #13083 had to fix it. Accept
  `null` (clear) on update and test it.

## 5. Checklist for the spec's "upstream PR contents" section

1. Migration `NNN_provider_node_<name>.sql` with the next free number. Keep it nullable, with no
   default behaviour change. Optionally add an `isSchemaAlreadyApplied` case keyed to the final number.
2. Tests:
   - schema test (including `null` clears)
   - route create/update test
   - DB/migration test (fresh + upgrade) under `tests/unit/db/`
   - transfer test (`PORTABLE_NODE_FIELDS`)
   - modal test under `tests/unit/ui/*.test.tsx`
   - Stryker `tap.testFiles` entries if a test imports a mutated module
3. New `en.json` keys translated into all 66 locales. ICU-valid, zh glossary respected.
4. Migration-count bump in README + 66 README rows + `AGENTS.md` + `llm.txt` + 66 llm.txt mirrors,
   flagged in the PR body for owner approval. Or say explicitly that it is left to the maintainer,
   as #16087 did.
5. `changelog.d/features/<PR>-<slug>.md`.
6. Optional: an OpenAPI operation-description line, plus the regenerated
   `skills/omni-providers/references/endpoints.md`.
7. PR body:
   - feature template with command-output evidence
   - no AI footer or trailer
   - base-red line if the base is still red
   - a note that the migration number may need a merge-time renumber

## Not verified

- Coverage headroom per metric (not run locally; needs the sharded CI run).
- Which ESLint rule flagged #12590's duplicate `toNumber`.
- Whether maintainers prefer the count bump in the contributor PR or at merge. Both are on record;
  neither is written policy.
- That a not-yet-merged renumber never needs a `RENAMED_MIGRATION_COMPATIBILITY` entry. This is
  inferred from code and commit messages.
- Headroom figures (i18n ratio margins, cycles, boundary) are point-in-time. They move with every
  merge to the release branch.
