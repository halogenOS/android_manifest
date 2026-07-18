# halogenOS (XOS) Project Documentation

## CRITICAL: Behavioral Requirements

**NEUTRAL NAMING.** Do NOT use "halogenOS", "halogen", "XOS" or similar in:
- File names, property names, attribute names, service names, AIDL names
- Function names, variable names
- Any identifier that would advertise or endorse the project

The code must stay **neutral and reusable**. Exceptions:
- Copyright headers: "The halogenOS Project" can be added
- Infrastructure-coupled code (e.g., `external/xos`) where it's unavoidable
- Comments where the mentioning it is necessary for understanding

**MANIFEST COMMIT MESSAGES.** When adding repos to the manifest:
- Use format: `Track <repo>` (e.g., `Track system/sepolicy`)
- Keep it short and simple

**COPYRIGHT HEADERS.** Use the current year in copyright headers added or updated by you (e.g., `Copyright (C) 2026 The halogenOS Project`). If there's already a year, extend it (like 2024-2026).

**NO `Change-Id:` LINES IN COMMIT MESSAGES.** There is no Gerrit workflow on this project. Do not add `Change-Id:` trailers when authoring or amending commits. If there is already one, keep it.

**SCRATCH FILES.** Keep temporary files (logs, research notes, plans) in a `.cache` directory at the top of the source tree (the tree root is not a git repo, so nothing picks it up). Do not put a `.cache` directory inside repos that contain `Android.bp` files - for those, fall back to `/tmp`.

**VERIFY BEFORE COPYING.** When copying code from a reference commit:
- Check if later commits modified or reverted what you're copying:
  ```bash
  git log --oneline <ref_commit>..HEAD -- path/to/file
  ```
- A field/function existing in an old commit doesn't mean it exists now - Google reverts features for ABI compatibility
- When in doubt, check the current header/source to confirm the code path still exists

**RESOLVE CHERRY-PICK CONFLICTS CAREFULLY.** When git conflicts occur during cherry-pick:
1. **ALWAYS check what THIS commit actually changes** before resolving:
   ```bash
   git show <commit_hash> -- path/to/file  # See only what THIS commit changes
   ```
2. **Don't blindly take "theirs"** - The incoming side may contain cumulative changes from multiple commits that we don't want
3. **NEVER use `git checkout --theirs` or `git checkout --ours`** - These replace the ENTIRE file, not just the conflicting hunks. This silently discards all other changes in the file. Always resolve conflicts manually by editing the conflict markers.
4. **Only include lines from THIS commit** - If the conflict shows 5 added lines but `git show` reveals the commit only adds 1 line, only add that 1 line. The other 4 lines came from other commits in upstream's history that we haven't picked

**PREFER UNCONDITIONAL OVER FLAG-GATED FOR ASB FIXES.** When a bulletin commit conflicts with a flag-gated version of the same fix already in HEAD:
- This happens when Google ships a fix behind an aconfig flag first, then later "ungates" it (makes it unconditional) in a security bulletin. The bulletin commit's header often reads `(cherry picked from commit <the original flagged commit>)`.
- The conflict can *look* like a clean duplicate ("HEAD already has it"), tempting a "keep HEAD" resolution. **Do NOT** — HEAD's flag gate means the fix only runs if the flag is enabled in `build/release/aconfig/...flag_values.textproto`, which is OUTSIDE the patch. Keeping HEAD can silently ship the security fix DISABLED.
- **Resolve toward unconditional**: take the incoming (ungated) condition so the fix always runs, regardless of release config. This matches Google's intent (they abandoned the flag) and removes the failure mode.
- Exception: keep the gate only if there's a concrete reason (e.g., the fix has a known regression being staged). When in doubt for a bulletin patch, go unconditional.
- A typo baked into a flag name (e.g. `reboke` for `revoke`) is load-bearing once shipped — it cannot be fixed without migration, and signals the flagged path may be under-tested. Another reason to prefer the ungated path Google moved to.

**HANDLE MODIFY/DELETE CONFLICTS PROPERLY.** When a file was modified in the commit but deleted in HEAD:
1. **NEVER just delete the file** - the change would be incomplete
2. **Investigate why** - the file may have been:
   - Rewritten in Kotlin (check for `.kt` equivalent)
   - Moved to a different location
   - Merged into another file
   - Refactored into a different architecture
3. **Check other sources**:
   - Does LineageOS have an updated version of this commit?
   - Does another ROM (The-Clover-Project, yaap, etc.) have a solution?
   - Search GitHub for the Change-Id
4. **Find where the logic moved**:
   ```bash
   # Search for key function/class names from the deleted file
   grep -r "functionName" --include="*.java" --include="*.kt" services/
   ```
5. **If stuck**, ask the user for guidance - don't just skip

**ONE CATEGORY AT A TIME.** When cherry-picking from a scratchpad (a prepared list of commits to apply, usually grouped by category):
- Only work on one category at a time unless explicitly asked to do more
- Wait for user confirmation before moving to the next category

**ANALYZE COMMITS SYSTEMATICALLY.** When picking commits from a previous XOS version:

1. **Check dependencies first** - Before picking a commit, check if earlier commits are prerequisites:
   ```bash
   git log xos/XOS-16.0 --oneline -- <file>
   git log xos/XOS-16.2 --oneline -- <file>
   ```
   Apply prerequisites first, oldest to newest.

2. **Use existing remotes** - Don't clone to /tmp. Check `git remote -v` and use existing upstream remotes.

3. **Batch-check commits** - Use oneliners to inspect multiple commits:
   ```bash
   echo "=== commit1 ===" && git show commit1 --stat --oneline && git log commit1 -1 --format="%b" | head -5 && echo "=== commit2 ===" && git show commit2 --stat --oneline && git log commit2 -1 --format="%b" | head -5
   ```

4. **Cross-reference with upstream** - Check if commits exist in LineageOS/AOSP by Change-Id:
   ```bash
   git log lineage/lineage-24.0 --oneline --grep="Change-Id-here"
   ```

5. **Evaluate each commit**:
   - Read the full diff with `git show <commit>`
   - Ask: Is this still relevant? Is it superseded? Does it apply to supported devices?
   - Skip: Non-Treble fixes, prebuilt vendor hacks, old shims that upstream dropped

6. **Check for sandboxedplay branches** - GmsCompat commits come from a separate branch:
   ```bash
   git branch -r | grep "sandboxedplay"
   ```
   - Skip gmscompat commits when picking from main XOS branch - they merge separately
   - Check PER REPO since different repos have different sandboxedplay branches

7. **Apply in correct order** - `git log` shows newest first, but cherry-pick oldest first:
   ```bash
   git log xos/XOS-16.0 --oneline | grep -n "commitA\|commitB"  # Higher line = older
   ```

**More bash tricks:**
- Limit `git log` depth for local searches: `git log -2000 <branch>` - AOSP repos have millions of commits, unbounded log will hang
- List commits in range with full hash + message: `git log --format='%H [%s]' --reverse A^..B`
- Escape special chars for sed replacement: `sed 's/[\/&]/\\&/g' | sed 's/\[/\\[/g' | sed 's/\]/\\]/g'`

**DEBUG BUILD ERRORS BY TRACING TO SOURCE.** When a build error occurs:
1. **Check if LineageOS has the same issue** - if we have a problem, they likely do too. Other ROMs can be checked as well.
2. **Search for TODOs** - the fix may already be documented but not yet implemented:
   ```bash
   grep -r "TODO.*<keyword>" product/halogenOS/
   ```
3. **Find the commit that introduced/removed the code**:
   ```bash
   git blame -L<start>,<end> <file>    # Which commit touched these lines
   git log -S "code_snippet" -- file   # Commits that added/removed this text
   ```
4. **Compare commit sizes** - 500+ lines vs upstream's 8 = bad merge conflict resolution
5. **Check for reverts** - `git log --grep="Revert" -- path/to/file`
6. **Before manually fixing**, find the proper commit to cherry-pick or reference

Example workflow:
```bash
# Find commits that added/removed specific code
git log -S "BUILT_OTATOOLS_PACKAGE" -- core/Makefile

# Blame specific lines to find which commit touched them
git blame -L3200,3215 core/Makefile

# Compare our commit vs LineageOS
git show our_commit --stat    # 988 lines changed
git show lineage_commit --stat  # 8 lines changed  <- bad conflict resolution!

# Check if LineageOS reverted something
git log lineage/lineage-23.2 --grep="Revert.*super image"
```

**FIX BAD CHERRY-PICKS WITH REVERT + REBASE.** When a cherry-pick has bad conflict resolution:
1. `git stash` any uncommitted changes
2. `git revert -n <bad_commit>` - revert without committing
3. `git add -A && git commit --fixup=<bad_commit>`
4. `git rebase --autosquash <bad_commit>^` (no `-i` needed for autosquash)
5. If empty commit results: `git reset HEAD^ && git rebase --continue`

**CRITICAL RULES FOR EDITING REBASE TODO FILES:**
- **SURGICAL EDITS ONLY** - Do NOT rewrite the entire file. Use targeted Edit operations.
- **PRESERVE EXACT FORMAT** - Do NOT remove `#` symbols, change spacing, or modify lines you're not reordering
- **NO RANDOM CHANGES** - If a line has `pick abc123 # some message # empty`, keep it EXACTLY that way unless you're specifically changing that line's position or action
- **MULTIPLE EDITS IN ONE MESSAGE** - When making several changes, use multiple Edit tool calls in a single message to avoid partial state
- **VERIFY BEFORE SIGNALING** - Read the file after editing to confirm changes are correct before signaling the editor wrapper to continue (e.g. sending SIGUSR1)

**"EMPTY COMMIT" DURING REBASE.** When git says "The previous cherry-pick is now empty", there are three possible causes:
1. **rerere auto-resolution** - Git's rerere (reuse recorded resolution) cached a previous conflict resolution and auto-applied it, making the diff empty. Fix: `git checkout -m <file>` to recreate the real conflict.
2. **Intentionally empty commit** - Marker commits or structural commits that are meant to be empty. Fix: Use `--keep-empty` flag with rebase.
3. **Actually became empty** - The changes were reverted or superseded by later commits in history. This is legitimate - the commit genuinely has no effect.

**FOLLOW SQUASH/FIXUP INSTRUCTIONS LITERALLY.** When commits in the scratchpad have comments like `# squash into <hash>` or `# fixup into <hash>`:
- `# squash into` → use `squash` action (preserves commit message)
- `# fixup into` → use `fixup` action (discards commit message)
- Do NOT manually edit the changes in - this loses the proper structure and causes issues later
- The target commit should be `pick`, followed by `squash`/`fixup` lines for each auxiliary commit

**DEBUG MAKE/SOONG CONFLICTS.** For "overriding commands for target X, previously defined at Y":
1. Read BOTH files at the exact line numbers from the error
2. For Soong-generated files (`out/soong/installs-*.mk`), the line shows the source module
3. Track back to the Android.bp that defines the conflicting module

Example:
```bash
# Error says: Makefile:3203 conflicts with out/soong/installs-sdk_phone64_x86_64.mk:46629

# Read the Soong-generated file at that line
sed -n '46625,46635p' out/soong/installs-sdk_phone64_x86_64.mk

# This shows something like:
# out/target/.../adb_debug.prop: out/soong/.intermediates/system/core/rootdir/adb_debug.test_harness.prop/...
# Now you know the source is system/core/rootdir/Android.bp

# Grep to find more context
grep -n "adb_debug" out/soong/installs-sdk_phone64_x86_64.mk | head -10
```

**PRESERVE AUTHORSHIP.** When applying changes from other repositories:
- Cherry-pick commits instead of manually copying code - this preserves the original author
- Use existing remotes (`git remote -v`) or add one using helpers (see "Git Remote Conventions" below)
- NEVER manually edit files to apply someone else's changes - that erases their authorship

**NEVER REBASE WHEN THE LOCAL UNPUSHED COMMITS CONTAIN MERGES.** This rule is about your LOCAL unpushed commits, not the remote. This is not needed for regular work, usually only in the context of bringup or tracking a new repo. Before `git pull --rebase` or `git rebase`:
1. **Check whether any of your LOCAL unpushed commits are merge commits**:
   ```bash
   git log --oneline --merges <remote>/<branch>..HEAD
   ```
2. If local unpushed merges exist, `git pull --no-rebase` (regular merge) instead — rebasing would flatten them, destroying the merge history and potentially duplicating commits.
3. If the command returns nothing (plain linear commits), **always rebase**. This is the normal case on our own branches (`XOS-*`) where unpushed work is linear — `git pull --rebase` is the default, no check needed in practice.
4. Remote state (whether the remote branch has new work) is irrelevant to this rule. Push rejection with "remote contains work you don't have" only tells you to integrate; rebase vs merge is decided by whether YOUR local commits contain merges.

**VERIFY UPSTREAM BRANCHES WITH GITHUB API.** When adding or updating projects with `upstream="..."`:
- DO NOT guess branch names (e.g., don't assume `16` exists when XOS-16.0 used `XOS-15.2`)
- Use GitHub API to check available branches:
  ```bash
  curl -s "https://api.github.com/repos/LineageOS/android_packages_apps_Aperture/branches" | grep -o '"name": "[^"]*"'
  ```
- Match the branch to the XOS version:
  - XOS-16.2 (QPR2/BP4A) → `16-qpr2`, `lineage-23.2`
  - XOS-17.0 (QPR0/CP2A) → `17`, `lineage-24.0`
- For LineageOS, prefer `lineage-24.0` over `lineage-23.x`

**CHECK PREVIOUS XOS MANIFEST BEFORE ADDING PROJECTS.** When adding a project to the current XOS manifest:
- ALWAYS check how the project was tracked in the previous XOS version first:
  ```bash
  cd manifest && git fetch XOS <previous-branch> && git show XOS/<previous-branch>:snippets/XOS.xml | grep "project_name"
  ```
- If previous version used `merge-aosp="true"`: Use the same attribute, create branch from AOSP tag, cherry-pick needed commits
- If previous version used `upstream="..."`: Use the same pattern with updated branch (e.g., `lineage-23.2` → `lineage-24.0`)
- If the project is NEW: Decide based on whether we need AOSP base + cherry-picks (`merge-aosp="true"`) or direct tracking (`upstream="..."`)
- NEVER create branches directly from upstream (e.g., `git checkout -b XOS-17.0 lineage/lineage-24.0`) for `merge-aosp="true"` projects

---

## Overview

halogenOS (XOS) is a custom Android ROM based on AOSP (Android Open Source Project). The current bringup is for Android 17 QPR0 (CP2A) based on `android-17.0.0_r1`.

### Key Characteristics
- Based directly on AOSP, not LineageOS
- Uses hardware/device support from LineageOS selectively (only what's needed)
- Maintains its own manifest, build tools, and product configuration
- Git server: `git.halogenos.org`
- GitLab API is used for repository management
- Uses `$(CUSTOM_PRODUCT_DIR)` variable instead of hardcoded paths (expands to `product/halogenOS`) – this will change soon

### Working with the Codebase
**AOSP is huge** - avoid global searches across the entire tree or even large directories like `frameworks/base/`. They take a very long time and are usually unnecessary. Instead:
- Use `ls` and navigate iteratively to find what you need
- Use path discriminators to narrow searches to specific subdirectories
- Know the directory structure - most changes happen in predictable locations
- When you need to search, target the most specific directory possible

### Branch Naming Convention
- Main branches follow pattern: `XOS-<version>` (e.g., `XOS-16.0`, `XOS-16.2`, `XOS-17.0`)
- Minor version branches (e.g., `XOS-16.0.1`) indicate a rebase was performed to avoid breaking commit history

---

## Directory Structure

### Core XOS Directories

| Path | Description |
|------|-------------|
| `manifest/` | Main manifest repository with AOSP + XOS snippets |
| `manifest/snippets/XOS.xml` | XOS-specific project definitions (remote, core repos) |
| `manifest/snippets/remove.xml` | Projects removed from AOSP |
| `product/halogenOS/` | Main product configuration (replaces vendor/lineage) - remove this when migrated |
| `external/xos/` | XOS tools and development environment |
| `external/xos/xostools/` | Build scripts, helpers (includes `build` command) |
| `external/xos/xostools-ng/` | Python tools for merging, mirroring, bulletins |
| `external/xos/devshell/` | Nix-based development environment |

### Hardware Directories (from LineageOS/CLO)

| Path | Description |
|------|-------------|
| `device/lineage/sepolicy/` | SELinux policies from LineageOS |
| `hardware/lineage/` | LineageOS hardware abstraction (compat, interfaces) |
| `hardware/qcom-caf/` | Qualcomm CodeLinaro (CLO, formerly CAF) hardware |

### Device-Specific (fetched via roomservice/dependencies)

Device trees, kernels, and vendor blobs are **not** part of the base manifest. They are fetched on-demand via:
- `breakfast <device>` - triggers `roomservice.py`
- Dependencies defined in `aosp.dependencies` (or `lineage.dependencies` fallback) in device trees
- Added to `.repo/local_manifests/roomservice.xml`

---

## Manifest System

### Structure
The manifest (`manifest/default.xml`) contains the full AOSP project list with XOS modifications applied via snippets:

```
manifest/
├── default.xml          # Full AOSP manifest (android-17.0.0_r1)
├── snippets/
│   ├── XOS.xml          # XOS remote and core projects
│   └── remove.xml       # Projects removed from AOSP
└── README.md
```

### XOS Remote Definition (snippets/XOS.xml)
```xml
<remote
  name="XOS"
  fetch="https://git.halogenos.org/halogenOS"
  pushurl="git@git.halogenos.org:halogenOS"
  revision="refs/heads/XOS-17.0"
/>
```

### Core XOS Projects (always included)
- `external/xos` (android_external_xos)
- `manifest` (android_manifest) - with `merge-aosp="true"`
- `product/halogenOS` (android_product_halogenOS)

### Removing AOSP Projects
To remove a project from AOSP, add it to `snippets/remove.xml`:
```xml
<remove-project path="path/to/project" name="platform/path/to/project" />
```

---

## Build System

### Environment Setup

**Using Nix (recommended for interactive use):**
```bash
nix develop path:external/xos/devshell
# Or with direnv:
direnv allow
aosp-env
```

**Important:** Always use the `path:` prefix when working with local Nix flakes, otherwise Nix won't pick up local changes:
```bash
# Wrong - won't see local changes:
nix eval .#packages.x86_64-linux.execShell --apply 'x: "ok"'

# Correct - uses local changes:
nix eval 'path:.#packages.x86_64-linux.execShell' --apply 'x: "ok"'
```

**Standard setup:**
```bash
source build/envsetup.sh
```

**Oneliner to trigger a build (device = codename, type = user, userdebug, eng):**
```bash
mkdir -p .cache/logs && : > .cache/logs/build.log && nix run 'path:external/xos/devshell#execShell' -- -c 'source local.conf && source build/envsetup.sh && build full aosp_<device>-cp2a-<type> noclean' > .cache/logs/build.log
```

`local.conf` is an optional user-local file at the tree root (exports like `OUT_DIR`); drop `source local.conf &&` if you don't have one.

### Key Environment Variables
Set in `product/halogenOS/vendorsetup.sh`:
- `CUSTOM_PRODUCT` = "halogenOS"
- `CUSTOM_PRODUCT_DIR` = "product/halogenOS"
- `CUSTOM_PRODUCT_NAME` = "halogenOS"
- `ROM_VERSION` = extracted from manifest (e.g., "XOS-17.0")
- `TOP` = source root directory

### Build Commands

**XOS `build` command** (from xostools.sh):
```bash
build full aosp_<device>-<build_id>-userdebug
# Example:
build full aosp_Pong-cp2a-userdebug

# Dirty build (skip clean):
build full aosp_Pong-cp2a-userdebug noclean
```

The `build` command:
- Automatically determines optimal thread count
- Runs `lunch` and `breakfast` as needed
- Runs `make clean` unless `noclean` is specified
- Executes build with `m --skip-soong-tests`

**Standard AOSP commands also work:**
```bash
lunch aosp_<device>-<build_id>-userdebug
m bacon
```

### Device Initialization
```bash
breakfast <device>
# Triggers roomservice.py to fetch device tree and dependencies
```

---

## Product Configuration ($(CUSTOM_PRODUCT_DIR)/)

**Important:** Always use `$(CUSTOM_PRODUCT_DIR)` in makefiles instead of hardcoding `product/halogenOS`. This variable is set in `vendorsetup.sh` and ensures portability.

### Directory Structure
```
product/halogenOS/
├── config/
│   ├── common.mk              # Main common configuration
│   ├── common_mobile.mk       # Mobile device config
│   ├── common_full_phone.mk   # Full phone config
│   ├── branding.mk            # Version/branding properties
│   ├── BoardConfigKernel.mk   # Kernel build configuration
│   ├── BoardConfigSoong.mk    # Soong build configuration
│   ├── overlays.mk            # Runtime Resource Overlays
│   ├── apps.mk                # Included apps
│   └── permissions/           # Permission XMLs
├── build/
│   └── tools/
│       └── roomservice.py     # Device tree fetcher
├── overlay/                   # Static overlays
├── overlays/                  # Runtime Resource Overlays
├── bootanimation/             # Boot animation resources
├── charger/                   # Off-mode charging resources
├── fonts/                     # Custom fonts
├── prebuilt/                  # Prebuilt binaries/configs
├── release/                   # Release configuration
└── vendorsetup.sh             # Environment setup script
```

### Device Product Makefile Pattern
Device makefiles inherit from XOS configs:
```makefile
# Inherit common XOS configuration
$(call inherit-product, $(CUSTOM_PRODUCT_DIR)/config/common_full_phone.mk)

PRODUCT_NAME := aosp_<Device>
PRODUCT_DEVICE := <Device>
PRODUCT_MANUFACTURER := <Manufacturer>
PRODUCT_BRAND := <Brand>
PRODUCT_MODEL := <Model>
```

---

## Dependencies System (roomservice.py)

### How It Works
1. `breakfast <device>` triggers `roomservice.py`
2. Searches `git.halogenos.org` GitLab API for device repository
3. Reads `aosp.dependencies` from device tree (falls back to `lineage.dependencies`)
4. Recursively fetches all dependencies
5. Adds projects to `.repo/local_manifests/roomservice.xml`
6. Runs `repo sync` for new projects

### Dependency File Format (aosp.dependencies)
```json
[
  {
    "repository": "android_vendor_<manufacturer>_<device>",
    "target_path": "vendor/<manufacturer>/<device>",
    "revision": "XOS-17.0"
  },
  {
    "repository": "android_kernel_<manufacturer>_<soc>",
    "target_path": "kernel/<manufacturer>/<soc>",
    "revision": "XOS-17.0"
  }
]
```

### Repository Naming Convention
- Device trees: `android_device_<manufacturer>_<device>`
- Vendor blobs: `android_vendor_<manufacturer>_<device>`
- Kernels: `android_kernel_<manufacturer>_<soc>`
- Hardware: `android_hardware_<vendor>`

### Branch Fallback
If the latest branch doesn't exist, roomservice falls back to branches defined in `custom_default_fallback_revisions` (e.g., `XOS-16.2`).

---

## Manifest Project Patterns (XOS.xml)

### Project Attributes

**`merge-aosp="true"`** - Project is a fork of AOSP with XOS changes merged on top:
```xml
<project path="frameworks/base" name="android_frameworks_base" remote="XOS" merge-aosp="true" />
```

**`upstream="..."`** - Tracks upstream source (LineageOS, GrapheneOS, etc.):
```xml
<project path="hardware/lineage/compat" name="android_hardware_lineage_compat" remote="XOS"
         upstream="https://github.com/LineageOS/android_hardware_lineage_compat#lineage-24.0" />
```

**`revision="XOS-16.0.1"`** - Uses specific branch (often after a rebase):
```xml
<project path="packages/apps/Etar" name="android_packages_apps_Etar" remote="XOS"
         revision="XOS-16.0.1" upstream="https://github.com/LineageOS/android_packages_apps_Etar#lineage-23.0" />
```

### Snippet Files

| File | Purpose |
|------|---------|
| `snippets/XOS.xml` | XOS projects (forks, new repos, LineageOS/CLO hardware) |
| `snippets/remove.xml` | AOSP projects **replaced** by XOS versions |
| `snippets/aosp-remove.xml` | AOSP projects **removed** without replacement |

### Categories of Projects in XOS.xml

1. **AOSP Forks with XOS changes** (`merge-aosp="true"`):
   - `frameworks/base`, `frameworks/av`, `frameworks/native`
   - `system/core`, `system/sepolicy`, `build/make`, `build/soong`
   - Various `packages/apps/*` and `packages/modules/*`

2. **LineageOS Hardware/Device Support**:
   - `device/lineage/sepolicy`
   - `hardware/lineage/compat`, `hardware/lineage/interfaces`
   - Prebuilts: `prebuilts/extract-tools`, `prebuilts/tools-lineage`

3. **Qualcomm/CLO Hardware** (`hardware/qcom-caf/`):
   - Per-SoC directories: `msm8953`, `msm8996`, `sdm845`, `sm8150`, `sm8450`, `sm8550`, `sm8650`, `sm8750`
   - Each contains: `audio/`, `display/`, `media/` (varies by SoC)
   - Common: `hardware/qcom-caf/common` with linkfiles for build guards

4. **GrapheneOS Components**:
   - `packages/apps/GmsCompat`, `packages/apps/AppCompatConfig`
   - `external/AppCompatConfig`, `external/GmsCompatConfig`
   - `packages/apps/Seedvault`

5. **XOS-specific**:
   - `external/xos` (tools)
   - `product/halogenOS` (product config)
   - `packages/overlays/Custom`, `packages/overlays/accents`
   - `external/custom-fonts`, `external/custom-preference`

---

## Commit Message Patterns

### Feature/Change Commits
Commits typically use a prefix indicating the component:
```
gmscompat: add stub for TelephonyManager.getImei()
SystemUI: Reduce screenshot dismiss delay to 3 seconds
init: set build_tags to release-keys
sepolicy: Label pihooks gms disable props
```

### Fixup Commits
Used for corrections to previous commits (often squashed later):
```
fixup! infrastructure for custom handling of known packages
fixup! [SQUASH] Introduce PropImitationHooks
```

Do NOT write fixup commits yourself – these are written by `git commit --fixup`.

### Merge Commits

**ASB merges:**
```
Merge branch 'android-16.0.0_r1-ASB_2025-09-01' into XOS-16.0
```

**Upstream merges (GrapheneOS, LineageOS):**
```
Merge remote-tracking branch 'upstream/16' into XOS-16.0
Merge remote-tracking branch 'upstream/lineage-23.0' into XOS-16.0
```

---

## Key Features & Components

### GmsCompat (Sandboxed Google Play)
From GrapheneOS - allows running Google Play Services as a regular app:
- `packages/apps/GmsCompat` - Main app
- `external/GmsCompatConfig` - Configuration
- `frameworks/base` gmscompat: prefixed commits for hooks/stubs
- Provides stubs for privileged APIs (IMEI, serial, etc.)

### Updating Sandboxed Play Branches
GmsCompat commits live on `android-17-sandboxedplay` branches (per-repo; the name tracks the Android version/QPR, e.g. `android-16-qpr2-sandboxedplay` on XOS-16.2). To update them with new commits from XOS-17.0:

1. **Find the latest author date across all sandboxedplay branches**:
   ```bash
   git log -1 --format="%ct" XOS/android-17-sandboxedplay
   ```

2. **Check for gmscompat-related commits on XOS-17.0 after that date**:
   ```bash
   git log --since="<date>" --format="%h %s" XOS-17.0 --grep="gmscompat"
   ```

3. **Verify commits are NOT already on sandboxedplay by message** (hashes differ because branches base off different commits):
   ```bash
   git log --oneline -1 --grep="<commit subject>" XOS/android-17-sandboxedplay
   ```

4. **Check topological order and dependencies**. A commit that renames a method may depend on a prior commit that added it. Cherry-pick in dependency order (oldest first) or conflicts will occur:
   ```bash
   git log --oneline --graph <oldest>..<newest>
   ```

5. **Cherry-pick to sandboxedplay**:
   ```bash
   git fetch XOS android-17-sandboxedplay
   git checkout -B android-17-sandboxedplay XOS/android-17-sandboxedplay
   git cherry-pick <commit>
   git push XOS android-17-sandboxedplay
   ```

### SystemUI Quick Settings

**Aconfig Flags for QS Features:**
- QS feature flags are defined in `packages/SystemUI/aconfig/systemui.aconfig`
- Flag values are set in `product/halogenOS/release/aconfig/cp2a/com.android.systemui/`
- Format for enabling flags in `flag_values.textproto`:
  ```
  flag_value {
    package: "com.android.systemui"
    name: "flag_name"
    state: ENABLED
    permission: READ_ONLY
  }
  ```

**QsInCompose (New QS UI):**
- When `QsInCompose` is enabled, the system uses `quick_settings_tiles_new_default` instead of `quick_settings_tiles_default`
- Both resources should be overridden in the SystemUI overlay if customizing default tiles
- Check `QSHost.getDefaultSpecs()` to see which resource is used

**Custom QS Tile Dependencies:**
- Tiles use Dagger injection via module classes (e.g., `LineageModule.kt`)
- Tile configs are provided via `@Provides @IntoMap @StringKey(TILE_SPEC)`
- String resources typically go in `cm_strings.xml` (or similar)
- Stock tile list in `config.xml` must include new tile specs

**Cherry-picking QS Tiles from Other ROMs:**
- **Avoid LineageOS SDK dependencies**: Commits using `org.lineageos.internal.*` require LineageOS SDK
- **AIDL interfaces are fine**: `vendor.lineage.*.*I*` imports are from `hardware/lineage` (AIDL), not SDK
- **Preferred sources for vanilla tiles**: Clover Project, DerpFest, or previous XOS versions
- **Check imports carefully**: Look for `org.lineageos.internal` in tile source files before picking

**QS Tile Behavior Patterns:**
- `handleClick()` - Primary click action
- `handleSecondaryClick()` / `handleToggleClick()` - Secondary/toggle action
- `handleLongClick()` - Long press action (usually opens settings)
- `isAvailable()` - Controls whether tile appears in editor (often gated by aconfig flags)
- Dialog-based actions must run on main thread (use `withContext(mainContext)` in coroutines)

---

## Cherry-Picking from Other ROMs

### ROM Preference Order

When cherry-picking commits or features from other ROMs, prefer sources in this order:

1. **LineageOS** - Primary upstream for most features, BUT avoid commits using LineageOS SDK (`org.lineageos.internal.*`)
2. **DerpFest** - Often has clean, vanilla implementations without SDK dependencies
3. **The-Clover-Project** - Good alternative for vanilla implementations
4. **YAAP** - Another source for clean implementations
5. **crDroid** - Additional fallback option
6. **Previous XOS version** - Last resort, may have outdated code

**Why this order matters:**
- LineageOS has the most mature codebase but ties features to their SDK
- DerpFest and Clover maintain vanilla AOSP compatibility
- Using SDK-dependent code requires porting the entire SDK (not worth it for individual features)

### Identifying SDK Dependencies

**Imports to AVOID (require LineageOS SDK):**
```java
import org.lineageos.internal.*
import lineageos.app.*
import lineageos.hardware.*
import lineageos.providers.*              // includes LineageSettings
```

**Common SDK patterns to watch for:**
- `LineageSettings.System.*` or `LineageSettings.Secure.*` → use `Settings.System.*` / `Settings.Secure.*` instead
- `org.lineageos.platform.internal.R.*` → use `com.android.internal.R.*` or SystemUI resources instead

**Imports that are SAFE (from hardware/lineage AIDL):**
```java
import vendor.lineage.powershare.IPowerShare  // AIDL interface
import vendor.lineage.touch.*                  // AIDL interfaces
import vendor.lineage.fastcharge.*            // AIDL interfaces
```

The pattern `vendor.lineage.*.*I*` indicates AIDL interfaces from `hardware/lineage/interfaces/`, which are standalone and do not require the SDK.

### Common Remotes in XOS Repositories

| Remote Name | Description |
|-------------|-------------|
| `xos` / `XOS` | XOS GitLab (case-sensitive!) |
| `los` | LineageOS GitHub |
| `aosp` | AOSP upstream |
| `upstream` | Primary upstream (varies by repo) |
| `clover` | The-Clover-Project |
| `derpfest` | DerpFest-AOSP |
| `yaap` | YAAP |
| `graphene` | GrapheneOS |

### Branch Naming by ROM

| ROM | Current Branch Pattern | Example |
|-----|------------------------|---------|
| LineageOS | `lineage-24.Y` | `lineage-24.0` (QPR0) |
| DerpFest | `16.2` | `16.2` (no Android 17 branch yet) |
| Clover | `17` | `17` |
| YAAP | `sixteen` | `sixteen` (no Android 17 branch yet) |
| GrapheneOS | `17` | `17` |
| crDroid | `16.0` | `16.0` (no Android 17 branch yet) |

Last verified 2026-07 - re-check with the GitHub API (see "VERIFY UPSTREAM BRANCHES") before relying on this table.

**LineageOS branch versions:**
- `lineage-23.0` - Android 16 QPR0
- `lineage-23.1` - Android 16 QPR1
- `lineage-23.2` - Android 16 QPR2
- `lineage-24.0` - Android 17 QPR0 (current)

### Workflow for Cherry-Picking

1. **Check existing remotes**: `git remote -v`
2. **Add needed remote**: `git remote add <name> <url>` or use helpers like `addLOS`
3. **Fetch the branch**: `git fetch <remote> <branch>`
4. **Check commit for SDK deps**: `git show <commit> | grep -E "org\.lineageos|lineageos\.(app|hardware)"`
5. **If SDK-dependent**: Try the next ROM in preference order
6. **If clean**: Cherry-pick with `git cherry-pick <commit>`

---

## Git Remote Conventions

**ALWAYS check existing remotes first** with `git remote -v` before adding new ones or cloning separate repos. Most XOS repositories already have upstream remotes configured (e.g., `lineage`, `upstream`, `aosp`). Use these instead of cloning to `/tmp/`.

Repositories typically have these remotes configured:

| Remote | Purpose |
|--------|---------|
| `XOS` / `xos` | Primary XOS GitLab (fetch: https, push: ssh) |
| `xosgh` | XOS GitHub mirror |
| `aosp` | AOSP upstream |
| `upstream` | Primary upstream (GrapheneOS, LineageOS, etc.) |
| `lineage` | LineageOS upstream (in product/halogenOS → vendor/lineage) |
| Other ROMs | For cherry-picking (yaap, Neoteric-OS, etc.) |

Example from `frameworks/base`:
```
XOS     https://git.halogenos.org/halogenOS/android_frameworks_base (fetch)
XOS     git@git.halogenos.org:halogenOS/android_frameworks_base (push)
aosp    https://android.googlesource.com/platform/frameworks/base
upstream https://github.com/GrapheneOS/platform_frameworks_base.git
```

### Remote Name Case Sensitivity
- Remote names are **case-sensitive** in git
- `XOS` and `xos` are different remotes
- Check with `git remote -v` to see exact names before using

### Adding Remotes

Use helper functions from `external/xos/xostools/` (available after `source build/envsetup.sh`):

```bash
# To add XOS remote with correct SSH push URL:
bash -c 'cd /path/to/source-tree && source build/envsetup.sh 2>/dev/null && cd path/to/repo && addXos'
```

Available functions:
- `addXos` - XOS GitLab with SSH push URL
- `addLOS` - LineageOS
- `addAosp` - AOSP
- `addOther <org>` - other ROMs with `android_` prefix (e.g., `addOther yaap`)
- `addOtherShort <org>` - other ROMs without `android_` prefix

### Common ROM Repository URLs

When cherry-picking commits, check these ROMs for updated versions:

| ROM | GitHub Organization | Branch Pattern | Example frameworks/base URL |
|-----|---------------------|----------------|----------------------------|
| LineageOS | `LineageOS` | `lineage-XX.Y` | `https://github.com/LineageOS/android_frameworks_base` |
| GrapheneOS | `GrapheneOS` | `XX-qprY` | `https://github.com/GrapheneOS/platform_frameworks_base` |
| DerpFest-AOSP | `DerpFest-AOSP` | `16`, `15` | `https://github.com/DerpFest-AOSP/android_frameworks_base` |
| YAAP | `yaap` | `sixteen`, `fifteen` | `https://github.com/yaap/frameworks_base` |
| CalyxOS | `CalyxOS` | `android16-qprX` | `https://gitlab.com/CalyxOS/platform_frameworks_base` |
| The-Clover-Project | `The-Clover-Project` | TBD | `https://github.com/The-Clover-Project/frameworks_base` |
| crDroid | `crdroidandroid` | `15.0` | `https://github.com/crdroidandroid/android_frameworks_base` |

Note: Repository naming varies - some use `android_` prefix (LineageOS), some use `platform_` prefix (GrapheneOS), some use neither (YAAP).

### Fetching AOSP Tags
When you need to compare with AOSP:
```bash
# Fetch a specific tag to FETCH_HEAD:
git fetch aosp android-17.0.0_r1

# Then use FETCH_HEAD to reference it:
git show FETCH_HEAD:path/to/file
git log HEAD..FETCH_HEAD -- path/to/file
git log FETCH_HEAD..HEAD -- path/to/file  # Our commits not in AOSP
```

### Fetching XOS Branches
```bash
# Fetch specific branch:
git fetch xos XOS-16.2

# Reference as xos/XOS-16.2 after fetching:
git show xos/XOS-16.2:snippets/XOS.xml

# Or use FETCH_HEAD immediately after fetch:
git fetch xos XOS-16.2 && git show FETCH_HEAD:file
```

### The manifest/ Directory
- The `manifest/` directory in the source tree is the **working manifest**
- Changes here are used by `reticulate-splines` immediately (no push needed)
- `.repo/manifests/` is what `repo` uses - separate from `manifest/`
- Edit `manifest/snippets/XOS.xml`, run reticulate-splines, then commit and push

---

## Reticulate Splines Tool

The `reticulate-splines` tool automates branch creation for XOS repos:

```bash
nix run path:external/xos/xostools-ng#reticulate-splines
```

### What It Does
1. Parses `manifest/snippets/XOS.xml` for projects with `upstream="..."` attribute
2. For each project:
   - Clones/initializes the repo if needed
   - Fetches from the upstream URL
   - Checks out the upstream branch → creates XOS-17.0 branch
   - Pushes to XOS remote
3. Skips projects where XOS-17.0 branch already exists

### Using It for New Repos
To quickly create XOS-17.0 branches from XOS-16.2 (for repos without external upstream):

1. Add to manifest with temporary upstream pointing to XOS-16.2:
   ```xml
   <project path="packages/overlays/Custom" name="android_packages_overlays_Custom" remote="XOS"
            upstream="https://git.halogenos.org/halogenOS/android_packages_overlays_Custom#XOS-16.2" />
   ```

2. Run reticulate-splines (no need to push manifest first - it reads local files)

3. Remove the temporary upstream attribute after branches are created:
   ```xml
   <project path="packages/overlays/Custom" name="android_packages_overlays_Custom" remote="XOS" />
   ```

4. Commit and push manifest

### Project Types
- **With `merge-aosp="true"`**: AOSP forks - reticulate-splines skips these (handled differently)
- **With `upstream="..."`**: Will be processed by reticulate-splines
- **Without upstream**: Pure XOS repos - need manual branch creation or temporary upstream trick

### List Changes Tool

Shows unpushed commits and uncommitted changes across all repos:

```bash
nix run 'path:external/xos/xostools-ng#list-changes'
```

---

## Release Configuration

### Flag System (product/halogenOS/release/)
```
release/
├── release_config_map.textproto   # Maps release configs
├── release_configs/cp2a.textproto # cp2a release config
├── build_config/cp2a.scl          # Build config for cp2a release
├── flag_declarations/             # Custom flags (textproto)
│   ├── RELEASE_DEVICE_MAINTAINERS.textproto
│   └── RELEASE_PLATFORM_SECURITY_PATCH_OVERRIDE.textproto
├── flag_values/cp2a/              # Flag value overrides (per build id)
└── aconfig/cp2a/                  # Android config values (per build id)
```

### Custom Flags
- `RELEASE_DEVICE_MAINTAINERS` - Device maintainer info (base64 encoded in build)
- `RELEASE_PLATFORM_SECURITY_PATCH_OVERRIDE` - Override security patch date

---

## AOSP Tips

### Branch Recommendations
- Use `android-latest-release` instead of `aosp-main` or `master` for building and contributing to AOSP
- Google officially recommends `android-latest-release` as the stable development branch

---

## Project Wiki

The public wiki at https://github.com/halogenOS/android_manifest/wiki documents XOS troubleshooting, build/runtime error fixes, and device porting guides. Check it when hitting build or runtime errors. If you work with a local clone of the wiki, pull the latest changes before reading.

### Writing to the wiki's `Fixing-errors` page

When documenting build error fixes:
- Keep entries concise - no step-by-step commands, just the error and solution
- Keep it general - the wiki is for everyone, not XOS-specific instructions
- The solution should be just the commit link - whether to fork, cherry-pick, or use upstream is the reader's decision
- Follow the existing pattern in the file: error message block, brief explanation, solution link
