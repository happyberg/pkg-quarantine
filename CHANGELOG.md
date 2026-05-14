# Changelog

All notable changes to this project are documented in this file.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.2.4] — 2026-05-14

### Changed

- README: removed em-dashes from prose, moved the deep-dive blog
  pointer right under the install snippet so it lands above the fold
  on npmjs.com, added `quarantine --version` to the install example,
  linked the LICENSE file, and added pointers to SECURITY.md and
  CONTRIBUTING.md. Dropped the "only defense" framing in favor of
  "one defense that did not get subverted" — accurate without
  overstating.
- README "Configuration" section: explicit note that `yarn` and `deno`
  ship per-project so the default `managers` array lists 11; the
  header still claims 13 supported because per-project enforcement is
  real, just gated differently.
- CHANGELOG: softened a few past entries (the `smol-toml` swap,
  the `--version` fix, the May-2026-wave framing) to remove
  marketing-flavored claims I could not verify byte-for-byte. The
  reality stands without them.

No code changes.

## [0.2.3] — 2026-05-14

### Changed

- **README**: Design section now mentions `smol-toml` instead of the
  stale `@iarna/toml` reference (the swap happened in 0.2.2; the
  README missed the rewrite). Also added a deep-dive pointer to the
  https://happyberg.com/blog/tanstack-mini-shai-hulud/ writeup so
  readers can land on the full attack chain analysis from npmjs.com.

No code changes.

## [0.2.2] — 2026-05-14

### Changed

- Replaced `@iarna/toml@2.2.5` (last published July 2023) with
  `smol-toml@1.6.1` (last published March 2026). `smol-toml`
  dual-publishes ESM + CJS, has a more recent maintenance cadence,
  and parses + stringifies the configs we touch (`global-config.ts`,
  `uv.ts`, `bun.ts`) without behavior changes in our test suite.
  Same runtime dep count (`commander` + `smol-toml`), zero transitive
  runtime deps. 154/154 tests pass after the swap.

## [0.2.1] — 2026-05-14

### Fixed

- `quarantine --version` now reports the actual release version. The
  string was hardcoded as `'0.1.3'` in `src/index.ts` and was not
  bumped through the 0.1.4 / 0.1.5 / 0.1.6 / 0.2.0 cycle. It is now
  injected at build time by `tsup` from `package.json` via a
  `__PKG_VERSION__` define, so future version bumps flow through to
  the CLI automatically.

## [0.2.0] — 2026-05-14

### Changed

- **Package moved to the `@happyberg` npm scope.** Now installed as
  `npm install -g @happyberg/pkg-quarantine` (was: `npm install -g
  pkg-quarantine`). The CLI binary name is unchanged: still `quarantine`
  on your PATH. Same code as 0.1.6, just under the org's namespace so
  the package appears under https://www.npmjs.com/~happyberg.
- The unscoped `pkg-quarantine` package on npm (versions 0.1.0 through
  0.1.6) is deprecated with a message pointing to the scoped name.

### Migration

If you have an existing install:

```bash
npm uninstall -g pkg-quarantine
npm install -g @happyberg/pkg-quarantine
```

The `quarantine` CLI remains identical. All `quarantine init`,
`quarantine audit`, `quarantine update`, and `quarantine status`
commands work the same way.

## [0.1.6] — 2026-05-14

### Changed

- **Package author rebranded to Happyberg Labs.** The repo moved from
  `dgilperez/pkg-quarantine` to `happyberg/pkg-quarantine` in 0.1.5,
  but the `author` field in `package.json` still read "David Gil",
  which is what npmjs.com displays on the package page. Now reads
  "Happyberg Labs" with `hola@happyberg.com` / https://happyberg.com.
  David Gil Pérez is listed in `contributors[]` as the original
  author and maintainer — the personal credit stays, the company
  branding lands on the npm page.

No code changes between 0.1.5 and 0.1.6. 0.1.5 was tagged and released
on GitHub but was not published to npm; npm jumps from 0.1.4 to 0.1.6.

## [0.1.5] — 2026-05-13

### Changed

- **README rewritten around the May 2026 supply-chain wave.** The previous
  README led with the March 31, 2026 axios compromise as the motivating
  incident. After the May 11 TanStack / Mini Shai-Hulud wave 4 attack
  (in which 172 malicious npm + PyPI artifacts were signed by TanStack's
  legitimate OIDC pipeline, Sigstore-verified and provenance-attested,
  indistinguishable from legitimate by every signature check), the
  framing now leads with the cadence of named incidents from Sept 2025
  through May 2026 and the observation that a release-age cooldown is
  one defense that the May wave did not subvert. axios stays in the
  cadence list. No behavior changes in this release; the tool's
  feature set is unchanged.

### Security (dev dependencies)

- `postcss` bumped to 8.5.14 (was 8.5.8) to clear
  [GHSA-qx2v-qp2m-jg93](https://github.com/advisories/GHSA-qx2v-qp2m-jg93)
  (moderate XSS in CSS stringify output). `postcss` is an indirect
  dev-only transitive; it is not shipped to users (`files: ["dist"]`).
  `npm audit` now reports 0 vulnerabilities.

## [0.1.4] — 2026-04-23

### Fixed

- **Critical: pnpm quarantine was silently ineffective on macOS.** Two
  independent bugs cancelled each other's symptoms so `quarantine audit`
  reported green while pnpm continued to install fresh packages.
  - **Wrong config path on macOS.** pnpm follows Apple's preferences
    convention — its global rc lives at `~/Library/Preferences/pnpm/rc`
    on darwin (see pnpm's own `getConfigDir`), *not* the XDG
    `~/.config/pnpm/rc` this tool was writing to. pnpm silently ignored
    the file. Linux users with XDG-style paths were unaffected.
  - **Wrong setting key.** Previous versions wrote `minimumReleaseAge`
    (camelCase). That alias is accepted by `pnpm config list` but is
    *not* applied during install resolution — only the kebab-case
    `minimum-release-age` actually gates installs. Reproduce on any
    platform by placing the camelCase form in the correct rc file and
    running `pnpm add <pkg>@<fresh-version>`: install proceeds.
- `paths.pnpmrc` now mirrors pnpm's own platform branches (XDG →
  macOS → POSIX → Windows) instead of assuming XDG everywhere.
- `quarantine audit pnpm` now flags legacy `minimumReleaseAge` entries
  as `missing` so anyone upgrading from 0.1.3 gets a warning at audit
  time instead of silent non-protection.

### Migration

If you ran `quarantine init` on macOS with 0.1.3 or earlier:

1. `quarantine init pnpm` to write the correct config (new path + key).
2. Delete the stale `~/.config/pnpm/rc` to avoid confusion — pnpm wasn't
   reading it anyway.
3. Verify: `pnpm config get minimum-release-age` should return `5760`.

## [0.1.3] — 2026-04-08

### Fixed

- README header now renders correctly on npmjs.com. The previous version
  used Unicode box-drawing characters to spell "QUARANTINE" — these
  rendered fine on GitHub but came out as scrambled glyphs on npm because
  npm's monospace font doesn't space the box-drawing chars correctly.
  Replaced with a clean H1 + tagline.
- The "Instructing your agent" section's nested markdown code block was
  being clipped on the right edge on npmjs.com (npm's code blocks don't
  word-wrap and the lines were too long). Lines have been hard-wrapped to
  fit a ~55-character column.
- Badge links now point to absolute GitHub URLs (`blob/master/LICENSE`,
  `blob/master/package.json`) instead of bare relative paths.

## [0.1.2] — 2026-04-08

### Fixed

- **Critical: npm version compatibility check now requires npm ≥ 11.10.0.**
  Previously the check used `MIN_SUPPORTED_MAJOR = 10`, which silently passed
  on every npm 10.x and 11.0–11.9 install — but `min-release-age` only shipped
  in npm 11.10.0 (Feb 2026). On older npm, the setting is silently ignored
  *and* npm warns it will hard-error in the next major. The audit was reporting
  green while providing zero protection. Compat check now does a proper semver
  comparison and the warning message tells the user exactly what to upgrade.
- README install snippet for `quarantine audit --exit-code` now actually exists
  (see Added below).
- README contradiction: `update --force` is now acknowledged in the description
  of `quarantine update`.
- README "supported managers" table reorganised into three honest tiers:
  native install-time quarantine, update-time quarantine, audit-only.
- Removed `composer.audit.block-insecure` from `init` — Composer 2.9+ enables
  it by default and the setting cannot be reliably applied via global
  `~/.config/composer/config.json` (see [composer/composer#12611](https://github.com/composer/composer/issues/12611)).

### Added

- `quarantine audit --exit-code` flag for CI integration. Exits 1 if any
  manager has warnings or missing settings. The README CI snippet now matches
  the implementation.
- `auditCommand()` returns `{ ok, warn, missing }` totals so callers (and
  tests) can introspect the result.

## [0.1.1] — 2026-04-07

### Fixed

- `NpmHandler.audit()` now emits an `npm-version-compat` warning when the
  installed npm is older than v10, explaining the `--before` / `min-release-age`
  interaction and providing an upgrade command ([#2]).
- Shared test `mockShell` helper now returns a sensible `10.2.3` npm version
  by default so audit tests aren't polluted by the version check.

## [0.1.0] — 2026-04-06

### Added

- `quarantine init` — write/merge quarantine config for all detected managers.
- `quarantine audit` — traffic-light report: green (ok), yellow (wrong value),
  red (missing).
- `quarantine status` — one-line-per-manager policy summary.
- `quarantine update` — quarantine-aware global package updater; checks publish
  dates via registry APIs before upgrading.
- Support for 13 package managers: npm, pnpm, bun, yarn, deno, uv, pip, gem,
  composer, go, brew, cargo, hex.
- Global config at `~/.config/quarantine/config.toml` with `quarantine_days`
  and `managers` fields.
- Auth-token-safe `.npmrc` parser (`//` lines preserved as scoped registry
  entries, not comments).
- Dependency-injection design: all commands receive `FileSystem` and `Shell`
  interfaces — zero disk or network I/O in tests.
- Native `fetch()` for registry API calls — no HTTP library dependency.
- 130+ tests across 27 test files.

[Unreleased]: https://github.com/happyberg/pkg-quarantine/compare/v0.2.4...HEAD
[0.2.4]: https://github.com/happyberg/pkg-quarantine/compare/v0.2.3...v0.2.4
[0.2.3]: https://github.com/happyberg/pkg-quarantine/compare/v0.2.2...v0.2.3
[0.2.2]: https://github.com/happyberg/pkg-quarantine/compare/v0.2.1...v0.2.2
[0.2.1]: https://github.com/happyberg/pkg-quarantine/compare/v0.2.0...v0.2.1
[0.2.0]: https://github.com/happyberg/pkg-quarantine/compare/v0.1.6...v0.2.0
[0.1.6]: https://github.com/happyberg/pkg-quarantine/compare/v0.1.5...v0.1.6
[0.1.5]: https://github.com/happyberg/pkg-quarantine/compare/v0.1.4...v0.1.5
[0.1.4]: https://github.com/happyberg/pkg-quarantine/compare/v0.1.3...v0.1.4
[0.1.3]: https://github.com/happyberg/pkg-quarantine/compare/v0.1.2...v0.1.3
[0.1.2]: https://github.com/happyberg/pkg-quarantine/compare/v0.1.1...v0.1.2
[0.1.1]: https://github.com/happyberg/pkg-quarantine/compare/v0.1.0...v0.1.1
[0.1.0]: https://github.com/happyberg/pkg-quarantine/releases/tag/v0.1.0
