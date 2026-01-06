# Implementation Plan for Issue #125: Repeat Options Feature

## Overview
Implement a `--last` flag that allows re-executing previously run commands with their original parameters, using environment variables to maintain state within the current shell session.

## Requirements

### Usage Patterns
1. `vtr task1 --last` - runs task1 using the input values and arguments from its previous execution
2. `vtr --last` - executes the last task with its stored parameters

### Design Constraints
- Use environment variable to store state (avoid filesystem clutter)
- State should be confined to current shell session
- Should work alongside issue #124 (replay CLI-supplied inputs)

## Current Architecture Analysis

### Key Components
- **Entry Point**: `vscode_task_runner/console.py::run()`
- **Argument Parser**: `vscode_task_runner/parser.py::parse_args()`
- **Task Executor**: `vscode_task_runner/executor.py::execute_tasks()`
- **Variable Resolution**: `vscode_task_runner/variables/resolve.py`
- **Input Handling**: Uses `VTR_INPUT_{input_id}` environment variables for overrides

### Data Flow
```
CLI → parse_args() → ArgParseResult(task_labels, extra_args)
    → execute_tasks() → resolve_variables() → subprocess.Popen()
```

## Implementation Strategy

### 1. Storage Design

**Environment Variable**: `VTR_LAST_EXECUTION`

**Storage Format** (JSON-encoded):
```json
{
  "task_labels": ["task1", "task2"],
  "extra_args": ["--arg1", "value1"],
  "inputs": {
    "input_id_1": "value1",
    "input_id_2": "value2"
  }
}
```

### 2. Components to Modify

#### A. Add New Model (`vscode_task_runner/models/last_execution.py`)
- Create Pydantic model for `LastExecution` containing:
  - `task_labels: list[str]`
  - `extra_args: list[str]`
  - `inputs: dict[str, str]`
- Methods:
  - `to_json()` - serialize to JSON string
  - `from_json()` - deserialize from JSON string
  - `to_env_var()` - encode for environment variable storage
  - `from_env_var()` - decode from environment variable

#### B. Update Argument Parser (`vscode_task_runner/parser.py`)
- Add `--last` flag to argparse in `parse_args()`
- Add logic to:
  1. Check if `--last` flag is present
  2. Load `VTR_LAST_EXECUTION` from environment
  3. Merge with current arguments:
     - If task labels provided: use them, but load extra_args and inputs from last execution
     - If no task labels: load everything from last execution
  4. Validate that last execution data exists when `--last` is used
- Handle error cases:
  - No previous execution found
  - Corrupted/invalid JSON data

#### C. Update Console (`vscode_task_runner/console.py`)
- After successful task execution, capture execution state
- Create `LastExecution` object with:
  - Task labels from `ArgParseResult`
  - Extra args from `ArgParseResult`
  - Input values from `RUNTIME_VARIABLES.INPUTS`
- Export to environment variable for parent shell
- Print instruction to user on how to export (similar to direnv or other shell tools)

#### D. Update Runtime Variables (`vscode_task_runner/variables/runtime.py`)
- When `--last` is used with stored inputs:
  - Populate `VTR_INPUT_*` environment variables before execution
  - This integrates with existing input resolution system

### 3. Detailed Implementation Steps

#### Step 1: Create LastExecution Model
**File**: `vscode_task_runner/models/last_execution.py`

```python
import json
import os
from typing import Optional
from pydantic import BaseModel, Field


class LastExecution(BaseModel):
    """Model for storing last execution state."""
    task_labels: list[str] = Field(default_factory=list)
    extra_args: list[str] = Field(default_factory=list)
    inputs: dict[str, str] = Field(default_factory=dict)

    def to_json(self) -> str:
        """Serialize to JSON string."""
        return json.dumps(self.model_dump())

    @classmethod
    def from_json(cls, json_str: str) -> "LastExecution":
        """Deserialize from JSON string."""
        return cls(**json.loads(json_str))

    @classmethod
    def from_env_var(cls, env_var_name: str = "VTR_LAST_EXECUTION") -> Optional["LastExecution"]:
        """Load from environment variable."""
        value = os.environ.get(env_var_name)
        if not value:
            return None
        return cls.from_json(value)

    def apply_to_environment(self) -> None:
        """Apply stored inputs to environment variables."""
        for input_id, value in self.inputs.items():
            os.environ[f"VTR_INPUT_{input_id}"] = value
```

#### Step 2: Update Argument Parser
**File**: `vscode_task_runner/parser.py`

Add `--last` flag:
```python
parser.add_argument(
    "--last",
    action="store_true",
    help="Re-execute using parameters from the last execution",
)
```

Add logic after parsing to merge with last execution:
```python
if args.last:
    last_exec = LastExecution.from_env_var()
    if last_exec is None:
        raise Exception("No previous execution found. Run a task first before using --last.")

    # If task labels provided, use them; otherwise use last execution's tasks
    if not args.task_labels:
        args.task_labels = last_exec.task_labels

    # Always use extra args from last execution when --last is used
    # This allows: vtr task1 --last
    if not extra_args:
        extra_args = last_exec.extra_args

    # Apply input values to environment
    last_exec.apply_to_environment()
```

#### Step 3: Capture and Export Execution State
**File**: `vscode_task_runner/console.py`

After successful execution in `run()`:
```python
# Capture last execution state
from vscode_task_runner.models.last_execution import LastExecution
from vscode_task_runner.variables.runtime import RUNTIME_VARIABLES

last_exec = LastExecution(
    task_labels=result.task_labels,
    extra_args=result.extra_args,
    inputs=RUNTIME_VARIABLES.INPUTS,
)

# Print export command for user's shell
print("\nTo enable --last flag, run:")
print(f'export VTR_LAST_EXECUTION=\'{last_exec.to_json()}\'')
```

**Alternative approach**: Investigate if we can automatically export to parent shell (may require shell-specific wrapper scripts)

#### Step 4: Add Tests
**Files**: `tests/test_last_execution.py`, `tests/test_parser.py` updates

Test cases:
1. `LastExecution` model serialization/deserialization
2. `--last` with no previous execution (should error)
3. `--last` alone (uses last task labels and args)
4. `vtr task1 --last` (uses new task label but last args/inputs)
5. Multiple tasks with `--last`
6. Integration test with actual task execution and replay

#### Step 5: Update Documentation
**File**: `README.md`

Add section explaining:
- How to use `--last` flag
- How to export the execution state
- Example workflows

### 4. Edge Cases to Handle

1. **No previous execution**: Error message with helpful instruction
2. **Corrupted JSON in environment variable**: Clear error message, suggest clearing `VTR_LAST_EXECUTION`
3. **Tasks not found**: Normal task not found error should apply
4. **Input validation**: Ensure stored input values still valid for pickString options
5. **Multiple tasks with --last**: Should work normally
6. **Combining --last with extra args**: Last execution's extra args should be used, not new ones (unless we want to override)

### 5. Alternative Considerations

**Auto-export vs Manual export**:
- **Manual** (recommended): Print export command, user copies/pastes
  - Pros: Cross-shell compatible, no complex shell detection
  - Cons: Requires manual step

- **Auto-export**: Use shell-specific wrapper
  - Pros: Seamless UX
  - Cons: Complex, requires shell-specific code or wrapper scripts

**Recommendation**: Start with manual export, can enhance later with auto-export

### 6. Testing Strategy

1. **Unit Tests**:
   - LastExecution model methods
   - Parser with --last flag
   - Environment variable handling

2. **Integration Tests**:
   - Full execution cycle with state capture
   - Replay with --last
   - Input value preservation

3. **Manual Testing**:
   - Test in bash, zsh, PowerShell
   - Test both usage patterns
   - Test error cases

## Summary

This implementation provides a clean, environment-based solution for repeating task executions. The approach:
- ✅ Uses environment variables (per requirement)
- ✅ Keeps state in current shell session
- ✅ Integrates with existing input override system
- ✅ Supports both usage patterns from the issue
- ✅ Minimal changes to existing architecture
- ✅ Backwards compatible

## Files to Create/Modify

### Create:
- `vscode_task_runner/models/last_execution.py` - New model for state storage
- `tests/test_last_execution.py` - Tests for new model

### Modify:
- `vscode_task_runner/parser.py` - Add --last flag and merge logic
- `vscode_task_runner/console.py` - Capture and export state
- `vscode_task_runner/models/__init__.py` - Export new model
- `tests/test_parser.py` - Add tests for --last flag
- `README.md` - Documentation

## Implementation Order

1. Create `LastExecution` model with tests
2. Update parser to add `--last` flag and merge logic
3. Update console to capture and export state
4. Add integration tests
5. Update documentation
6. Manual testing across shells

## Open Questions

1. Should `vtr task1 --last --extra-arg` override the stored extra args or append to them?
   - **Recommendation**: Last execution's args take precedence when `--last` is used

2. Should we support clearing the last execution state?
   - **Recommendation**: User can `unset VTR_LAST_EXECUTION` manually

3. Should we limit the size of stored state?
   - **Recommendation**: No limits initially; environment variables can handle reasonable JSON sizes
