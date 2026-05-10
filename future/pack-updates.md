# Pack Update Functionality

## Overview

Currently, sir.sh supports installing and removing packs, but updating packs requires manual removal and reinstallation. This enhancement would add `sir recipes:update` functionality to seamlessly update installed packs.

## Current State

- `sir recipes:install` - Install packs from Git
- `sir recipes:list` - List installed packs
- `sir recipes:remove` - Remove installed packs
- `sir recipes:update` - **Stubbed** (shows message to manually remove/reinstall)

## Proposed Implementation

### Command Signature

```bash
# Update all packs
sir recipes:update

# Update specific pack
sir recipes:update <pack-id>

# Update with specific version/ref
sir recipes:update <pack-id> --ref=v2.0.0

# Update project packs
sir recipes:update --project
```

### Update Process

1. **Check Current Version**
   - Read current ref/commit from `packs.lock.json`
   - Fetch latest commits from remote

2. **Detect Changes**
   - Compare current commit with remote HEAD (or specified ref)
   - Show diff summary to user

3. **Backup Current Pack**
   - Create backup in `.sir/packs/.backup/<pack-id>-<timestamp>/`
   - Allow rollback if update fails

4. **Pull Updates**
   - `git pull` if already cloned
   - OR re-clone with new ref

5. **Update Lock File**
   - Update commit hash and timestamp
   - Preserve installation metadata

6. **Validate Pack**
   - Verify `sir-pack.json` exists and is valid
   - Check method definitions are loadable
   - Run any pack-provided validation scripts

### Lock File Format Enhancement

```json
{
  "packs": {
    "pack-name": {
      "url": "https://github.com/owner/repo.git",
      "ref": "main",
      "commit": "abc123def456",  // Add: current commit hash
      "type": "github",
      "metadata": { ... },
      "installed_at": "2026-01-12T10:00:00Z",
      "updated_at": "2026-01-12T15:00:00Z"  // Add: last update time
    }
  }
}
```

### Update Strategies

#### 1. In-Place Update (Default)
```php
// Navigate to pack directory
cd ~/.sir/packs/pack-name

// Pull latest changes
git fetch origin
git reset --hard origin/main  // or specified ref
```

#### 2. Clean Reinstall
```php
// Remove old pack
rm -rf ~/.sir/packs/pack-name

// Clone fresh
git clone --depth 1 --branch main https://...
```

#### 3. Rollback Support
```php
// If update fails, restore from backup
cp -r ~/.sir/packs/.backup/pack-name-timestamp ~/.sir/packs/pack-name
```

## User Experience

### Interactive Update

```bash
$ sir recipes:update

Checking for updates...

┌─────────────┬─────────┬───────────┬──────────┐
│ Pack        │ Current │ Available │ Status   │
├─────────────┼─────────┼───────────┼──────────┤
│ sir-git     │ v1.0.0  │ v1.2.0    │ ⬆ Update │
│ sir-tasks   │ v2.1.0  │ v2.1.0    │ ✓ Latest │
└─────────────┴─────────┴───────────┴──────────┘

Update sir-git to v1.2.0? [yes/no]: yes

Updating sir-git...
  ✓ Backed up current version
  ✓ Pulled latest changes
  ✓ Validated pack structure
  ✓ Updated lock file

✓ Successfully updated 1 pack
```

### Automated Update

```bash
$ sir recipes:update --all --yes

Updating all packs...
  ✓ sir-git: v1.0.0 → v1.2.0
  ✓ sir-tasks: Already up to date
  ⨯ sir-custom: Update failed (rolled back)

✓ Updated 1 pack, 1 already up to date, 1 failed
```

## Error Handling

### Network Errors
```
✗ Failed to fetch updates for sir-git
  Network error: Could not connect to github.com
  The pack remains at version v1.0.0
```

### Validation Errors
```
✗ Update failed for sir-git
  Pack validation failed: Missing required method definitions
  ✓ Rolled back to v1.0.0
```

### Conflict Resolution
```
⚠ Local changes detected in sir-git
  The pack directory has uncommitted changes.
  
  Options:
  1. Discard local changes and update
  2. Skip this pack
  3. Backup local changes and update
  
  Choose [1/2/3]:
```

## Implementation Checklist

- [ ] Add commit hash tracking to lock file
- [ ] Implement git fetch/pull logic in PackManager
- [ ] Add version comparison utilities
- [ ] Create backup/rollback mechanism
- [ ] Update RecipesUpdateCommand with full implementation
- [ ] Add progress indicators for long operations
- [ ] Write tests for update scenarios
- [ ] Handle edge cases (conflicts, failures, etc.)
- [ ] Add `--dry-run` flag to preview updates
- [ ] Document update process in user guide

## Dependencies

- Existing `PackManager` class
- Git CLI
- Lock file read/write functionality
- Progress output utilities

## Testing Strategy

### Unit Tests
- Version comparison logic
- Lock file updates
- Backup/restore operations

### Integration Tests
- Full update cycle
- Rollback on failure
- Multiple pack updates
- Network error handling

### Manual Testing
- Test with real packs from GitHub
- Test with different ref types (tags, branches, commits)
- Test update conflicts
- Test rollback scenarios

## Future Considerations

- **Auto-update**: Scheduled automatic updates
- **Version Constraints**: Specify allowed version ranges
- **Change Notifications**: Show changelog when updating
- **Dependency Updates**: Update pack dependencies
- **Semantic Versioning**: Smart version comparison
- **Update Channels**: Stable, beta, dev channels
