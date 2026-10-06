# Defense-in-Depth Validation

## Overview

When you fix a bug caused by invalid data, adding validation at one place feels sufficient. But that single check can be bypassed by different code paths, refactoring, or mocks.

**Core principle:** Add validation at boundaries that can be bypassed through demonstrated alternate paths. Guard the operation that enforces the invariant; avoid redundant checks without a concrete failure mode.

## Why Multiple Layers

Single validation: "We fixed the bug"
Multiple justified layers: "The tested bypass paths are blocked"

Different layers catch different cases:
- Entry validation rejects malformed input
- Business logic catches edge cases
- Environment guards prevent context-specific dangers
- Debug logging helps when other layers fail

## The Four Layers

### Layer 1: Entry Point Validation
**Purpose:** Reject obviously invalid input at API boundary

```typescript
function createProject(name: string, workingDirectory: string) {
  if (!workingDirectory || workingDirectory.trim() === '') {
    throw new Error('workingDirectory cannot be empty');
  }
  if (!existsSync(workingDirectory)) {
    throw new Error('workingDirectory does not exist');
  }
  if (!statSync(workingDirectory).isDirectory()) {
    throw new Error('workingDirectory is not a directory');
  }
  // ... proceed
}
```

### Layer 2: Business Logic Validation
**Purpose:** Ensure data makes sense for this operation

```typescript
function initializeWorkspace(projectDir: string, sessionId: string) {
  if (!projectDir) {
    throw new Error('projectDir required for workspace initialization');
  }
  // ... proceed
}
```

### Layer 3: Environment Guards
**Purpose:** Prevent dangerous operations in specific contexts

```typescript
import * as path from 'node:path';
import { realpathSync } from 'node:fs';
import { tmpdir } from 'node:os';

async function gitInit(directory: string) {
  if (process.env.NODE_ENV === 'test') {
    if (!directory || !directory.trim()) {
      throw new Error('Test working directory is required');
    }
    // Existing directories only; resolve symlinks before checking containment.
    const normalized = realpathSync(directory);
    const tmpDir = realpathSync(tmpdir());
    const relative = path.relative(tmpDir, normalized);
    if (relative === '..' || relative.startsWith(`..${path.sep}`) || path.isAbsolute(relative)) {
      throw new Error('Refusing operation outside the test temporary directory');
    }
  }
  // ... proceed; shared/untrusted writable paths need protection from path races.
}
```

### Layer 4: Debug Instrumentation
**Purpose:** Capture context for forensics

```typescript
async function gitInit(directory: string) {
  logger.debug('About to git init', {
    directoryPresent: Boolean(directory),
    // Add only allowlisted metadata; redact paths/traces before sharing.
  });
  // ... proceed
}
```

## Applying the Pattern

When you find a bug:

1. **Trace the data flow** - Where does bad value originate? Where used?
2. **Map all checkpoints** - List every point data passes through
3. **Choose guards** - Select layers with a concrete bypass or invariant to enforce
4. **Test relevant paths** - Verify those bypasses in an isolated test environment; diagnostic logging is observation, not a validation guard

## Example from Session

Bug: Empty `projectDir` caused `git init` in source code

**Data flow:**
1. Test setup → empty string
2. `Project.create(name, '')`
3. `WorkspaceManager.createWorkspace('')`
4. `git init` runs in `process.cwd()`

**Four layers added:**
- Layer 1: `Project.create()` validates not empty/exists/writable
- Layer 2: `WorkspaceManager` validates projectDir not empty
- Layer 3: `WorktreeManager` refuses git init outside tmpdir in tests
- Layer 4: Stack trace logging before git init

**Historical reported result:** 1847 tests passed in the original session; this is not a universal guarantee or a current reproduction.

## Key Insight

The original session reported reasons for four layers; evaluate those reasons for the current system. During testing, each layer caught bugs the others missed:
- Different code paths bypassed entry validation
- Mocks bypassed business logic checks
- Edge cases on different platforms needed environment guards
- Debug logging identified structural misuse

**Choose sufficient validation points.** Additional guards require a bypass case; do not expand a bounded fix into unsolicited hardening.
