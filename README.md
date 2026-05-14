# pkg-quarantine

> **\[ pkg \] —— wait 4 days ——> \[ install \]**  ← supply-chain blocker

[![npm version](https://img.shields.io/npm/v/@happyberg/pkg-quarantine)](https://www.npmjs.com/package/@happyberg/pkg-quarantine)
[![license](https://img.shields.io/npm/l/@happyberg/pkg-quarantine)](https://github.com/happyberg/pkg-quarantine/blob/master/LICENSE)
[![node](https://img.shields.io/node/v/@happyberg/pkg-quarantine)](https://github.com/happyberg/pkg-quarantine/blob/master/package.json)

**Block freshly-published packages before they reach your machine.**

One command configures a release-age cooldown across every supported package manager on your machine. Malicious versions of hijacked packages are typically [detected and pulled within hours to a few days](https://socket.dev/blog/npm-introduces-minimumreleaseage-and-bulk-oidc-configuration), so a 4-day hold sits comfortably outside that window.

```bash
npm install -g @happyberg/pkg-quarantine
quarantine init
quarantine --version   # 0.2.4
```

That's it. For managers with native release-age support (npm, pnpm, bun, uv, yarn, deno), every install now silently rejects anything published in the last 4 days. For managers without it (pip, gem, composer, cargo, hex), `quarantine update` enforces the same policy at update time.

> **Deep dive**: [TanStack and the day provenance attestation stopped being a defense](https://happyberg.com/blog/tanstack-mini-shai-hulud/). Full breakdown of the May 11, 2026 attack chain, the defenses that failed, and the cooldown that held.

---

## Why this exists

Supply-chain attacks are arriving in waves, the defenses we built for them are being defeated one by one, and a release-age cooldown is the layer that still holds.

### The May 2026 wave

On May 11, 2026 between 19:20 and 19:26 UTC, [84 malicious npm artifacts across 42 `@tanstack` packages](https://www.aikido.dev/blog/mini-shai-hulud-is-back-tanstack-compromised) were published. Within 48 hours the same campaign covered 172 packages across npm and PyPI ([Wiz](https://www.wiz.io/blog/mini-shai-hulud-strikes-again-tanstack-more-npm-packages-compromised), [Snyk](https://snyk.io/blog/tanstack-npm-packages-compromised/), [Socket](https://socket.dev/blog/tanstack-npm-packages-compromised-mini-shai-hulud-supply-chain-attack), [Endor Labs](https://www.endorlabs.com/learn/shai-hulud-compromises-the-tanstack-ecosystem-80-packages-compromised)). `@tanstack/react-router` alone has ~12M weekly downloads.

The detail that matters: the malicious versions were signed. Attackers chained a `pull_request_target` Pwn Request, GitHub Actions cache poisoning, and OIDC token extraction from runner process memory to publish through TanStack's legitimate trusted-publishing pipeline. Sigstore verified the artifacts. Provenance attestation showed them as authentic. They were indistinguishable from legitimate by every signature check the ecosystem built for this case.

This is wave 4 of the Shai-Hulud campaign:

- **Sept 15, 2025**: [Shai-Hulud (original)](https://unit42.paloaltonetworks.com/npm-supply-chain-attack/), self-replicating worm, hundreds of packages.
- **Nov 2025**: [Shai-Hulud 2.0](https://www.microsoft.com/en-us/security/blog/2025/12/09/shai-hulud-2-0-guidance-for-detecting-investigating-and-defending-against-the-supply-chain-attack/), 25,000+ malicious GitHub repos, Zapier / PostHog / Postman hit.
- **March 2026**: Trivy scanner npm packages, attributed to TeamPCP.
- **March 31, 2026**: [axios](https://www.elastic.co/security-labs/axios-one-rat-to-rule-them-all) (attributed to UNC1069), `plain-crypto-js` trojan, ~100M weekly downloads.
- **April 1-8, 2026**: Asurion impersonation campaign (`sbxapps`, `asurion-hub-web`, `soluto-home-web`, `asurion-core`), multi-stage credential harvester.
- **April 21, 2026**: [pgserve worm](https://www.theregister.com/2026/04/22/another_npm_supply_chain_attack/), cross-registry npm + PyPI, self-propagating.
- **April 22, 2026**: [`@bitwarden/cli`](https://thehackernews.com/2026/04/bitwarden-cli-compromised-in-ongoing.html) malicious for 93 minutes, attributed to TeamPCP.
- **May 11-13, 2026**: TanStack / Mini Shai-Hulud wave 4, 172 packages, **signed by the legitimate pipeline**.

### What still works

Detection time. Socket flagged the TanStack artifacts within 6 minutes of publication. By the time anyone ran a fresh install long enough for the malicious versions to enter a dependency tree the second time, the security community had already pulled them.

A 4-day release-age cooldown closes the rest of the gap. Fresh installs of a brand-new version are held back until detection has caught up. It is one defense the May 2026 wave did not subvert, because it does not depend on trusting any party, signature, or attestation. It just waits.

The native settings exist:

| Manager | Setting |
|---------|---------|
| npm 11.10+ | `min-release-age` |
| pnpm | `minimum-release-age` |
| bun | `install.minimumReleaseAge` |
| uv | `exclude-newer` |
| yarn (per project) | `npmMinimalAgeGate` |
| deno (per project) | `minimumDependencyAge` |

They live in six config files with six different keys and six different units. `pip`, `gem`, `composer`, `cargo`, and `hex` still have no native install-time gate. And `npm < 11.10` silently accepts `min-release-age` while ignoring it. Green config, zero protection.

`pkg-quarantine` configures all of them at once and verifies they are actually in effect.

---

## Commands

```bash
quarantine init [managers...]        # Write/merge quarantine configs
quarantine audit [managers...]       # Traffic-light config report
quarantine status                    # Quick policy summary
quarantine update [managers...]      # Quarantine-aware global updater
```

### `quarantine init`

Writes quarantine settings to each manager's global config file. Existing settings and auth tokens are preserved; it merges, never clobbers.

```bash
quarantine init                  # All detected managers
quarantine init npm pnpm uv      # Specific managers
quarantine init --dry-run        # Preview without writing
quarantine init --days 7         # Custom quarantine period
```

### `quarantine audit`

Checks current config against desired quarantine state. Traffic-light output:

```
  npm     ✓ configured    min-release-age=4 days
  pnpm    ✓ configured    minimum-release-age=5760 minutes
  uv      ✗ missing       add exclude-newer to ~/.config/uv/uv.toml
  pip     ~ wrong value   only-binary not set
```

### `quarantine status`

One-line-per-manager summary of quarantine posture.

### `quarantine update`

Quarantine-aware global package updater. Checks each outdated package's publish date against the registry API before upgrading. Refuses to install anything that's too fresh unless you pass `--force`.

```bash
quarantine update                    # All managers
quarantine update npm                # Just npm
quarantine update --dry-run          # Preview without installing
quarantine update --force            # Bypass quarantine (prints a warning)
```

---

## For AI agents

**If you use Claude Code, Codex, Cursor, or any AI coding assistant that can install packages, this is especially for you.**

AI agents install dependencies automatically, often without a human reviewing the exact version or publish date. That's fine for productivity. It's a supply-chain risk if the agent happens to install a freshly-hijacked package.

`quarantine init` enforces the policy at the package manager level, so it applies to *every* install, whether a human typed it or an agent did.

### Setting it up once

```bash
# Install and configure everything:
npm install -g @happyberg/pkg-quarantine
quarantine init
```

After that, `npm install`, `pip install`, `uv add`, etc. will all silently enforce the quarantine with no further action needed.

### Instructing your agent

Add this to your `CLAUDE.md` (or equivalent agent instructions file):

```markdown
## Package Security

A 4-day quarantine policy is enforced via pkg-quarantine.
Versions published less than 4 days ago will not install.

Rules:
- Never run bare `npm install -g <pkg>@latest`.
  Use `quarantine update` instead.
- Before installing an unfamiliar package, check its
  publish date.
- If an install fails with a quarantine error, report
  it rather than bypassing it.
- The quarantine is a safety net, not an obstacle.
```

### Verifying your agent respects it

```bash
quarantine audit        # Check all manager configs are set
quarantine status       # One-line status summary
```

If an agent tries to bypass quarantine with `--ignore-scripts=false` or `--force`, treat that as a signal to review what it's installing.

### For CI pipelines

Add quarantine verification to your CI setup step:

```yaml
- name: Verify quarantine policy
  run: |
    npm install -g @happyberg/pkg-quarantine
    quarantine audit --exit-code   # exits 1 if any manager is misconfigured
```

---

## Supported managers

`pkg-quarantine` covers 13 package managers, but they fall into three honest tiers depending on what the underlying tool supports.

### Tier 1: Native install-time quarantine

These managers ship a built-in release-age gate. `init` writes the setting and *every* install (manual or via an AI agent) is automatically protected.

| Manager | Mechanism | Notes |
|---------|-----------|-------|
| **npm** | `min-release-age` in `~/.npmrc` | **Requires npm ≥ 11.10.0** (Feb 2026). Earlier versions silently ignore the setting; `quarantine audit` warns. |
| **pnpm** | `minimum-release-age` (minutes) in pnpm rc | Config lives at `~/Library/Preferences/pnpm/rc` on macOS, `~/.config/pnpm/rc` on Linux (or `$XDG_CONFIG_HOME/pnpm/rc`). |
| **bun** | `install.minimumReleaseAge` (seconds) in `bunfig.toml` | — |
| **uv** | `exclude-newer = "N days"` in `uv.toml` | — |
| **yarn** | `npmMinimalAgeGate` in `.yarnrc.yml` | Per-project only. `init` prints the snippet to add. |
| **deno** | `minimumDependencyAge` in `deno.json` | Per-project only. `init` prints the snippet to add. |

### Tier 2: Update-time quarantine via `quarantine update`

These managers have no native release-age config, so install-time enforcement isn't possible. `quarantine update` checks the registry API before upgrading and refuses fresh versions. Bare `pip install foo` / `gem install foo` / etc. are *not* gated; you must use `quarantine update`.

| Manager | Registry checked | Hardening `init` configures |
|---------|------------------|------------------------------|
| **pip** | PyPI JSON API | `only-binary = :all:` (blocks source-build attacks; not a quarantine itself) |
| **gem** | rubygems.org API | `BUNDLE_TRUST___POLICY: MediumSecurity` |
| **composer** | packagist API | `no-scripts`, `allow-plugins={}`, `secure-http` |
| **cargo** | crates.io API | Recommends `cargo-audit` |
| **hex** | hex.pm API | Recommends `mix_audit` |

> Composer 2.9+ already enables `audit.block-insecure` by default, so `init` no longer writes that setting.

### Tier 3: Audit and recommendation only

These managers don't have a release-age model at all. `init` prints best-practice recommendations and `audit` reports posture; there is no enforcement.

| Manager | What `init` does |
|---------|------------------|
| **go** | Verifies sumdb is active, recommends `govulncheck` |
| **brew** | Lists third-party taps as a posture warning |

---

## Configuration

Global config at `~/.config/quarantine/config.toml`:

```toml
quarantine_days = 4
managers = ["npm", "pnpm", "bun", "uv", "pip", "gem", "composer", "go", "brew", "cargo", "hex"]
```

Defaults apply if the file doesn't exist: 4-day hold, all managers.

The default `managers` array lists 11 entries because `yarn` and `deno` are gated per-project (their release-age settings live in `.yarnrc.yml` and `deno.json`, not a global rc). `pkg-quarantine` still counts them in its "13 supported managers" headline, but `quarantine init` for those prints the snippet you add to the project, instead of writing global config.

---

## Security and contributing

- Security issues: please follow the disclosure process in [SECURITY.md](SECURITY.md).
- Contributions welcome: see [CONTRIBUTING.md](CONTRIBUTING.md) and the [code of conduct](CODE_OF_CONDUCT.md).

---

## Design

- **2 runtime dependencies**: `commander` + `smol-toml`. Zero transitive deps.
- **Dependency injection**: All commands receive `FileSystem` and `Shell` interfaces. Tests use in-memory mocks; no disk or network in tests.
- **Auth-token safe**: The custom `.npmrc` parser treats `//` lines as scoped registry entries (not comments), preserving all auth tokens intact.
- **Native `fetch()`**: Registry API calls use Node's built-in fetch. No HTTP library dependency.
- **Merge, never clobber**: `init` reads existing config before writing, preserving all unrelated settings.

---

## Development

```bash
npm test              # Run tests
npm run build         # Build ESM + CJS
npm run lint          # Type-check
npm run test:watch    # Watch mode
```

---

## License

[MIT](LICENSE).
