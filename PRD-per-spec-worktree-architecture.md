# PRD: Per-Spec Worktree Architecture

## Executive Summary

Redesign the worktree system to support **one worktree per spec**, enabling:
- Multiple specs can be worked on simultaneously
- Each spec has its own isolated branch that persists until merge
- Clear mapping: spec → worktree → branch
- UI can show all pending branches ready for review/merge

## Current Problems

### Bug #1: Worktree Path Mismatch
- **Python:** Creates `.worktrees/auto-claude/`
- **UI:** Looks for `.worktrees/auto-claude-staging/`
- **Impact:** UI never finds worktrees

### Bug #2: Spec Name Ignored
- `get_existing_build_worktree(spec_name)` ignores the spec_name parameter
- All specs share ONE worktree
- **Impact:** Working on spec-003 would corrupt spec-002's work

### Bug #3: Single Worktree Design
- Only one worktree exists at a time
- Can only work on one spec at a time
- **Impact:** No parallel spec development

### Bug #4: Branches Deleted on Merge
- `merge_staging(delete_after=True)` deletes the branch
- **Impact:** Can't see which specs have work ready for review

### Bug #5: Process Hangs on Interactive Menu (Critical)
- `finalize_workspace()` displays interactive menu after build completes
- UI spawns process with **no TTY** - cannot receive keyboard input
- Process waits indefinitely for input that will never come
- **Impact:**
  - Process hangs forever → UI shows "Running" indefinitely
  - Status never transitions to `human_review`
  - Task stuck in `in_progress` state

**Evidence from terminal output:**
```
[1] ▶ Test the feature ([2] ✓ Add to my project now[3] 📄 Review what changed[4] ⏸ Decide later
```
Process is waiting for keyboard input that will never come.

---

## New Architecture

### Directory Structure

```
project/
├── .worktrees/
│   ├── 002-implement-memory/           # Worktree for spec 002
│   │   └── (full project copy)
│   ├── 003-fix-bug/                    # Worktree for spec 003
│   │   └── (full project copy)
│   └── 004-improve-ui/                 # Worktree for spec 004
│       └── (full project copy)
├── auto-claude/
│   └── specs/
│       ├── 002-implement-memory/
│       ├── 003-fix-bug/
│       └── 004-improve-ui/
└── (rest of project)
```

### Branch Naming Convention

```
auto-claude/{spec-id}

Examples:
- auto-claude/002-implement-memory
- auto-claude/003-fix-bug
- auto-claude/004-improve-ui
```

### Parallel Worker Branches (Temporary)

When running in parallel mode, workers create temporary branches:
```
worker-{id}/{chunk-id}

Examples:
- worker-1/chunk-001
- worker-2/chunk-002
```

These branch FROM the spec's worktree branch and merge back into it, then are deleted.

### Worktree-to-Spec Mapping

Each worktree directory name **matches** the spec folder name:
- Spec: `auto-claude/specs/002-implement-memory/`
- Worktree: `.worktrees/002-implement-memory/`
- Branch: `auto-claude/002-implement-memory`

This creates a **1:1:1 mapping** that's easy to reason about.

---

## Data Model Changes

### WorktreeInfo (Enhanced)

```python
@dataclass
class WorktreeInfo:
    """Information about a spec's worktree."""
    path: Path              # .worktrees/{spec-name}/
    branch: str             # auto-claude/{spec-name}
    spec_name: str          # The spec folder name (e.g., "002-implement-memory")
    base_branch: str        # Branch it was created from (e.g., "main")
    is_active: bool = True  # Whether worktree exists

    # Statistics (computed on demand)
    commit_count: int = 0
    files_changed: int = 0
    additions: int = 0
    deletions: int = 0
```

### WorktreeStatus (for UI)

```typescript
interface WorktreeStatus {
  exists: boolean;
  specId: string;           // Which spec this worktree is for
  worktreePath?: string;
  branch?: string;
  baseBranch?: string;
  commitCount?: number;
  filesChanged?: number;
  additions?: number;
  deletions?: number;
}
```

---

## Python Backend Changes

### 1. worktree.py - Complete Rewrite

```python
#!/usr/bin/env python3
"""
Git Worktree Manager - Per-Spec Architecture
=============================================

Each spec gets its own worktree:
- Worktree path: .worktrees/{spec-name}/
- Branch name: auto-claude/{spec-name}

This allows:
1. Multiple specs to be worked on simultaneously
2. Each spec's changes are isolated
3. Branches persist until explicitly merged
4. Clear 1:1:1 mapping: spec → worktree → branch
"""

import asyncio
import re
import shutil
import subprocess
from dataclasses import dataclass
from pathlib import Path
from typing import Optional


class WorktreeError(Exception):
    """Error during worktree operations."""
    pass


@dataclass
class WorktreeInfo:
    """Information about a spec's worktree."""
    path: Path
    branch: str
    spec_name: str
    base_branch: str
    is_active: bool = True
    commit_count: int = 0
    files_changed: int = 0
    additions: int = 0
    deletions: int = 0


class WorktreeManager:
    """
    Manages per-spec Git worktrees.

    Each spec gets its own worktree in .worktrees/{spec-name}/ with
    a corresponding branch auto-claude/{spec-name}.
    """

    def __init__(self, project_dir: Path, base_branch: Optional[str] = None):
        self.project_dir = project_dir
        self.base_branch = base_branch or self._get_current_branch()
        self.worktrees_dir = project_dir / ".worktrees"
        self._merge_lock = asyncio.Lock()

    def _get_current_branch(self) -> str:
        """Get the current git branch."""
        result = subprocess.run(
            ["git", "rev-parse", "--abbrev-ref", "HEAD"],
            cwd=self.project_dir,
            capture_output=True,
            text=True,
        )
        if result.returncode != 0:
            raise WorktreeError(f"Failed to get current branch: {result.stderr}")
        return result.stdout.strip()

    def _run_git(self, args: list[str], cwd: Optional[Path] = None) -> subprocess.CompletedProcess:
        """Run a git command and return the result."""
        return subprocess.run(
            ["git"] + args,
            cwd=cwd or self.project_dir,
            capture_output=True,
            text=True,
        )

    def setup(self) -> None:
        """Create worktrees directory if needed."""
        self.worktrees_dir.mkdir(exist_ok=True)

    # ==================== Per-Spec Worktree Methods ====================

    def get_worktree_path(self, spec_name: str) -> Path:
        """Get the worktree path for a spec."""
        return self.worktrees_dir / spec_name

    def get_branch_name(self, spec_name: str) -> str:
        """Get the branch name for a spec."""
        return f"auto-claude/{spec_name}"

    def worktree_exists(self, spec_name: str) -> bool:
        """Check if a worktree exists for a spec."""
        return self.get_worktree_path(spec_name).exists()

    def get_worktree_info(self, spec_name: str) -> Optional[WorktreeInfo]:
        """Get info about a spec's worktree."""
        worktree_path = self.get_worktree_path(spec_name)
        if not worktree_path.exists():
            return None

        # Verify the branch exists in the worktree
        result = self._run_git(["rev-parse", "--abbrev-ref", "HEAD"], cwd=worktree_path)
        if result.returncode != 0:
            return None

        actual_branch = result.stdout.strip()

        # Get statistics
        stats = self._get_worktree_stats(spec_name)

        return WorktreeInfo(
            path=worktree_path,
            branch=actual_branch,
            spec_name=spec_name,
            base_branch=self.base_branch,
            is_active=True,
            **stats
        )

    def _get_worktree_stats(self, spec_name: str) -> dict:
        """Get diff statistics for a worktree."""
        worktree_path = self.get_worktree_path(spec_name)

        stats = {
            "commit_count": 0,
            "files_changed": 0,
            "additions": 0,
            "deletions": 0,
        }

        if not worktree_path.exists():
            return stats

        # Commit count
        result = self._run_git(
            ["rev-list", "--count", f"{self.base_branch}..HEAD"],
            cwd=worktree_path
        )
        if result.returncode == 0:
            stats["commit_count"] = int(result.stdout.strip() or "0")

        # Diff stats
        result = self._run_git(
            ["diff", "--shortstat", f"{self.base_branch}...HEAD"],
            cwd=worktree_path
        )
        if result.returncode == 0 and result.stdout.strip():
            # Parse: "3 files changed, 50 insertions(+), 10 deletions(-)"
            match = re.search(r"(\d+) files? changed", result.stdout)
            if match:
                stats["files_changed"] = int(match.group(1))
            match = re.search(r"(\d+) insertions?", result.stdout)
            if match:
                stats["additions"] = int(match.group(1))
            match = re.search(r"(\d+) deletions?", result.stdout)
            if match:
                stats["deletions"] = int(match.group(1))

        return stats

    def create_worktree(self, spec_name: str) -> WorktreeInfo:
        """
        Create a worktree for a spec.

        Args:
            spec_name: The spec folder name (e.g., "002-implement-memory")

        Returns:
            WorktreeInfo for the created worktree
        """
        worktree_path = self.get_worktree_path(spec_name)
        branch_name = self.get_branch_name(spec_name)

        # Remove existing if present (from crashed previous run)
        if worktree_path.exists():
            self._run_git(["worktree", "remove", "--force", str(worktree_path)])

        # Delete branch if it exists (from previous attempt)
        self._run_git(["branch", "-D", branch_name])

        # Create worktree with new branch from base
        result = self._run_git([
            "worktree", "add", "-b", branch_name,
            str(worktree_path), self.base_branch
        ])

        if result.returncode != 0:
            raise WorktreeError(f"Failed to create worktree for {spec_name}: {result.stderr}")

        print(f"Created worktree: {worktree_path.name} on branch {branch_name}")

        return WorktreeInfo(
            path=worktree_path,
            branch=branch_name,
            spec_name=spec_name,
            base_branch=self.base_branch,
            is_active=True,
        )

    def get_or_create_worktree(self, spec_name: str) -> WorktreeInfo:
        """
        Get existing worktree or create a new one for a spec.

        Args:
            spec_name: The spec folder name

        Returns:
            WorktreeInfo for the worktree
        """
        existing = self.get_worktree_info(spec_name)
        if existing:
            print(f"Using existing worktree: {existing.path}")
            return existing

        return self.create_worktree(spec_name)

    def remove_worktree(self, spec_name: str, delete_branch: bool = False) -> None:
        """
        Remove a spec's worktree.

        Args:
            spec_name: The spec folder name
            delete_branch: Whether to also delete the branch
        """
        worktree_path = self.get_worktree_path(spec_name)
        branch_name = self.get_branch_name(spec_name)

        if worktree_path.exists():
            result = self._run_git(["worktree", "remove", "--force", str(worktree_path)])
            if result.returncode == 0:
                print(f"Removed worktree: {worktree_path.name}")
            else:
                print(f"Warning: Could not remove worktree: {result.stderr}")
                shutil.rmtree(worktree_path, ignore_errors=True)

        if delete_branch:
            self._run_git(["branch", "-D", branch_name])
            print(f"Deleted branch: {branch_name}")

        self._run_git(["worktree", "prune"])

    def merge_worktree(self, spec_name: str, delete_after: bool = False) -> bool:
        """
        Merge a spec's worktree branch back to base branch.

        Args:
            spec_name: The spec folder name
            delete_after: Whether to remove worktree and branch after merge

        Returns:
            True if merge succeeded
        """
        info = self.get_worktree_info(spec_name)
        if not info:
            print(f"No worktree found for spec: {spec_name}")
            return False

        print(f"Merging {info.branch} into {self.base_branch}...")

        # Switch to base branch in main project
        result = self._run_git(["checkout", self.base_branch])
        if result.returncode != 0:
            print(f"Error: Could not checkout base branch: {result.stderr}")
            return False

        # Merge the spec branch
        result = self._run_git([
            "merge", "--no-ff", info.branch,
            "-m", f"auto-claude: Merge {info.branch}"
        ])

        if result.returncode != 0:
            print(f"Merge conflict! Aborting merge...")
            self._run_git(["merge", "--abort"])
            return False

        print(f"Successfully merged {info.branch}")

        if delete_after:
            self.remove_worktree(spec_name, delete_branch=True)

        return True

    def commit_in_worktree(self, spec_name: str, message: str) -> bool:
        """Commit all changes in a spec's worktree."""
        worktree_path = self.get_worktree_path(spec_name)
        if not worktree_path.exists():
            return False

        self._run_git(["add", "."], cwd=worktree_path)
        result = self._run_git(["commit", "-m", message], cwd=worktree_path)

        if result.returncode == 0:
            return True
        elif "nothing to commit" in result.stdout + result.stderr:
            return True
        else:
            print(f"Commit failed: {result.stderr}")
            return False

    # ==================== Listing & Discovery ====================

    def list_all_worktrees(self) -> list[WorktreeInfo]:
        """List all spec worktrees."""
        worktrees = []

        if not self.worktrees_dir.exists():
            return worktrees

        for item in self.worktrees_dir.iterdir():
            if item.is_dir() and not item.name.startswith("worker-"):
                info = self.get_worktree_info(item.name)
                if info:
                    worktrees.append(info)

        return worktrees

    def list_all_spec_branches(self) -> list[str]:
        """List all auto-claude branches (even if worktree removed)."""
        result = self._run_git(["branch", "--list", "auto-claude/*"])
        if result.returncode != 0:
            return []

        branches = []
        for line in result.stdout.strip().split("\n"):
            branch = line.strip().lstrip("* ")
            if branch:
                branches.append(branch)

        return branches

    def get_changed_files(self, spec_name: str) -> list[tuple[str, str]]:
        """Get list of changed files in a spec's worktree."""
        worktree_path = self.get_worktree_path(spec_name)
        if not worktree_path.exists():
            return []

        result = self._run_git(
            ["diff", "--name-status", f"{self.base_branch}...HEAD"],
            cwd=worktree_path
        )

        files = []
        for line in result.stdout.strip().split("\n"):
            if not line:
                continue
            parts = line.split("\t", 1)
            if len(parts) == 2:
                files.append((parts[0], parts[1]))

        return files

    def get_change_summary(self, spec_name: str) -> dict:
        """Get a summary of changes in a worktree."""
        files = self.get_changed_files(spec_name)

        new_files = sum(1 for status, _ in files if status == "A")
        modified_files = sum(1 for status, _ in files if status == "M")
        deleted_files = sum(1 for status, _ in files if status == "D")

        return {
            "new_files": new_files,
            "modified_files": modified_files,
            "deleted_files": deleted_files,
        }

    def cleanup_all(self) -> None:
        """Remove all worktrees and their branches."""
        for worktree in self.list_all_worktrees():
            self.remove_worktree(worktree.spec_name, delete_branch=True)

    def cleanup_stale_worktrees(self) -> None:
        """Remove worktrees that aren't registered with git."""
        if not self.worktrees_dir.exists():
            return

        # Get list of registered worktrees
        result = self._run_git(["worktree", "list", "--porcelain"])
        registered_paths = set()
        for line in result.stdout.split("\n"):
            if line.startswith("worktree "):
                registered_paths.add(Path(line.split(" ", 1)[1]))

        # Remove unregistered directories
        for item in self.worktrees_dir.iterdir():
            if item.is_dir() and item not in registered_paths:
                print(f"Removing stale worktree directory: {item.name}")
                shutil.rmtree(item, ignore_errors=True)

        self._run_git(["worktree", "prune"])

    # ==================== Parallel Worker Support ====================

    def create_worker_worktree(self, spec_name: str, worker_id: str, chunk_id: str) -> tuple[Path, str]:
        """
        Create a temporary worker worktree that branches from a spec's worktree.

        Args:
            spec_name: The spec this worker is working on
            worker_id: Unique worker identifier
            chunk_id: The chunk being worked on

        Returns:
            Tuple of (worktree_path, branch_name)
        """
        spec_info = self.get_worktree_info(spec_name)
        if not spec_info:
            raise WorktreeError(f"Spec worktree does not exist: {spec_name}")

        worker_name = f"worker-{worker_id}"
        branch_name = f"worker-{worker_id}/{chunk_id}"
        worktree_path = self.worktrees_dir / worker_name

        # Clean up any existing worker worktree
        if worktree_path.exists():
            self._run_git(["worktree", "remove", "--force", str(worktree_path)])
        self._run_git(["branch", "-D", branch_name])

        # Create worker worktree branching FROM the spec's branch
        result = self._run_git([
            "worktree", "add", "-b", branch_name,
            str(worktree_path), spec_info.branch
        ])

        if result.returncode != 0:
            raise WorktreeError(f"Failed to create worker worktree: {result.stderr}")

        print(f"Created worker worktree: {worker_name} (from {spec_info.branch})")
        return worktree_path, branch_name

    def merge_worker_to_spec(self, spec_name: str, worker_branch: str) -> bool:
        """
        Merge a worker's branch back into the spec's worktree.

        Args:
            spec_name: The spec to merge into
            worker_branch: The worker branch to merge

        Returns:
            True if merge succeeded
        """
        spec_path = self.get_worktree_path(spec_name)
        if not spec_path.exists():
            print(f"Spec worktree does not exist: {spec_name}")
            return False

        print(f"Merging {worker_branch} into {spec_name}...")

        result = self._run_git(
            ["merge", "--no-ff", worker_branch, "-m", f"auto-claude: Merge {worker_branch}"],
            cwd=spec_path
        )

        if result.returncode != 0:
            print(f"Merge conflict! Aborting...")
            self._run_git(["merge", "--abort"], cwd=spec_path)
            return False

        print(f"Successfully merged {worker_branch}")
        return True

    def cleanup_worker_worktree(self, worker_id: str, branch_name: str) -> None:
        """Clean up a worker's temporary worktree and branch."""
        worker_name = f"worker-{worker_id}"
        worktree_path = self.worktrees_dir / worker_name

        if worktree_path.exists():
            self._run_git(["worktree", "remove", "--force", str(worktree_path)])

        self._run_git(["branch", "-D", branch_name])
        self._run_git(["worktree", "prune"])

    def cleanup_all_workers(self) -> None:
        """Remove all worker worktrees (preserves spec worktrees)."""
        if not self.worktrees_dir.exists():
            return

        for item in self.worktrees_dir.iterdir():
            if item.is_dir() and item.name.startswith("worker-"):
                self._run_git(["worktree", "remove", "--force", str(item)])

        # Clean up worker branches
        result = self._run_git(["branch", "--list", "worker-*/*"])
        if result.returncode == 0:
            for line in result.stdout.strip().split("\n"):
                branch = line.strip().lstrip("* ")
                if branch:
                    self._run_git(["branch", "-D", branch])

        self._run_git(["worktree", "prune"])

    def get_test_commands(self, spec_name: str) -> list[str]:
        """Detect likely test/run commands for the project."""
        worktree_path = self.get_worktree_path(spec_name)
        commands = []

        if (worktree_path / "package.json").exists():
            commands.append("npm install && npm run dev")
            commands.append("npm test")

        if (worktree_path / "requirements.txt").exists():
            commands.append("pip install -r requirements.txt")

        if (worktree_path / "Cargo.toml").exists():
            commands.append("cargo run")
            commands.append("cargo test")

        if (worktree_path / "go.mod").exists():
            commands.append("go run .")
            commands.append("go test ./...")

        if not commands:
            commands.append("# Check the project's README for run instructions")

        return commands
```

### 2. workspace.py - Updated Functions

```python
#!/usr/bin/env python3
"""
Workspace Selection and Management - Per-Spec Architecture
===========================================================

Each spec gets its own isolated worktree at .worktrees/{spec-name}/
"""

import shutil
import subprocess
import sys
from enum import Enum
from pathlib import Path
from typing import Optional

from worktree import WorktreeManager, WorktreeInfo
from ui import (
    Icons,
    icon,
    box,
    success,
    error,
    warning,
    info,
    muted,
    highlight,
    bold,
    print_header,
    print_status,
    print_key_value,
    select_menu,
    MenuOption,
)


class WorkspaceMode(Enum):
    """How auto-claude should work."""
    ISOLATED = "isolated"  # Work in a separate worktree (safe)
    DIRECT = "direct"      # Work directly in user's project


class WorkspaceChoice(Enum):
    """User's choice after build completes."""
    MERGE = "merge"        # Add changes to project
    REVIEW = "review"      # Show what changed
    TEST = "test"          # Test the feature in the worktree
    LATER = "later"        # Decide later


def has_uncommitted_changes(project_dir: Path) -> bool:
    """Check if user has unsaved work."""
    result = subprocess.run(
        ["git", "status", "--porcelain"],
        cwd=project_dir,
        capture_output=True,
        text=True,
    )
    return bool(result.stdout.strip())


def get_current_branch(project_dir: Path) -> str:
    """Get the current branch name."""
    result = subprocess.run(
        ["git", "rev-parse", "--abbrev-ref", "HEAD"],
        cwd=project_dir,
        capture_output=True,
        text=True,
    )
    return result.stdout.strip()


def get_existing_build_worktree(project_dir: Path, spec_name: str) -> Optional[Path]:
    """
    Check if there's an existing worktree FOR THIS SPECIFIC SPEC.

    Args:
        project_dir: The project directory
        spec_name: The spec folder name (e.g., "002-implement-memory")

    Returns:
        Path to worktree if it exists for this spec, None otherwise
    """
    worktree_path = project_dir / ".worktrees" / spec_name

    if worktree_path.exists():
        # Verify it's a valid git worktree
        git_dir = worktree_path / ".git"
        if git_dir.exists():
            return worktree_path

    return None


def copy_spec_to_worktree(
    source_spec_dir: Path,
    worktree_path: Path,
    spec_name: str,
) -> Path:
    """
    Copy spec files into the worktree so the AI can access them.

    Args:
        source_spec_dir: Original spec directory (may be outside worktree)
        worktree_path: Path to the worktree
        spec_name: Name of the spec folder

    Returns:
        Path to the spec directory inside the worktree
    """
    target_spec_dir = worktree_path / "auto-claude" / "specs" / spec_name

    target_spec_dir.parent.mkdir(parents=True, exist_ok=True)

    if target_spec_dir.exists():
        shutil.rmtree(target_spec_dir)

    shutil.copytree(source_spec_dir, target_spec_dir)

    return target_spec_dir


def setup_workspace(
    project_dir: Path,
    spec_name: str,
    mode: WorkspaceMode,
    source_spec_dir: Optional[Path] = None,
) -> tuple[Path, Optional[WorktreeManager], Optional[Path]]:
    """
    Set up the workspace for a specific spec.

    Uses per-spec worktrees - each spec gets its own isolated worktree.

    Args:
        project_dir: The project directory
        spec_name: Name of the spec being built
        mode: The workspace mode to use
        source_spec_dir: Optional source spec directory to copy to worktree

    Returns:
        Tuple of (working_directory, worktree_manager or None, localized_spec_dir or None)
    """
    if mode == WorkspaceMode.DIRECT:
        return project_dir, None, source_spec_dir

    print()
    print_status(f"Setting up workspace for {spec_name}...", "progress")

    manager = WorktreeManager(project_dir)
    manager.setup()

    # Get or create worktree FOR THIS SPECIFIC SPEC
    info = manager.get_or_create_worktree(spec_name)

    # Copy spec files to worktree if provided
    localized_spec_dir = None
    if source_spec_dir and source_spec_dir.exists():
        localized_spec_dir = copy_spec_to_worktree(
            source_spec_dir, info.path, spec_name
        )
        print_status(f"Spec files copied to workspace", "success")

    print_status(f"Workspace ready: {info.path.name}", "success")
    print()

    return info.path, manager, localized_spec_dir


def choose_workspace(
    project_dir: Path,
    spec_name: str,
    force_isolated: bool = False,
    force_direct: bool = False,
    auto_continue: bool = False,
) -> WorkspaceMode:
    """Let user choose where auto-claude should work."""
    if force_isolated:
        return WorkspaceMode.ISOLATED
    if force_direct:
        return WorkspaceMode.DIRECT
    if auto_continue:
        print("Auto-continue: Using isolated workspace for safety.")
        return WorkspaceMode.ISOLATED

    has_unsaved = has_uncommitted_changes(project_dir)

    if has_unsaved:
        content = [
            success(f"{icon(Icons.SHIELD)} YOUR WORK IS PROTECTED"),
            "",
            "You have unsaved work in your project.",
            "",
            "To keep your work safe, the AI will build in a",
            "separate workspace. Your current files won't be",
            "touched until you're ready.",
        ]
        print()
        print(box(content, width=60, style="heavy"))
        print()

        try:
            input(f"Press Enter to continue...")
        except KeyboardInterrupt:
            print()
            print_status("Cancelled.", "info")
            sys.exit(0)

        return WorkspaceMode.ISOLATED

    options = [
        MenuOption(
            key="isolated",
            label="Separate workspace (Recommended)",
            icon=Icons.SHIELD,
            description="Your current files stay untouched. Easy to review and undo.",
        ),
        MenuOption(
            key="direct",
            label="Right here in your project",
            icon=Icons.LIGHTNING,
            description="Changes happen directly. Best if you're not working on anything else.",
        ),
    ]

    choice = select_menu(
        title="Where should the AI build your feature?",
        options=options,
        allow_quit=True,
    )

    if choice is None:
        print()
        print_status("Cancelled.", "info")
        sys.exit(0)

    if choice == "direct":
        print()
        print_status("Working directly in your project.", "info")
        return WorkspaceMode.DIRECT
    else:
        print()
        print_status("Using a separate workspace for safety.", "success")
        return WorkspaceMode.ISOLATED


def show_build_summary(manager: WorktreeManager, spec_name: str) -> None:
    """Show a summary of what was built."""
    summary = manager.get_change_summary(spec_name)

    total = summary["new_files"] + summary["modified_files"] + summary["deleted_files"]

    if total == 0:
        print_status("No changes were made.", "info")
        return

    print()
    print(bold("What was built:"))
    if summary["new_files"] > 0:
        print(success(f"  + {summary['new_files']} new file{'s' if summary['new_files'] != 1 else ''}"))
    if summary["modified_files"] > 0:
        print(info(f"  ~ {summary['modified_files']} modified file{'s' if summary['modified_files'] != 1 else ''}"))
    if summary["deleted_files"] > 0:
        print(error(f"  - {summary['deleted_files']} deleted file{'s' if summary['deleted_files'] != 1 else ''}"))


def show_changed_files(manager: WorktreeManager, spec_name: str) -> None:
    """Show detailed list of changed files."""
    files = manager.get_changed_files(spec_name)

    if not files:
        print_status("No changes.", "info")
        return

    print()
    print(bold("Changed files:"))
    for status, filepath in files:
        if status == "A":
            print(success(f"  + {filepath}"))
        elif status == "M":
            print(info(f"  ~ {filepath}"))
        elif status == "D":
            print(error(f"  - {filepath}"))
        else:
            print(f"  {status} {filepath}")


def finalize_workspace(
    project_dir: Path,
    spec_name: str,
    manager: Optional[WorktreeManager],
    auto_continue: bool = False,  # NEW: Skip interactive menu when True
) -> WorkspaceChoice:
    """Handle post-build workflow.

    Args:
        project_dir: The project directory
        spec_name: Name of the spec
        manager: WorktreeManager instance (None if direct mode)
        auto_continue: If True, skip interactive menu (for UI/non-TTY mode)
    """
    if manager is None:
        content = [
            success(f"{icon(Icons.SUCCESS)} BUILD COMPLETE!"),
            "",
            "Changes were made directly to your project.",
            muted("Use 'git status' to see what changed."),
        ]
        print()
        print(box(content, width=60, style="heavy"))
        return WorkspaceChoice.MERGE

    content = [
        success(f"{icon(Icons.SUCCESS)} BUILD COMPLETE!"),
        "",
        "The AI built your feature in a separate workspace.",
    ]
    print()
    print(box(content, width=60, style="heavy"))

    show_build_summary(manager, spec_name)

    # CRITICAL: Skip interactive menu in non-TTY mode (UI spawned process)
    # This fixes Bug #5 - process hanging on interactive menu
    if auto_continue:
        worktree_path = manager.get_worktree_path(spec_name)
        print()
        print(muted("Build saved. Use the UI or CLI to merge/review/discard:"))
        print(muted(f"  --merge   : Add changes to your project"))
        print(muted(f"  --review  : See what was built"))
        print(muted(f"  --discard : Delete the build"))
        print()
        print(f"Worktree: {highlight(str(worktree_path))}")
        return WorkspaceChoice.LATER  # Let UI handle the decision

    worktree_path = manager.get_worktree_path(spec_name)

    options = [
        MenuOption(
            key="test",
            label="Test the feature (Recommended)",
            icon=Icons.PLAY,
            description="Run the app and try it out before adding to your project",
        ),
        MenuOption(
            key="merge",
            label="Add to my project now",
            icon=Icons.SUCCESS,
            description="Merge the changes into your files immediately",
        ),
        MenuOption(
            key="review",
            label="Review what changed",
            icon=Icons.FILE,
            description="See exactly what files were modified",
        ),
        MenuOption(
            key="later",
            label="Decide later",
            icon=Icons.PAUSE,
            description="Your build is saved - you can come back anytime",
        ),
    ]

    print()
    choice = select_menu(
        title="What would you like to do?",
        options=options,
        allow_quit=False,
    )

    if choice == "test":
        return WorkspaceChoice.TEST
    elif choice == "merge":
        return WorkspaceChoice.MERGE
    elif choice == "review":
        return WorkspaceChoice.REVIEW
    else:
        return WorkspaceChoice.LATER


def handle_workspace_choice(
    choice: WorkspaceChoice,
    project_dir: Path,
    spec_name: str,
    manager: WorktreeManager,
) -> None:
    """Execute the user's choice."""
    worktree_path = manager.get_worktree_path(spec_name)

    if choice == WorkspaceChoice.TEST:
        content = [
            bold(f"{icon(Icons.PLAY)} TEST YOUR FEATURE"),
            "",
            "Your feature is ready to test in a separate workspace.",
        ]
        print()
        print(box(content, width=60, style="heavy"))

        print()
        print("To test it, open a NEW terminal and run:")
        print()
        print(highlight(f"  cd {worktree_path}"))

        commands = manager.get_test_commands(spec_name)
        print()
        print("Then run your project:")
        for cmd in commands[:2]:
            print(f"  {cmd}")

        print()
        print(muted("-" * 60))
        print()
        print("When you're done testing:")
        print(highlight(f"  python auto-claude/run.py --spec {spec_name} --merge"))
        print()
        print("To discard (if you don't like it):")
        print(muted(f"  python auto-claude/run.py --spec {spec_name} --discard"))
        print()

    elif choice == WorkspaceChoice.MERGE:
        print()
        print_status("Adding changes to your project...", "progress")
        success_result = manager.merge_worktree(spec_name, delete_after=True)

        if success_result:
            print()
            print_status("Your feature has been added to your project.", "success")
        else:
            print()
            print_status("There was a conflict merging the changes.", "error")
            print(muted("Your build is still saved in the separate workspace."))
            print(muted("You may need to merge manually or ask for help."))

    elif choice == WorkspaceChoice.REVIEW:
        show_changed_files(manager, spec_name)
        print()
        print(muted("-" * 60))
        print()
        print("To see full details of changes:")
        info_obj = manager.get_worktree_info(spec_name)
        if info_obj:
            print(muted(f"  git diff {info_obj.base_branch}...{info_obj.branch}"))
        print()
        print("To test the feature:")
        print(highlight(f"  cd {worktree_path}"))
        print()
        print("To add these changes to your project:")
        print(highlight(f"  python auto-claude/run.py --spec {spec_name} --merge"))
        print()

    else:  # LATER
        print()
        print_status("No problem! Your build is saved.", "success")
        print()
        print("To test the feature:")
        print(highlight(f"  cd {worktree_path}"))
        print()
        print("When you're ready to add it:")
        print(highlight(f"  python auto-claude/run.py --spec {spec_name} --merge"))
        print()
        print("To see what was built:")
        print(muted(f"  python auto-claude/run.py --spec {spec_name} --review"))
        print()


def merge_existing_build(project_dir: Path, spec_name: str) -> bool:
    """Merge an existing build into the project."""
    worktree_path = get_existing_build_worktree(project_dir, spec_name)

    if not worktree_path:
        print()
        print_status(f"No existing build found for '{spec_name}'.", "warning")
        print()
        print("To start a new build:")
        print(highlight(f"  python auto-claude/run.py --spec {spec_name}"))
        return False

    content = [
        bold(f"{icon(Icons.SUCCESS)} ADDING BUILD TO YOUR PROJECT"),
    ]
    print()
    print(box(content, width=60, style="heavy"))

    manager = WorktreeManager(project_dir)

    show_build_summary(manager, spec_name)
    print()

    success_result = manager.merge_worktree(spec_name, delete_after=True)

    if success_result:
        print()
        print_status("Your feature has been added to your project.", "success")
        return True
    else:
        print()
        print_status("There was a conflict merging the changes.", "error")
        print(muted("You may need to merge manually."))
        return False


def review_existing_build(project_dir: Path, spec_name: str) -> bool:
    """Show what an existing build contains."""
    worktree_path = get_existing_build_worktree(project_dir, spec_name)

    if not worktree_path:
        print()
        print_status(f"No existing build found for '{spec_name}'.", "warning")
        print()
        print("To start a new build:")
        print(highlight(f"  python auto-claude/run.py --spec {spec_name}"))
        return False

    content = [
        bold(f"{icon(Icons.FILE)} BUILD CONTENTS"),
    ]
    print()
    print(box(content, width=60, style="heavy"))

    manager = WorktreeManager(project_dir)
    info_obj = manager.get_worktree_info(spec_name)

    show_build_summary(manager, spec_name)
    show_changed_files(manager, spec_name)

    print()
    print(muted("-" * 60))
    print()
    print("To test the feature:")
    print(highlight(f"  cd {worktree_path}"))
    print()
    print("To add these changes to your project:")
    print(highlight(f"  python auto-claude/run.py --spec {spec_name} --merge"))
    print()
    print("To see full diff:")
    if info_obj:
        print(muted(f"  git diff {info_obj.base_branch}...{info_obj.branch}"))
    print()

    return True


def discard_existing_build(project_dir: Path, spec_name: str) -> bool:
    """Discard an existing build (with confirmation)."""
    worktree_path = get_existing_build_worktree(project_dir, spec_name)

    if not worktree_path:
        print()
        print_status(f"No existing build found for '{spec_name}'.", "warning")
        return False

    content = [
        warning(f"{icon(Icons.WARNING)} DELETE BUILD RESULTS?"),
        "",
        "This will permanently delete all work for this build.",
    ]
    print()
    print(box(content, width=60, style="heavy"))

    manager = WorktreeManager(project_dir)

    show_build_summary(manager, spec_name)

    print()
    print(f"Are you sure? Type {highlight('delete')} to confirm: ", end="")

    try:
        confirmation = input().strip().lower()
    except KeyboardInterrupt:
        print()
        print_status("Cancelled. Your build is still saved.", "info")
        return False

    if confirmation != "delete":
        print()
        print_status("Cancelled. Your build is still saved.", "info")
        return False

    manager.remove_worktree(spec_name, delete_branch=True)

    print()
    print_status("Build deleted.", "success")
    return True


def check_existing_build(project_dir: Path, spec_name: str) -> bool:
    """Check if there's an existing build and offer options."""
    worktree_path = get_existing_build_worktree(project_dir, spec_name)

    if not worktree_path:
        return False

    content = [
        info(f"{icon(Icons.INFO)} EXISTING BUILD FOUND"),
        "",
        "There's already a build in progress for this spec.",
    ]
    print()
    print(box(content, width=60, style="heavy"))

    options = [
        MenuOption(
            key="continue",
            label="Continue where it left off",
            icon=Icons.PLAY,
            description="Resume building from the last checkpoint",
        ),
        MenuOption(
            key="review",
            label="Review what was built",
            icon=Icons.FILE,
            description="See the files that were created/modified",
        ),
        MenuOption(
            key="merge",
            label="Add to my project now",
            icon=Icons.SUCCESS,
            description="Merge the existing build into your project",
        ),
        MenuOption(
            key="fresh",
            label="Start fresh",
            icon=Icons.ERROR,
            description="Discard current build and start over",
        ),
    ]

    print()
    choice = select_menu(
        title="What would you like to do?",
        options=options,
        allow_quit=True,
    )

    if choice is None:
        print()
        print_status("Cancelled.", "info")
        sys.exit(0)

    if choice == "continue":
        return True
    elif choice == "review":
        review_existing_build(project_dir, spec_name)
        print()
        input(f"Press Enter to continue building...")
        return True
    elif choice == "merge":
        merge_existing_build(project_dir, spec_name)
        return False
    elif choice == "fresh":
        discarded = discard_existing_build(project_dir, spec_name)
        return not discarded
    else:
        return True
```

### 3. coordinator.py - Parallel Execution Changes

Update the `SwarmCoordinator` class to use per-spec worktrees:

```python
#!/usr/bin/env python3
"""
Multi-Agent Parallelism Coordinator - Per-Spec Architecture
============================================================

Workers create temporary worktrees that branch FROM the spec's worktree,
then merge back into it when complete.

Flow:
1. Spec worktree exists at .worktrees/{spec-name}/
2. Worker creates .worktrees/worker-{id}/ branching from spec's branch
3. Worker completes chunk
4. Worker merges into spec's worktree
5. Worker worktree is deleted
"""

# ... imports ...

class SwarmCoordinator:
    """Coordinates parallel execution using per-spec worktrees."""

    def __init__(
        self,
        spec_dir: Path,
        project_dir: Path,
        max_workers: int = 3,
        model: str = "claude-opus-4-5-20251101",
        verbose: bool = False,
    ):
        self.spec_dir = spec_dir
        self.project_dir = project_dir
        self.max_workers = max_workers
        self.model = model
        self.verbose = verbose
        self.spec_name = spec_dir.name

        # State tracking
        self.workers: dict[str, WorkerAssignment] = {}
        self.claimed_files: dict[str, str] = {}
        self.plan: Optional[ImplementationPlan] = None
        self.worktree_manager: Optional[WorktreeManager] = None

        self.progress_file = spec_dir / "parallel_progress.json"
        self._merge_lock = asyncio.Lock()
        self.status_manager = StatusManager(project_dir)

    async def run_worker(
        self,
        worker_id: str,
        phase: Phase,
        chunk: Chunk,
    ) -> bool:
        """Run a single worker on a chunk using a dedicated worktree."""

        # Create worker worktree branching FROM spec's worktree
        try:
            worktree_path, branch_name = self.worktree_manager.create_worker_worktree(
                self.spec_name, worker_id, chunk.id
            )
        except WorktreeError as e:
            print_status(f"Worker {worker_id}: Failed to create worktree: {e}", "error")
            return False

        # Claim the chunk
        if not self.claim_chunk(worker_id, phase, chunk, worktree_path, branch_name):
            print(f"Worker {worker_id}: Failed to claim chunk {chunk.id}")
            self.worktree_manager.cleanup_worker_worktree(worker_id, branch_name)
            return False

        try:
            # ... run agent session in worktree_path ...

            chunk_success = True  # Set based on agent result

            # Commit changes
            if chunk_success:
                self.worktree_manager.commit_in_worktree(self.spec_name,
                    f"auto-claude: Complete {chunk.id}\n\n{chunk.description}")

            # Release the chunk
            self.release_chunk(worker_id, chunk.id, chunk_success, None)

            # Merge to spec worktree if successful
            if chunk_success:
                async with self._merge_lock:
                    merge_success = self.worktree_manager.merge_worker_to_spec(
                        self.spec_name, branch_name
                    )
                    if not merge_success:
                        print_status(f"Worker {worker_id}: Merge failed for {chunk.id}", "error")
                        chunk_success = False

            # Clean up worker worktree
            self.worktree_manager.cleanup_worker_worktree(worker_id, branch_name)

            return chunk_success

        except Exception as e:
            print_status(f"Worker {worker_id}: Error: {e}", "error")
            self.release_chunk(worker_id, chunk.id, False, str(e))
            self.worktree_manager.cleanup_worker_worktree(worker_id, branch_name)
            return False

    async def run_parallel(self) -> Path:
        """Main coordination loop."""

        # Initialize worktree manager
        self.worktree_manager = WorktreeManager(self.project_dir)
        self.worktree_manager.setup()

        # Get or create worktree FOR THIS SPECIFIC SPEC
        spec_info = self.worktree_manager.get_or_create_worktree(self.spec_name)
        print_key_value("Spec worktree", str(spec_info.path))
        print_key_value("Spec branch", spec_info.branch)

        # ... rest of coordination loop ...

        # Cleanup: remove worker worktrees, preserve spec worktree
        self.worktree_manager.cleanup_all_workers()

        return spec_info.path
```

### 4. run.py - Integration Updates

Key changes needed in `run.py`:

```python
# In list_specs() function - update the build check
def list_specs(project_dir: Path, spec_dir: Path):
    """List all available specs with their status."""
    # ...
    for spec_folder in sorted(spec_dir.iterdir()):
        # ...
        # Check for existing build (uses spec-specific path now)
        has_build = get_existing_build_worktree(project_dir, folder_name) is not None
        # ...

# CRITICAL FIX for Bug #5: Pass auto_continue to finalize_workspace
# This prevents the process from hanging on interactive menu when spawned by UI
# Around line 745 in current run.py:
if worktree_manager:
    choice = finalize_workspace(
        project_dir,
        spec_dir.name,
        worktree_manager,
        auto_continue=args.auto_continue  # PASS THE FLAG - fixes hang!
    )
    # Only handle choice if interactive (not auto_continue)
    if not args.auto_continue:
        handle_workspace_choice(choice, project_dir, spec_dir.name, worktree_manager)

# The --cleanup-worktrees command (NEW)
parser.add_argument(
    "--cleanup-worktrees",
    action="store_true",
    help="Remove all worktrees and their branches",
)

# Handle cleanup command
if args.cleanup_worktrees:
    manager = WorktreeManager(project_dir)
    worktrees = manager.list_all_worktrees()
    if not worktrees:
        print_status("No worktrees to clean up.", "info")
    else:
        print(f"Found {len(worktrees)} worktree(s):")
        for wt in worktrees:
            print(f"  - {wt.spec_name} ({wt.branch})")
        print()
        confirm = input("Delete all? Type 'delete' to confirm: ")
        if confirm.strip().lower() == 'delete':
            manager.cleanup_all()
            print_status("All worktrees removed.", "success")
        else:
            print_status("Cancelled.", "info")
    return
```

---

## UI Backend Changes

### ipc-handlers.ts - Fixed Paths & Per-Spec Support

```typescript
import { execSync } from 'child_process';
import { existsSync, readdirSync, statSync } from 'fs';
import * as path from 'path';

/**
 * Get the worktree path for a specific spec
 */
function getWorktreePath(projectPath: string, specId: string): string {
  return path.join(projectPath, '.worktrees', specId);
}

/**
 * Get the branch name for a specific spec
 */
function getBranchName(specId: string): string {
  return `auto-claude/${specId}`;
}

/**
 * Get the worktree status for a task
 */
ipcMain.handle(
  IPC_CHANNELS.TASK_WORKTREE_STATUS,
  async (_, taskId: string): Promise<IPCResult<WorktreeStatus>> => {
    try {
      const { task, project } = findTaskAndProject(taskId);
      if (!task || !project) {
        return { success: false, error: 'Task not found' };
      }

      // Per-spec worktree path
      const worktreePath = getWorktreePath(project.path, task.specId);

      if (!existsSync(worktreePath)) {
        return {
          success: true,
          data: {
            exists: false,
            specId: task.specId
          }
        };
      }

      try {
        // Get current branch in worktree
        const branch = execSync('git rev-parse --abbrev-ref HEAD', {
          cwd: worktreePath,
          encoding: 'utf-8'
        }).trim();

        // Get base branch
        let baseBranch = 'main';
        try {
          const result = execSync('git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null', {
            cwd: project.path,
            encoding: 'utf-8'
          }).trim();
          baseBranch = result.replace('refs/remotes/origin/', '');
        } catch {
          baseBranch = 'main';
        }

        // Get commit count
        let commitCount = 0;
        try {
          const countOutput = execSync(`git rev-list --count ${baseBranch}..HEAD`, {
            cwd: worktreePath,
            encoding: 'utf-8'
          }).trim();
          commitCount = parseInt(countOutput, 10) || 0;
        } catch {
          commitCount = 0;
        }

        // Get diff stats
        let filesChanged = 0;
        let additions = 0;
        let deletions = 0;

        try {
          const diffStat = execSync(`git diff --shortstat ${baseBranch}...HEAD`, {
            cwd: worktreePath,
            encoding: 'utf-8'
          }).trim();

          const filesMatch = diffStat.match(/(\d+) files? changed/);
          const addMatch = diffStat.match(/(\d+) insertions?\(\+\)/);
          const delMatch = diffStat.match(/(\d+) deletions?\(-\)/);

          if (filesMatch) filesChanged = parseInt(filesMatch[1], 10);
          if (addMatch) additions = parseInt(addMatch[1], 10);
          if (delMatch) deletions = parseInt(delMatch[1], 10);
        } catch {
          // Ignore diff errors
        }

        return {
          success: true,
          data: {
            exists: true,
            specId: task.specId,
            worktreePath,
            branch,
            baseBranch,
            commitCount,
            filesChanged,
            additions,
            deletions
          }
        };
      } catch (gitError) {
        console.error('Git error getting worktree status:', gitError);
        return {
          success: true,
          data: { exists: true, specId: task.specId, worktreePath }
        };
      }
    } catch (error) {
      console.error('Failed to get worktree status:', error);
      return {
        success: false,
        error: error instanceof Error ? error.message : 'Failed to get worktree status'
      };
    }
  }
);

/**
 * Get the diff for a task's worktree
 */
ipcMain.handle(
  IPC_CHANNELS.TASK_WORKTREE_DIFF,
  async (_, taskId: string): Promise<IPCResult<WorktreeDiff>> => {
    try {
      const { task, project } = findTaskAndProject(taskId);
      if (!task || !project) {
        return { success: false, error: 'Task not found' };
      }

      const worktreePath = getWorktreePath(project.path, task.specId);

      if (!existsSync(worktreePath)) {
        return { success: false, error: 'No worktree found for this task' };
      }

      // Get base branch
      let baseBranch = 'main';
      try {
        const result = execSync('git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null', {
          cwd: project.path,
          encoding: 'utf-8'
        }).trim();
        baseBranch = result.replace('refs/remotes/origin/', '');
      } catch {
        baseBranch = 'main';
      }

      const files: WorktreeDiffFile[] = [];

      try {
        const numstat = execSync(`git diff --numstat ${baseBranch}...HEAD`, {
          cwd: worktreePath,
          encoding: 'utf-8'
        }).trim();

        const nameStatus = execSync(`git diff --name-status ${baseBranch}...HEAD`, {
          cwd: worktreePath,
          encoding: 'utf-8'
        }).trim();

        // Parse name-status
        const statusMap: Record<string, 'added' | 'modified' | 'deleted' | 'renamed'> = {};
        for (const line of nameStatus.split('\n')) {
          if (!line) continue;
          const [status, filePath] = line.split('\t');
          if (status === 'A') statusMap[filePath] = 'added';
          else if (status === 'M') statusMap[filePath] = 'modified';
          else if (status === 'D') statusMap[filePath] = 'deleted';
          else if (status.startsWith('R')) statusMap[filePath] = 'renamed';
        }

        // Parse numstat
        for (const line of numstat.split('\n')) {
          if (!line) continue;
          const [add, del, filePath] = line.split('\t');
          files.push({
            path: filePath,
            status: statusMap[filePath] || 'modified',
            additions: parseInt(add, 10) || 0,
            deletions: parseInt(del, 10) || 0
          });
        }
      } catch {
        // Ignore parse errors
      }

      const summary = `${files.length} file(s) changed`;

      return {
        success: true,
        data: { files, summary }
      };
    } catch (error) {
      console.error('Failed to get worktree diff:', error);
      return {
        success: false,
        error: error instanceof Error ? error.message : 'Failed to get worktree diff'
      };
    }
  }
);

/**
 * Merge the worktree changes into the main branch
 */
ipcMain.handle(
  IPC_CHANNELS.TASK_WORKTREE_MERGE,
  async (_, taskId: string): Promise<IPCResult<WorktreeMergeResult>> => {
    try {
      const { task, project } = findTaskAndProject(taskId);
      if (!task || !project) {
        return { success: false, error: 'Task not found' };
      }

      const worktreePath = getWorktreePath(project.path, task.specId);

      if (!existsSync(worktreePath)) {
        return {
          success: false,
          error: 'No worktree found for this task'
        };
      }

      // Run Python merge command
      return new Promise((resolve) => {
        const proc = spawn('python', [
          path.join(project.path, 'auto-claude', 'run.py'),
          '--spec', task.specId,
          '--merge'
        ], {
          cwd: project.path,
          env: process.env
        });

        let output = '';
        proc.stdout.on('data', (data) => { output += data.toString(); });
        proc.stderr.on('data', (data) => { output += data.toString(); });

        proc.on('close', (code) => {
          if (code === 0) {
            resolve({
              success: true,
              data: {
                success: true,
                specId: task.specId,
                message: 'Worktree merged successfully'
              }
            });
          } else {
            resolve({
              success: false,
              error: `Merge failed: ${output}`
            });
          }
        });
      });
    } catch (error) {
      console.error('Failed to merge worktree:', error);
      return {
        success: false,
        error: error instanceof Error ? error.message : 'Failed to merge worktree'
      };
    }
  }
);

/**
 * Discard the worktree changes
 */
ipcMain.handle(
  IPC_CHANNELS.TASK_WORKTREE_DISCARD,
  async (_, taskId: string): Promise<IPCResult<WorktreeDiscardResult>> => {
    try {
      const { task, project } = findTaskAndProject(taskId);
      if (!task || !project) {
        return { success: false, error: 'Task not found' };
      }

      const worktreePath = getWorktreePath(project.path, task.specId);

      if (!existsSync(worktreePath)) {
        return {
          success: true,
          data: {
            success: true,
            message: 'No worktree to discard'
          }
        };
      }

      try {
        // Get the branch name
        const branch = execSync('git rev-parse --abbrev-ref HEAD', {
          cwd: worktreePath,
          encoding: 'utf-8'
        }).trim();

        // Remove the worktree
        execSync(`git worktree remove --force "${worktreePath}"`, {
          cwd: project.path,
          encoding: 'utf-8'
        });

        // Delete the branch
        try {
          execSync(`git branch -D "${branch}"`, {
            cwd: project.path,
            encoding: 'utf-8'
          });
        } catch {
          // Branch might already be deleted
        }

        // Prune worktrees
        execSync('git worktree prune', {
          cwd: project.path,
          encoding: 'utf-8'
        });

        return {
          success: true,
          data: {
            success: true,
            message: 'Worktree discarded successfully'
          }
        };
      } catch (gitError) {
        console.error('Git error discarding worktree:', gitError);
        return {
          success: false,
          error: `Failed to discard worktree: ${gitError instanceof Error ? gitError.message : 'Unknown error'}`
        };
      }
    } catch (error) {
      console.error('Failed to discard worktree:', error);
      return {
        success: false,
        error: error instanceof Error ? error.message : 'Failed to discard worktree'
      };
    }
  }
);

/**
 * List all worktrees in a project (for dashboard)
 */
ipcMain.handle(
  'task:listWorktrees',
  async (_, projectId: string): Promise<IPCResult<WorktreeInfo[]>> => {
    try {
      const project = projectStore.getProject(projectId);
      if (!project) {
        return { success: false, error: 'Project not found' };
      }

      const worktreesDir = path.join(project.path, '.worktrees');
      if (!existsSync(worktreesDir)) {
        return { success: true, data: [] };
      }

      const worktrees: WorktreeInfo[] = [];

      for (const specId of readdirSync(worktreesDir)) {
        // Skip worker worktrees
        if (specId.startsWith('worker-')) continue;

        const worktreePath = path.join(worktreesDir, specId);
        const stat = statSync(worktreePath);

        if (!stat.isDirectory()) continue;

        try {
          const branch = execSync('git rev-parse --abbrev-ref HEAD', {
            cwd: worktreePath,
            encoding: 'utf-8'
          }).trim();

          worktrees.push({
            specId,
            worktreePath,
            branch,
            exists: true
          });
        } catch {
          // Skip invalid worktrees
        }
      }

      return { success: true, data: worktrees };
    } catch (error) {
      return {
        success: false,
        error: error instanceof Error ? error.message : 'Failed to list worktrees'
      };
    }
  }
);
```

---

## Type Definitions (shared/types.ts)

Add/update these interfaces:

```typescript
/**
 * Worktree status for a specific spec
 */
export interface WorktreeStatus {
  exists: boolean;
  specId: string;              // Which spec this worktree is for
  worktreePath?: string;
  branch?: string;
  baseBranch?: string;
  commitCount?: number;
  filesChanged?: number;
  additions?: number;
  deletions?: number;
}

/**
 * Summary of a worktree in a project (for listing)
 */
export interface WorktreeInfo {
  specId: string;
  worktreePath: string;
  branch: string;
  exists: boolean;
}

/**
 * Result of a worktree merge operation
 */
export interface WorktreeMergeResult {
  success: boolean;
  specId: string;
  message: string;
  conflicts?: string[];
}

/**
 * Result of a worktree discard operation
 */
export interface WorktreeDiscardResult {
  success: boolean;
  message: string;
}
```

---

## Summary of Changes

| Component | Old Behavior | New Behavior |
|-----------|--------------|--------------|
| **Worktree Path** | `.worktrees/auto-claude/` | `.worktrees/{spec-name}/` |
| **Branch Name** | `auto-claude/{spec-name}` | Same (unchanged) |
| **Worktree Count** | ONE for all specs | ONE per spec |
| **UI Path** | `.worktrees/auto-claude-staging/` (wrong!) | `.worktrees/{spec-name}/` |
| **Branch on Merge** | Deleted by default | Preserved by default |
| **Parallel Specs** | Not supported | Fully supported |
| **Worker Worktrees** | Branch from staging | Branch from spec's worktree |
| **Post-Build Menu** | Always interactive (hangs UI) | Skipped when `--auto-continue` |

## Task Status Flow

When UI spawns task execution with `--auto-continue`:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   in_progress   │───▶│   ai_review     │───▶│  human_review   │───▶│      done       │
│                 │    │                 │    │                 │    │                 │
│ Chunks building │    │ QA agent runs   │    │ Process exits   │    │ User merges     │
│                 │    │                 │    │ with code 0     │    │ via UI/CLI      │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
```

**Key Fix:** When `--auto-continue` is set:
1. `finalize_workspace()` skips interactive menu
2. Process exits cleanly with code 0
3. UI detects exit → sets status to `human_review`
4. User can merge/review/discard via UI buttons

---

## Benefits

1. **Isolation**: Each spec's work is completely isolated
2. **Parallel Work**: Can work on multiple specs simultaneously
3. **Clear Mapping**: spec → worktree → branch (1:1:1)
4. **Persistence**: Branches remain until explicitly merged
5. **Discoverability**: Easy to list all in-progress specs
6. **No More Bugs**: Fixes path mismatch, spec confusion, and branch deletion issues

---

## CLI Commands

```bash
# Build a spec (creates per-spec worktree)
python auto-claude/run.py --spec 002-feature

# Parallel execution (workers branch from spec worktree)
python auto-claude/run.py --spec 002-feature --parallel 3

# Review build
python auto-claude/run.py --spec 002-feature --review

# Merge build to project
python auto-claude/run.py --spec 002-feature --merge

# Discard build
python auto-claude/run.py --spec 002-feature --discard

# Clean up ALL worktrees (NEW)
python auto-claude/run.py --cleanup-worktrees
```

---

## Implementation Checklist

### Python Backend
- [ ] Replace `worktree.py` with per-spec implementation
- [ ] Replace `workspace.py` with per-spec implementation (includes Bug #5 fix)
- [ ] Update `coordinator.py` for per-spec parallel execution
- [ ] Update `run.py`:
  - [ ] Add `--cleanup-worktrees` command
  - [ ] Pass `auto_continue` to `finalize_workspace()` (Bug #5 fix)

### UI Backend (TypeScript)
- [ ] Fix `ipc-handlers.ts` worktree paths (use `getWorktreePath(projectPath, specId)`)
- [ ] Update TypeScript types in `shared/types.ts`
- [ ] Add `task:listWorktrees` IPC handler

### Testing
- [ ] Test single spec workflow (CLI)
- [ ] Test single spec workflow (UI) - verify no hang on completion
- [ ] Test parallel spec workflow
- [ ] Test multiple specs simultaneously
- [ ] Test merge/discard operations
- [ ] Verify status transitions: `in_progress` → `human_review` → `done`
