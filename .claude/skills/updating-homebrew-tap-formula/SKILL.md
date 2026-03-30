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
2. **Fetch the upstream formula** from homebrew-core raw GitHub content:
   `https://raw.githubusercontent.com/Homebrew/homebrew-core/master/Formula/{first-letter}/{formula-name}.rb`
3. **Diff the two** and classify each change:

| Change Type | Action |
|------------|--------|
| Version URL + sha256 | Always update |
| New Homebrew DSL fields (e.g., `compatibility_version`) | Apply — these are framework changes |
| Dependency changes (e.g., `uses_from_macos` to `on_linux`) | Apply — upstream dependency decisions |
| Build flags you intentionally differ on (e.g., `BUILD_SHARED_LIBS`) | **Preserve local value** |
| Bottle hashes | Update if your tap uses upstream bottles; remove block if you build your own |
| Test block changes | Apply unless you have custom tests |

4. **Apply changes** selectively per the table above
5. **Verify** the final formula reads correctly

## Common Mistakes

- Overwriting intentional local build flags with upstream values (e.g., reverting `OFF` to `ON` for `BUILD_SHARED_LIBS`)
- Keeping stale bottle hashes from a previous version that won't match the new source
- Missing new Homebrew DSL fields that upstream added (like `compatibility_version`)
- Forgetting to check dependency block changes (e.g., `uses_from_macos` replaced by platform-specific `on_linux` blocks)
