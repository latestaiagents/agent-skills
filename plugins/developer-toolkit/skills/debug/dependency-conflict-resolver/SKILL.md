---
name: dependency-conflict-resolver
description: |
  Use this skill when resolving dependency conflicts. Activate when the user has package version conflicts,
  npm/yarn/pnpm install failures, peer dependency warnings, duplicate packages, or module resolution errors.
---

# Dependency Conflict Resolver

Untangle and resolve package dependency conflicts.

## When to Use

- Package installation fails
- Peer dependency warnings
- "Module not found" errors
- Duplicate package warnings
- Version mismatch errors
- Lock file conflicts

## Common Error Types

### 1. Peer Dependency Conflicts

```
npm WARN ERESOLVE unable to resolve dependency tree
npm WARN peer react@"^17.0.0" from package-a@1.0.0
npm WARN   peer react@"^18.0.0" from package-b@2.0.0
```

**Cause:** Two packages require different versions of the same peer dependency.

### 2. Version Conflicts

```
npm ERR! Could not resolve dependency:
npm ERR! peer typescript@">=4.0 <5.0" from some-package@1.0.0
npm ERR! node_modules/some-package
npm ERR!   some-package@"^1.0.0" from the root project
```

**Cause:** Package requires a version range you're not using.

### 3. Duplicate Packages

```
warning "package-a" has unmet peer dependency "lodash@^4.0.0".
warning "package-b" has incorrect peer dependency "lodash@^3.0.0".
```

**Cause:** Multiple versions of the same package installed.

## Diagnostic Commands

### NPM

```bash
# See dependency tree
npm ls

# Find specific package
npm ls react

# Find why package is installed
npm why lodash

# Check for outdated packages
npm outdated

# Audit for issues
npm audit
```

### Yarn

```bash
# See dependency tree
yarn list

# Find specific package
yarn why react

# Check for duplicates
yarn dedupe --check

# Interactive upgrade
yarn upgrade-interactive
```

### PNPM

```bash
# See dependency tree
pnpm ls

# Find specific package
pnpm why react

# List all versions of a package
pnpm ls --depth Infinity | grep lodash
```

## Resolution Strategies

### Strategy 1: Update to Compatible Versions

```bash
# Find what versions are needed
npm ls react --all

# Update packages that need newer version
npm update package-a package-b

# Or install specific compatible version
npm install react@18.2.0
```

### Strategy 2: Override/Resolution

**NPM (package.json):**
```json
{
  "overrides": {
    "react": "18.2.0",
    "package-a": {
      "lodash": "4.17.21"
    }
  }
}
```

**Yarn (package.json):**
```json
{
  "resolutions": {
    "react": "18.2.0",
    "**/lodash": "4.17.21"
  }
}
```

**PNPM (package.json):**
```json
{
  "pnpm": {
    "overrides": {
      "react": "18.2.0",
      "lodash@^3.0.0": "4.17.21"
    }
  }
}
```

### Strategy 3: Force Install (Last Resort)

```bash
# NPM - force through conflicts
npm install --legacy-peer-deps

# Or with force flag
npm install --force

# Yarn
yarn install --ignore-peer-deps
```

**Warning:** This can lead to runtime errors. Use only when you've verified compatibility.

### Strategy 4: Deduplicate

```bash
# NPM
npm dedupe

# Yarn
yarn dedupe

# PNPM (automatic, but can force)
pnpm install
```

## Debugging Deep Conflicts

### Trace the Dependency Chain

```bash
# NPM - full tree
npm ls --all > deps.txt

# Find the conflict path
grep -B5 "lodash@3" deps.txt
```

**Example output:**
```
├─┬ package-a@1.0.0
│ └── lodash@4.17.21
├─┬ package-b@2.0.0
│ └─┬ old-dependency@0.5.0
│   └── lodash@3.10.1  <- Conflict source
```

### Check Package Requirements

```bash
# View package.json of installed package
cat node_modules/package-name/package.json | jq '.peerDependencies'

# Or use npm view
npm view package-name peerDependencies
```

## Common Scenarios

### Scenario 1: React Version Mismatch

```
npm WARN react-dom@18.2.0 requires react@^18.2.0, but react@17.0.2 is installed
```

**Solution:**
```bash
# Upgrade React
npm install react@18 react-dom@18

# Or if you need React 17
npm install react-dom@17
```

### Scenario 2: TypeScript Version

```
error TS2307: Cannot find module '@types/node' or its corresponding type declarations.
```

**Solution:**
```bash
# Check TypeScript version
npx tsc --version

# Install compatible types
npm install -D @types/node@ts$(npx tsc --version | cut -d' ' -f2 | cut -d'.' -f1-2)

# Or specific version
npm install -D @types/node@20
```

### Scenario 3: Native Module Rebuild

```
npm ERR! node-pre-gyp ERR! build error
```

**Solution:**
```bash
# Rebuild native modules
npm rebuild

# Or for specific package
npm rebuild bcrypt

# Clear cache and reinstall
rm -rf node_modules package-lock.json
npm install
```

### Scenario 4: Workspace Conflicts (Monorepo)

```
error Workspaces can only be enabled in private projects.
```

**Solution in package.json:**
```json
{
  "private": true,
  "workspaces": [
    "packages/*"
  ]
}
```

## Lock File Conflicts

### Git Merge Conflicts in Lock Files

```bash
# Don't manually resolve - regenerate
git checkout --theirs package-lock.json
npm install

# Or
rm package-lock.json
npm install
```

### Lock File Out of Sync

```bash
# NPM
rm package-lock.json
npm install

# Yarn
rm yarn.lock
yarn install

# PNPM
rm pnpm-lock.yaml
pnpm install
```

## Prevention

### Pin Versions

```json
{
  "dependencies": {
    "lodash": "4.17.21",     // Exact version
    "react": "^18.2.0",      // Minor updates OK
    "express": "~4.18.0"     // Patch updates only
  }
}
```

### Use Lock Files

```bash
# Always commit lock files
git add package-lock.json  # or yarn.lock, pnpm-lock.yaml

# CI should use ci command
npm ci  # instead of npm install
```

### Regular Updates

```bash
# Check for updates regularly
npm outdated

# Update incrementally
npm update

# Use tools like renovate/dependabot
```

## AI-Assisted Resolution

```markdown
Help me resolve this dependency conflict:

**Error message:**
```
[paste full error]
```

**Current package.json dependencies:**
```json
[paste dependencies]
```

**What I've tried:**
- [list attempts]

Please:
1. Explain what's conflicting
2. Show the dependency chain causing it
3. Suggest the safest resolution
4. Note any potential runtime issues
```

## Best Practices

1. **Read the full error** - It usually tells you what's wrong
2. **Check compatibility** - Before installing new packages
3. **Use lock files** - For reproducible installs
4. **Update regularly** - Small updates are easier than big jumps
5. **Test after resolving** - Conflicts can cause runtime issues
6. **Document workarounds** - Future you will thank you
7. **Avoid force** - Unless you understand the consequences
