---
name: file-operation-safety
description: |
  CRITICAL SAFETY SKILL - Use this skill before any file system operation that deletes, moves, or overwrites
  files. Activate when the user wants to rm, mv, cp (with overwrite), clear directories, or perform bulk
  file operations. This skill prevents accidental file loss through verification and confirmation protocols.
---

# File Operation Safety

**CRITICAL: File deletions can be permanent. This skill prevents irreversible file loss.**

## Danger Levels

| Level | Operations | Requires |
|-------|------------|----------|
| ⛔ CRITICAL | `rm -rf`, recursive delete on directories | Full path verification + explicit confirmation |
| 🔴 HIGH | `rm` multiple files, `mv` with overwrite | List all files + confirmation |
| 🟠 MEDIUM | `rm` single file, `cp` with overwrite | Verify file exists + confirm |
| 🟢 LOW | `cp` to new location, `mkdir`, `touch` | Standard execution |

## Pre-Operation Protocol

### Before ANY Deletion

**Step 1: Verify the path**

```bash
# ALWAYS expand and verify the path first
realpath /path/to/target
ls -la /path/to/target

# Check what's inside (for directories)
find /path/to/target -type f | wc -l  # Count files
du -sh /path/to/target                 # Check size
```

**Step 2: Present impact to user**

```markdown
## File Deletion Impact Assessment

**Command:** rm -rf ./node_modules
**Resolved path:** /Users/dev/myproject/node_modules

**Contents:**
- Total files: 45,234
- Total directories: 8,901
- Total size: 234 MB
- Contains hidden files: Yes (.cache, .bin)

**Can be recreated:** Yes (via `npm install`)

Proceed with deletion? Type "yes, delete node_modules" to confirm.
```

### For Critical Operations

#### rm -rf on Directory

```markdown
⛔ **CRITICAL: RECURSIVE DELETE REQUESTED**

**Command:** rm -rf /var/www/myapp

**⚠️ FULL PATH CHECK:**
- Requested path: /var/www/myapp
- Resolved path: /var/www/myapp
- Parent directory: /var/www (contains 3 other directories)
- Is symlink: No

**Contents to be deleted:**
- Files: 1,234
- Directories: 89
- Size: 456 MB
- Includes:
  - Source code (*.js, *.ts): 234 files
  - Configuration (*.json, *.yml): 12 files
  - Environment files (.env): 1 file ⚠️
  - Uploaded files (uploads/): 567 files ⚠️

**Recovery:**
- Git repository: Yes (can recover source code)
- Uploads folder: NO GIT ⚠️ (not recoverable)
- .env file: NO GIT ⚠️ (not recoverable)

**Before deletion, I recommend:**
1. Backup uploads folder: `cp -r uploads /backup/uploads_$(date +%Y%m%d)`
2. Backup .env: `cp .env /backup/.env_$(date +%Y%m%d)`

Proceed with deletion after backup? Type "yes, delete /var/www/myapp after backup" to confirm.
```

#### rm on Home Directory or Root

```markdown
⛔⛔⛔ **EXTREME DANGER: CATASTROPHIC PATH DETECTED**

**Command:** rm -rf ~/ (or /, /home, /usr, /etc, /var)

**🚫 I WILL NOT EXECUTE THIS COMMAND.**

This would delete:
- Your entire home directory
- All personal files, documents, and configurations
- SSH keys, credentials, and secrets
- Potentially your entire system

**This is almost certainly a mistake.**

If you need to clean up:
- Specify exact subdirectories to delete
- Use `rm -rf ~/specific/directory` instead
- Never use wildcards at root level

**I will not proceed with this request under any circumstances.**
```

#### rm with Wildcards

```markdown
🔴 **DANGEROUS: WILDCARD DELETION**

**Command:** rm -rf *.log
**Working directory:** /var/log/myapp

**Files matching pattern:**
1. access.log (234 MB)
2. error.log (45 MB)
3. debug.log (12 MB)
4. app.2024-01-15.log (5 MB)
5. app.2024-01-14.log (4 MB)
... and 23 more files

**Total files:** 28
**Total size:** 345 MB

**Potential issues:**
- Pattern `*.log` is broad - verify these are all logs you want to delete
- Some may be needed for debugging recent issues
- Rotating logs may have important history

**Safer alternatives:**
```bash
# Delete only old logs
find . -name "*.log" -mtime +30 -delete

# Keep recent logs
rm -f *.log.2023*
```

Proceed with deleting all 28 .log files? Type "yes, delete 28 log files" to confirm.
```

### Moving Files (Overwrite Risk)

```markdown
## File Move Impact Assessment

**Command:** mv ./config.json ./backup/config.json

**⚠️ OVERWRITE WARNING:**
A file already exists at the destination!

**Source file:**
- Path: ./config.json
- Size: 2.3 KB
- Modified: 2024-01-16 14:30:00

**Destination file (WILL BE OVERWRITTEN):**
- Path: ./backup/config.json
- Size: 1.8 KB
- Modified: 2024-01-15 09:00:00

**Differences detected:** Yes (source is newer and larger)

**Options:**
1. Overwrite (lose old backup)
2. Rename destination first: `mv ./backup/config.json ./backup/config.json.old`
3. Use versioned name: `mv ./config.json ./backup/config.$(date +%Y%m%d).json`
4. Cancel

Choose an option or type "yes, overwrite backup/config.json" to proceed with overwrite.
```

### Copying with Overwrite

```markdown
## File Copy Impact Assessment

**Command:** cp -r ./src ./backup/src

**⚠️ DIRECTORY OVERWRITE WARNING:**
The destination directory already exists!

**Source directory:**
- Path: ./src
- Files: 234
- Size: 12 MB

**Destination directory (contents may be OVERWRITTEN):**
- Path: ./backup/src
- Files: 198
- Size: 10 MB
- Modified: 2024-01-14 16:00:00

**What will happen:**
- 198 existing files may be overwritten with newer versions
- 36 new files will be added
- 0 files in destination will be deleted (cp doesn't delete)

**Safer alternatives:**
```bash
# Create versioned backup
cp -r ./src ./backup/src_$(date +%Y%m%d_%H%M%S)

# Use rsync for smarter sync
rsync -av --backup --suffix=.bak ./src/ ./backup/src/
```

Choose approach or type "yes, overwrite backup/src" to proceed.
```

## Safe Patterns

### Safe Deletion with Confirmation

```bash
# Interactive deletion (asks for each file)
rm -i file1 file2 file3

# Show what would be deleted (dry run)
find /path -name "*.tmp" -print  # Preview
find /path -name "*.tmp" -delete # Execute after verification

# Move to trash instead of permanent delete
# macOS
mv file ~/.Trash/

# Linux (with trash-cli)
trash-put file
```

### Safe Directory Cleanup

```bash
# Never use: rm -rf *
# Instead, be explicit:
rm -rf ./node_modules
rm -rf ./dist
rm -rf ./.cache

# Or use a list:
DIRS_TO_CLEAN="node_modules dist .cache"
for dir in $DIRS_TO_CLEAN; do
  echo "Deleting: $dir"
  rm -rf "./$dir"
done
```

### Backup Before Risky Operations

```bash
# Create timestamped backup
BACKUP_DIR="./backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"
cp -r ./important_folder "$BACKUP_DIR/"

# Then proceed with risky operation
rm -rf ./important_folder
```

## Protection Mechanisms

### Alias Safeguards (Add to .bashrc/.zshrc)

```bash
# Make rm interactive by default
alias rm='rm -i'

# Require verbose output for dangerous operations
alias rm='rm -iv'
alias mv='mv -iv'
alias cp='cp -iv'

# Prevent recursive operations on root
alias rm='rm --preserve-root'

# Add confirmation for rf
rmrf() {
  echo "⚠️ About to delete: $@"
  echo "Files: $(find "$@" -type f 2>/dev/null | wc -l)"
  read -p "Are you sure? (yes/no) " confirm
  if [ "$confirm" = "yes" ]; then
    rm -rf "$@"
  else
    echo "Aborted"
  fi
}
```

### Safe Delete Function

```bash
# Safe delete with trash support
safe_delete() {
  local target="$1"
  local trash_dir="${HOME}/.local/share/Trash/files"

  if [ ! -e "$target" ]; then
    echo "Error: $target does not exist"
    return 1
  fi

  # Show what will be deleted
  echo "Target: $(realpath "$target")"
  if [ -d "$target" ]; then
    echo "Type: Directory"
    echo "Files: $(find "$target" -type f | wc -l)"
    echo "Size: $(du -sh "$target" | cut -f1)"
  else
    echo "Type: File"
    echo "Size: $(ls -lh "$target" | awk '{print $5}')"
  fi

  echo ""
  read -p "Move to trash? (yes/no) " confirm

  if [ "$confirm" = "yes" ]; then
    mkdir -p "$trash_dir"
    mv "$target" "$trash_dir/$(basename "$target").$(date +%s)"
    echo "Moved to trash. Recover from: $trash_dir"
  else
    echo "Aborted"
  fi
}
```

## Common Dangerous Patterns to Block

| Pattern | Risk | Safe Alternative |
|---------|------|------------------|
| `rm -rf /` | System destruction | Never execute |
| `rm -rf ~` | Home directory loss | Be specific: `rm -rf ~/Downloads/temp` |
| `rm -rf *` | Current directory wipe | List files explicitly |
| `rm -rf .` | Current directory wipe | `rm -rf ./specific-folder` |
| `rm -rf ..` | Parent directory wipe | Never use |
| `mv * /dev/null` | Data loss | Use specific file names |
| `> important_file` | File content wipe | Check file existence first |

## Recovery Information

### If Files Were Accidentally Deleted

```bash
# Check trash first (macOS/Linux with trash-cli)
ls ~/.Trash/           # macOS
ls ~/.local/share/Trash/files/  # Linux

# For recent deletions, check backup
ls ~/Dropbox/  # If using cloud backup
ls ~/.snapshots/  # If using Time Machine or snapshots

# For Git-tracked files
git checkout -- deleted_file
git restore deleted_file

# For untracked files in Git directory
# Unfortunately, mostly unrecoverable
# Prevention is the only cure
```

## Best Practices Summary

| DO | DON'T |
|----|-------|
| Verify path with `realpath` first | Trust relative paths |
| Preview with `ls` before `rm` | Use wildcards blindly |
| Use `-i` flag for interactive | Auto-confirm everything |
| Backup before bulk operations | Assume recovery is possible |
| Move to trash instead of delete | Use `rm -rf` casually |
| Use explicit file lists | Use `*` in dangerous commands |
| Double-check working directory | Run commands from wrong location |

## Pre-Flight Checklist

Before any deletion:
- [ ] Path is verified with `realpath` or `pwd`
- [ ] Not in home directory, root, or system paths
- [ ] Contents previewed with `ls` or `find`
- [ ] Important files backed up
- [ ] Wildcard patterns tested with `echo` first
- [ ] Working directory confirmed
- [ ] Not running as root (unless absolutely necessary)
