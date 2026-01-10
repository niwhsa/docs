# Disk & Storage Troubleshooting Cheatsheet

Commands for diagnosing and cleaning up disk space on macOS.

## Quick Checks

```bash
# Overall disk usage (your data volume)
df -h ~

# All volumes
df -h

# Physical disks and volume structure
diskutil list
```

## Find What's Using Space

### Top-level folders
```bash
# Largest folders in home
du -sh ~/*/ 2>/dev/null | sort -hr | head -15

# Include hidden folders
du -sh ~/.*/ 2>/dev/null | sort -hr | head -10

# Current directory
du -sh */ | sort -hr | head -10
```

### Recursive view (>100MB folders expand one level)
```bash
du -sh */ 2>/dev/null | sort -hr | while read size dir; do
  date=$(stat -f "%Sm" -t "%Y-%m-%d" "$dir")
  printf "%-8s %s  %s\n" "$size" "$date" "$dir"
  
  # If > 100M, show one level deeper
  size_mb=$(du -sm "$dir" 2>/dev/null | cut -f1)
  if [ "$size_mb" -gt 100 ]; then
    du -sh "$dir"*/ "$dir".* 2>/dev/null | sort -hr | head -5 | while read subsize subdir; do
      subdate=$(stat -f "%Sm" -t "%Y-%m-%d" "$subdir" 2>/dev/null)
      printf "  └─ %-8s %s  %s\n" "$subsize" "$subdate" "$subdir"
    done
  fi
done
```

### Find large files
```bash
# Files > 100MB
find ~ -type f -size +100M 2>/dev/null

# Files > 100MB with details
find ~ -type f -size +100M 2>/dev/null -exec ls -lh {} \;

# Files > 1GB
find ~ -type f -size +1G 2>/dev/null -exec ls -lh {} \;
```

## Safe Cleanup Commands

### Development caches
```bash
# Homebrew (old versions, cache)
brew cleanup -s
brew autoremove

# npm cache
npm cache clean --force

# Cargo registry cache (Rust)
rm -rf ~/.cargo/registry/cache/*

# Rust target directories (rebuilds automatically)
find ~ -name "target" -type d -path "*/rust/*" -exec rm -rf {} + 2>/dev/null

# Python virtual environments (if not needed)
find ~ -name ".venv" -type d -exec rm -rf {} + 2>/dev/null
find ~ -name "venv" -type d -exec rm -rf {} + 2>/dev/null

# node_modules (can reinstall with npm install)
find ~ -name "node_modules" -type d -prune -exec rm -rf {} + 2>/dev/null
```

### Browser caches
```bash
rm -rf ~/Library/Caches/Google/*
rm -rf ~/Library/Caches/Firefox/*
rm -rf ~/Library/Caches/BraveSoftware/*
rm -rf ~/Library/Caches/Arc/*
rm -rf ~/Library/Caches/com.operasoftware.Opera/*
```

### Xcode (if installed)
```bash
rm -rf ~/Library/Developer/Xcode/DerivedData/*
```

### Docker
```bash
docker system prune -a
```

### Terraform provider cache
```bash
rm -rf ~/.terraform.d/plugin-cache/*
```

## Quick One-liner Cleanup
```bash
brew cleanup -s && npm cache clean --force 2>/dev/null && rm -rf ~/.cargo/registry/cache/* && echo "Done" && df -h ~
```

## macOS Storage Management UI
```bash
open /System/Applications/Utilities/Storage\ Management.app
```
Or: **Apple Menu → About This Mac → Storage → Manage**

## Understanding macOS APFS Volumes

macOS uses APFS with multiple volumes sharing one container:

```
┌─────────────────────────────────────────────┐
│         APFS Container (shared space)        │
│  ┌────────┐ ┌────────┐ ┌────────┐           │
│  │ System │ │  Data  │ │   VM   │   ...     │
│  │  10GB  │ │ 190GB  │ │  11GB  │           │
│  └────────┘ └────────┘ └────────┘           │
│         ← FREE SPACE (shared) →              │
└─────────────────────────────────────────────┘
```

| Volume | Purpose |
|--------|---------|
| System | macOS (read-only) |
| Data | Your home directory, apps |
| VM | Swap/sleep image |
| Preboot | Boot files |

**Key:** All volumes share free space dynamically.

## Commands Reference

| Command | Purpose |
|---------|---------|
| `df -h ~` | Check your data volume space |
| `df -h` | All volumes |
| `diskutil list` | Physical disks & volumes |
| `du -sh */` | Folder sizes (current dir) |
| `du -sh */ \| sort -hr` | Sorted by size |
| `find ~ -size +100M` | Large files |
| `brew cleanup -s` | Clean Homebrew |
| `npm cache clean --force` | Clean npm |

