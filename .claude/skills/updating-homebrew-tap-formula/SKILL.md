---
name: updating-homebrew-tap-formula
description: Use when updating a formula in a custom Homebrew tap to match a newer version from homebrew-core, while preserving local customizations like build flags or dependency overrides
---

# Updating a Homebrew Tap Formula from Upstream

## Overview

Sync a custom Homebrew tap formula with its upstream homebrew-core counterpart while preserving intentional local differences (e.g., static vs shared builds, patched dependencies).

## When to Use

- Upstream homebrew-core has a newer version of a formula you maintain in a tap
- You need to update version, hash, bottles, and upstream structural changes
- You must NOT lose local build customizations

## Workflow

1. **Read the local formula** to understand current version and local customizations
2. **Archive the current version as a versioned formula** before touching the main file. This preserves the old version so users can still pin it via `brew install <tap>/<formula>@<old-version>`. See [Archiving the Old Version](#archiving-the-old-version) below.
3. **Fetch the upstream formula** from homebrew-core raw GitHub content:
   `https://raw.githubusercontent.com/Homebrew/homebrew-core/master/Formula/{first-letter}/{formula-name}.rb`
4. **Diff the two** and classify each change:

   | Change Type | Action |
   |------------|--------|
   | Version URL + sha256 | Always update |
   | New Homebrew DSL fields (e.g., `compatibility_version`) | Apply — these are framework changes |
   | Dependency changes (e.g., `uses_from_macos` to `on_linux`) | Apply — upstream dependency decisions |
   | Build flags you intentionally differ on (e.g., `BUILD_SHARED_LIBS`) | **Preserve local value** |
   | Bottle hashes | Update if your tap uses upstream bottles; remove block if you build your own |
   | Test block changes | Apply unless you have custom tests |

5. **Apply changes** selectively per the table above
6. **Verify** the final formula reads correctly

## Archiving the Old Version

Before updating `Formula/<name>.rb` to the new version, copy its current contents into a versioned formula file. Steps:

1. Copy the current `Formula/<name>.rb` to `Formula/<name>@<current-version>.rb` verbatim.
2. **Rename the class** to match the versioned filename. Homebrew's mapping converts `lld@22.1.4` to `LldAT2214` (CamelCase name, `AT`, version digits with dots stripped). To confirm the exact class name, run:

   ```bash
   brew ruby -e 'require "formulary"; puts Formulary.class_s("<name>@<version>")'
   ```

3. **Remove the inherited `bottle do` block.** Its `sha256`s belong to the unversioned name and won't resolve under the versioned filename. The archived version builds from source until the bottle workflow rebuilds a correct `<name>@<version>` bottle for it (see [Regenerating Bottles](#regenerating-bottles-self-built-tap)).
4. **Add `keg_only :versioned_formula`** in place of the removed bottle block. This prevents the versioned install from conflicting with the main formula's symlinks.
5. **Leave everything else (including `head`, `livecheck`, `link_overwrite`, dependency stanzas) byte-identical to the historical content.** The point of the archive is faithful historical preservation, not redesign.
6. Verify with `ruby -c Formula/<name>@<current-version>.rb`.

After archiving, proceed to update `Formula/<name>.rb` to the new upstream version per the main workflow.

## Regenerating Bottles (self-built tap)

If the tap builds and serves its own bottles instead of pouring upstream
homebrew-core ones, the `bottle do` block must be rebuilt on every version
bump — the old block's version and `sha256`s no longer match the new source.

**Version-matching is the hard part.** When the formula links another
Homebrew formula (here `lld` links `llvm`), a bottle is only ABI-valid
against the *exact* version of that dependency it was built with. And
GitHub's macOS runner images drift: at the same moment, `macos-15-intel`
may ship Homebrew `llvm` 22.1.7 while `macos-14` ships 22.1.8. So no single
formula version satisfies every runner — you need a bottle of `lld@22.1.7`
*and* of `lld` (22.1.8) live at once, each built against its matching llvm.

This repo automates it via `.github/workflows/build-bottles.yml`
(**manual `workflow_dispatch` only** — a push trigger would re-fire on the
bottle-block merge and rebuild forever):

1. The **build** job, on each macOS runner, reads that runner's Homebrew
   `llvm` version `V`, then bottles the formula whose version equals `V` —
   the main `lld` if it matches, otherwise the archived `lld@V`. This is the
   same selection WasmEdge's `Resolve matching lld formula` step makes, so
   every runner pours a bottle built against the llvm it actually has. It is
   **idempotent per (formula, tag)**: if the chosen formula already declares a
   bottle for this runner's tag it skips entirely, so a re-run only fills
   genuine gaps — it never rebuilds a working bottle (which would churn the
   rebuild number or drop another runner's tag). Otherwise it runs
   `brew install --build-bottle` + `brew bottle`.
2. The **publish** job groups the uploaded JSONs by formula (the name before
   `--`), runs `brew bottle --merge --write` per formula to write each block
   (correct `root_url` + per-OS `sha256`), uploads the tarballs to the
   release, and opens one PR with all changed `Formula/*.rb`.
3. **Merge that PR.** Afterwards `brew install <tap>/<name>` (or
   `<name>@<ver>`) pours instead of compiling.

**Run it** after every version bump, and again whenever a runner image's
llvm drifts and the consumer CI starts building lld from source (the symptom:
a macOS job back at its from-source wall-clock, log shows
`Installing .../lld@<ver> dependency: cmake`).

Notes:
- The bottle **matrix mirrors the consuming CI's runners 1:1** — only build
  bottles for OS/arch combos something actually pours. A platform without a
  bottle just builds from source via a
  `brew install <f> || brew install --build-from-source <f>` fallback.
- **Archived versioned formulae get their own bottles here** — that is why
  the archive step removes the *inherited* `bottle do` block (its `sha256`s
  belong to the unversioned name) but does **not** keg the version away
  forever: the workflow rebuilds a correct `lld@<ver>` bottle when a runner
  still needs it. A `keg_only` formula bottles fine.
- `brew bottle` writes a **double-dash** local name
  (`<name>--<version>.<tag>.bottle[.<rebuild>].tar.gz`), but Homebrew fetches
  a custom-`root_url` bottle with a **single dash**. The workflow renames the
  first `--`→`-` before upload; publish the single-dash names. (Versioned
  names keep their `@`: `lld@22.1.7-22.1.7.<tag>.bottle.tar.gz`.)
- A 3rd-party tap must be trusted before `--build-bottle`:
  `brew tap <user>/<tap> "$PWD" && brew trust <user>/<tap>`.
- `cmake` (and any other `=> :build` dep) is pulled in only when building from
  source. A poured bottle skips it, so the **consumer** must install such
  build tools itself rather than relying on them as a side effect of the
  formula's source build.

## Common Mistakes

- Overwriting intentional local build flags with upstream values (e.g., reverting `OFF` to `ON` for `BUILD_SHARED_LIBS`)
- Keeping stale bottle hashes from a previous version that won't match the new source
- Missing new Homebrew DSL fields that upstream added (like `compatibility_version`)
- Forgetting to check dependency block changes (e.g., `uses_from_macos` replaced by platform-specific `on_linux` blocks)
- Skipping the archive step and overwriting the old version directly — this drops the ability for users to pin the previous version via `<formula>@<old-version>`
- Forgetting `keg_only :versioned_formula` in the archived file — without it, the versioned formula will conflict with the main formula's symlinks at install time
- Keeping the `bottle do` block in the archived file — those bottle URLs won't resolve under the new versioned filename
- Bumping the version of a self-built-bottle tap but not regenerating the bottles — leaving consumers compiling from source, or (worse) keeping a stale `bottle do` block whose `sha256`s no longer match the new source. See [Regenerating Bottles](#regenerating-bottles-self-built-tap).
