# Android ROM Manifest Management Guide

This document captures key learnings from systematic Android ROM manifest updates, specifically migrating LineageOS upstream branches from lineage-22.x to lineage-23.x while preserving team customizations.

## Overview

Android ROM manifests contain hundreds of repositories with complex branch dependencies. Bulk updates can break builds if not done carefully. This guide covers systematic approaches to safely update manifest files.

## Key Principles

### 1. Never Make Assumptions About Branch Patterns

**❌ Wrong Approach:**
```bash
# Don't assume all repositories follow the same pattern
sed -i 's/lineage-22.2/lineage-23.0/g' manifest.xml
```

**✅ Correct Approach:**
- Check each repository individually
- Verify branch availability before updating
- Understand that CAF repositories use SoC-specific branches

### 2. Preserve Team Commits

Before updating any upstream branch:
1. Check for team commits using committer emails
2. If team commits exist, create a revision branch (e.g., XOS-16.0.1)
3. Cherry-pick team commits to the new branch if needed

```bash
# Check for team commits
git log --pretty=format:"%ce" upstream/old-branch..local-branch | \
grep -E "@(example\.org|teamdomain\.dev)" | wc -l
```

### 3. Always Backup Before Bulk Operations

```bash
cp manifest.xml /tmp/manifest.xml.backup_$(date +%s)
```

## Repository Categories and Branch Patterns

### Standard LineageOS Repositories
- **Pattern:** `lineage-22.2` → `lineage-23.0`
- **Examples:** packages/apps/*, frameworks/*, system/*

### CAF (Code Aurora Forum) Repositories
CAF repositories use SoC-specific branches that must match the hardware:

- **sm8450:** `lineage-22.2-caf-sm8450` → `lineage-23.0-caf-sm8450`
- **sm8550:** `lineage-22.2-caf-sm8550` → `lineage-23.0-caf-sm8550`
- **sm8650:** `lineage-22.2-caf-sm8650` → `lineage-23.0-caf-sm8650`
- **sm8750:** `lineage-22.2-caf-sm8750` → `lineage-23.0-caf-sm8750`

**⚠️ Critical:** Never change `sm8550` repositories to use `sm8450` branches!

### Legacy and Special Branches
- **Legacy UM:** `lineage-22.2-legacy-um` → `lineage-23.0-legacy-um`
- **Legacy:** `lineage-22.2-legacy` → `lineage-23.0-legacy`
- **CAF:** `lineage-22.2-caf` → `lineage-23.0-caf`

## Systematic Update Process

### Phase 1: Analysis
1. **Extract all LineageOS upstream repositories:**
   ```bash
   grep 'upstream="https://github.com/LineageOS/' manifest.xml | \
   grep 'lineage-22' > repositories_to_check.txt
   ```

2. **Check branch availability for each repository:**
   ```bash
   # For each repo, verify the target branch exists
   git ls-remote --heads "$repo_url" "$target_branch"
   ```

3. **Check for team commits:**
   ```bash
   # Count commits by team members
   team_commits=$(git log --pretty=format:"%ce" upstream/old..local | \
   grep -E "@(example\.org|teamdomain\.dev)" | wc -l)
   ```

### Phase 2: Preparation
1. **Create revision branches for repositories with team commits:**
   ```bash
   git checkout -b XOS-16.0.1
   git push -u origin XOS-16.0.1
   ```

2. **Add revision attributes to manifest:**
   ```xml
   <project path="repo/path" name="repo_name" remote="XOS" 
            revision="XOS-16.0.1" 
            upstream="https://github.com/LineageOS/repo#lineage-23.0" />
   ```

### Phase 3: Systematic Updates
Update repositories in categories, never individually:

1. **Standard repositories without team commits**
2. **CAF repositories by SoC family** 
3. **Repositories with team commits (add revision bumps)**
4. **Legacy and special repositories**

### Phase 4: Verification
1. **Verify all branches exist:**
   ```bash
   # Extract all upstream references and verify
   grep 'upstream="https://github.com/LineageOS/' manifest.xml | while read line; do
     # Extract repo URL and branch, then verify with git ls-remote
   done
   ```

2. **Check for branch mismatches:**
   ```bash
   # Ensure sm8550 repos use sm8550 branches, not sm8450
   grep "sm8550.*sm8450" manifest.xml  # Should return nothing
   ```

## Common Mistakes and How to Avoid Them

### 1. Bulk SED Operations
**Problem:** Using sed to replace all occurrences without understanding context
```bash
# DANGEROUS - This corrupts the manifest
sed -i 's/#lineage-22.2/#lineage-23.0/g' manifest.xml
```

**Solution:** Use git to check original branches and update systematically:
```bash
git show HEAD:manifest.xml | grep "repo/path"  # Check original
# Then update with precise old_string → new_string replacements
```

### 2. SoC Branch Mismatches
**Problem:** Changing sm8550 repositories to use sm8450 branches
```xml
<!-- WRONG -->
<project path="hardware/qcom-caf/sm8550/audio/agm" 
         upstream="...#lineage-23.0-caf-sm8450" />

<!-- CORRECT -->
<project path="hardware/qcom-caf/sm8550/audio/agm" 
         upstream="...#lineage-23.0-caf-sm8550" />
```

### 3. Ignoring Non-Existent Branches
**Problem:** Updating to branches that don't exist
**Solution:** Always verify branch existence before updating

### 4. Losing Team Commits
**Problem:** Updating upstream without preserving local changes
**Solution:** Create revision branches and add `revision="XOS-16.0.1"` attribute

## Tools and Scripts

### Branch Verification Script
```bash
#!/usr/bin/env bash
manifest="manifest.xml"
grep 'upstream="https://github.com/LineageOS/' "$manifest" | while read line; do
    upstream=$(echo "$line" | grep -o 'upstream="[^"]*"' | cut -d'"' -f2)
    if [[ "$upstream" == *"#"* ]]; then
        repo_url=$(echo "$upstream" | cut -d'#' -f1)
        branch=$(echo "$upstream" | cut -d'#' -f2)
        path=$(echo "$line" | grep -o 'path="[^"]*"' | cut -d'"' -f2)
        
        echo "Checking $path -> $branch"
        if git ls-remote --heads "$repo_url" "$branch" 2>/dev/null | grep -q "$branch"; then
            echo "  ✅ Branch exists"
        else
            echo "  ❌ Branch NOT found: $branch"
        fi
    fi
done
```

### Team Commit Checker
```bash
#!/usr/bin/env bash
check_team_commits() {
    local repo_path="$1"
    local old_branch="$2"
    local new_branch="$3"
    
    if [ -d "$repo_path" ]; then
        cd "$repo_path"
        team_commits=$(git log --pretty=format:"%ce" \
        origin/"$old_branch"..HEAD 2>/dev/null | \
        grep -E "@(teamdomain\.org|example\.dev|teamname\.cc)" | wc -l)
        
        if [ "$team_commits" -gt 0 ]; then
            echo "⚠️ $repo_path has $team_commits team commits"
            return 1
        else
            echo "✅ $repo_path safe to update"
            return 0
        fi
    fi
}
```

## Final Checklist

Before committing manifest changes:

- [ ] All upstream branches verified to exist
- [ ] No SoC branch mismatches (sm8550 ≠ sm8450)
- [ ] Team commits preserved with revision branches
- [ ] Backup created before bulk operations
- [ ] Git diff reviewed for unexpected changes
- [ ] Test build started to verify manifest integrity

## Repository Categories Reference

### Safe to Update (No Team Commits Required)
- Most packages/apps/* repositories
- Most vendor/qcom/opensource/* repositories  
- Hardware repositories with available lineage-23.0 branches

### Requires Revision Bumps (Team Commits Present)
- Repositories where `git log upstream/old..local` shows team commits
- Add `revision="XOS-16.0.1"` attribute
- Ensure XOS-16.0.1 branch exists with team commits

### Cannot Update (No lineage-23.0 Available)
- Older hardware repositories (SDM845, SM7250, SM8150 GPS/display)
- Some vendor repositories without lineage-23.0 branches
- Legacy repositories that haven't been updated upstream

This guide ensures systematic, safe manifest updates while preserving the integrity of team customizations and build compatibility.