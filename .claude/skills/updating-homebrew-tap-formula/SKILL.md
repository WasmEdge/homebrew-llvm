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

3. **Remove the `bottle do` block.** Bottle filenames are derived from the formula name, so bottles built under the unversioned name don't resolve under the versioned name. Users build the archived version from source.
4. **Add `keg_only :versioned_formula`** in place of the removed bottle block. This prevents the versioned install from conflicting with the main formula's symlinks.
5. **Leave everything else (including `head`, `livecheck`, `link_overwrite`, dependency stanzas) byte-identical to the historical content.** The point of the archive is faithful historical preservation, not redesign.
6. Verify with `ruby -c Formula/<name>@<current-version>.rb`.

After archiving, proceed to update `Formula/<name>.rb` to the new upstream version per the main workflow.

## Common Mistakes

- Overwriting intentional local build flags with upstream values (e.g., reverting `OFF` to `ON` for `BUILD_SHARED_LIBS`)
- Keeping stale bottle hashes from a previous version that won't match the new source
- Missing new Homebrew DSL fields that upstream added (like `compatibility_version`)
- Forgetting to check dependency block changes (e.g., `uses_from_macos` replaced by platform-specific `on_linux` blocks)
- Skipping the archive step and overwriting the old version directly — this drops the ability for users to pin the previous version via `<formula>@<old-version>`
- Forgetting `keg_only :versioned_formula` in the archived file — without it, the versioned formula will conflict with the main formula's symlinks at install time
- Keeping the `bottle do` block in the archived file — those bottle URLs won't resolve under the new versioned filename
