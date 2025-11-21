# Implementation Plan: Command Output Separation in Zellij (TDD Approach)

## Overview

**Goal**: Add support for command output separators to enable logging terminal output with clear boundaries between commands.

**Approach**: Implement OSC 133 (Shell Integration Protocol) support in Zellij's terminal emulator using Test-Driven Development (TDD).

**Development Method**: RED-GREEN-REFACTOR cycles
- 🔴 **RED**: Write a failing test
- 🟢 **GREEN**: Write minimal code to make it pass
- 🔵 **REFACTOR**: Clean up and optimize

## Why OSC 133?

- **Industry Standard**: Used by VSCode, iTerm2, WezTerm, and other modern terminals
- **Rich Metadata**: Supports prompt markers, command text, exit codes, timestamps
- **Invisible**: No visual output, purely semantic markers
- **Future-Proof**: If Zellij adds full shell integration later, this will be compatible
- **Easy to Emit**: Simple escape sequences from shell hooks

## OSC 133 Protocol Specification

The protocol uses `ESC]133;<marker>;[params]ESC\` format:

- **OSC 133;A**: Prompt start (before PS1)
- **OSC 133;B**: Prompt end / Command line start (after PS1, before user types)
- **OSC 133;C**: Command execution start (after user presses Enter)
- **OSC 133;D;[exit_code]**: Command end with optional exit code
- **OSC 133;E;[command_text]**: Command line text (optional)

## TDD Implementation Cycles

---

## 🔴 CYCLE 1: OSC 133;A - Basic Prompt Start Marker

### RED: Write Failing Test First

**File**: `zellij-server/src/panes/unit/grid_tests.rs`

**Location**: Add at the end of the file (before closing braces)

```rust
#[test]
fn test_osc_133_prompt_start_marker() {
    // Arrange: Create a test grid
    let mut grid = create_test_grid();

    // Act: Send OSC 133;A (prompt start)
    let params = vec![b"133".as_ref(), b"A".as_ref()];
    grid.osc_dispatch(&params, true);

    // Assert: Should create one PromptStart marker
    assert_eq!(grid.command_markers.len(), 1);
    assert_eq!(grid.command_markers[0].marker_type, CommandMarkerType::PromptStart);
    assert_eq!(grid.command_markers[0].line_number, 0);
    assert_eq!(grid.command_markers[0].column_number, 0);
}
```

**Run the test** (it will fail):
```bash
cd /home/user/zellij
cargo test test_osc_133_prompt_start_marker
```

**Expected failure**: Compilation errors about missing types.

### GREEN: Minimal Implementation to Pass

**File**: `zellij-server/src/panes/grid.rs`

**Step 1**: Add data structures (around line 40, after imports)

```rust
use std::time::SystemTime;

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum CommandMarkerType {
    PromptStart,
}

#[derive(Debug, Clone)]
pub struct CommandMarker {
    pub marker_type: CommandMarkerType,
    pub line_number: usize,
    pub column_number: usize,
    pub timestamp: u64,
}
```

**Step 2**: Add storage to Grid struct (around line 314)

```rust
pub struct Grid {
    // ... existing fields ...
    pub command_markers: Vec<CommandMarker>,
}
```

**Step 3**: Initialize in `Grid::new()` (around line 503)

```rust
Grid {
    // ... existing fields ...
    command_markers: Vec::new(),
}
```

**Step 4**: Add minimal OSC handler in `osc_dispatch()` (around line 2575, before default case)

```rust
match params[0] {
    // ... existing cases ...

    b"133" => {
        if params.len() >= 2 && params[1] == b"A" {
            let timestamp = SystemTime::now()
                .duration_since(SystemTime::UNIX_EPOCH)
                .map(|d| d.as_millis() as u64)
                .unwrap_or(0);

            self.command_markers.push(CommandMarker {
                marker_type: CommandMarkerType::PromptStart,
                line_number: self.cursor.y + self.lines_above.len(),
                column_number: self.cursor.x,
                timestamp,
            });
        }
    },

    _ => { /* ... */ }
}
```

**Run the test again**:
```bash
cargo test test_osc_133_prompt_start_marker
```

**Expected**: Test passes! ✅

### REFACTOR: Clean Up (Optional for Cycle 1)

No refactoring needed yet - code is simple enough.

---

## 🔴 CYCLE 2: Add Remaining Marker Types (B, C, D, E)

### RED: Write Tests for All Marker Types

**File**: `zellij-server/src/panes/unit/grid_tests.rs`

```rust
#[test]
fn test_osc_133_prompt_end_marker() {
    let mut grid = create_test_grid();

    let params = vec![b"133".as_ref(), b"B".as_ref()];
    grid.osc_dispatch(&params, true);

    assert_eq!(grid.command_markers.len(), 1);
    assert_eq!(grid.command_markers[0].marker_type, CommandMarkerType::PromptEnd);
}

#[test]
fn test_osc_133_command_start_marker() {
    let mut grid = create_test_grid();

    let params = vec![b"133".as_ref(), b"C".as_ref()];
    grid.osc_dispatch(&params, true);

    assert_eq!(grid.command_markers.len(), 1);
    assert_eq!(grid.command_markers[0].marker_type, CommandMarkerType::CommandExecutionStart);
}

#[test]
fn test_osc_133_command_end_marker_with_exit_code() {
    let mut grid = create_test_grid();

    let params = vec![b"133".as_ref(), b"D".as_ref(), b"127".as_ref()];
    grid.osc_dispatch(&params, true);

    assert_eq!(grid.command_markers.len(), 1);
    assert_eq!(grid.command_markers[0].marker_type, CommandMarkerType::CommandEnd);
    assert_eq!(grid.command_markers[0].exit_code, Some(127));
}

#[test]
fn test_osc_133_command_line_text() {
    let mut grid = create_test_grid();

    let params = vec![b"133".as_ref(), b"E".as_ref(), b"ls -la".as_ref()];
    grid.osc_dispatch(&params, true);

    assert_eq!(grid.command_markers.len(), 1);
    assert_eq!(grid.command_markers[0].marker_type, CommandMarkerType::CommandLine);
    assert_eq!(grid.command_markers[0].command_text, Some("ls -la".to_string()));
}
```

**Run tests** (they will fail):
```bash
cargo test test_osc_133
```

**Expected failures**: Missing enum variants and struct fields.

### GREEN: Extend Implementation

**File**: `zellij-server/src/panes/grid.rs`

**Step 1**: Extend `CommandMarkerType` enum

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum CommandMarkerType {
    PromptStart,           // OSC 133;A
    PromptEnd,             // OSC 133;B
    CommandExecutionStart, // OSC 133;C
    CommandEnd,            // OSC 133;D
    CommandLine,           // OSC 133;E
}
```

**Step 2**: Extend `CommandMarker` struct

```rust
#[derive(Debug, Clone)]
pub struct CommandMarker {
    pub marker_type: CommandMarkerType,
    pub line_number: usize,
    pub column_number: usize,
    pub exit_code: Option<i32>,
    pub command_text: Option<String>,
    pub timestamp: u64,
}
```

**Step 3**: Update OSC handler to handle all types

```rust
b"133" => {
    if params.len() >= 2 {
        let marker_type = match params[1] {
            b"A" => Some(CommandMarkerType::PromptStart),
            b"B" => Some(CommandMarkerType::PromptEnd),
            b"C" => Some(CommandMarkerType::CommandExecutionStart),
            b"D" => Some(CommandMarkerType::CommandEnd),
            b"E" => Some(CommandMarkerType::CommandLine),
            _ => None,
        };

        if let Some(marker_type) = marker_type {
            let timestamp = SystemTime::now()
                .duration_since(SystemTime::UNIX_EPOCH)
                .map(|d| d.as_millis() as u64)
                .unwrap_or(0);

            let exit_code = if marker_type == CommandMarkerType::CommandEnd && params.len() >= 3 {
                std::str::from_utf8(params[2])
                    .ok()
                    .and_then(|s| s.trim().parse::<i32>().ok())
            } else {
                None
            };

            let command_text = if marker_type == CommandMarkerType::CommandLine && params.len() >= 3 {
                std::str::from_utf8(params[2])
                    .ok()
                    .map(|s| s.to_string())
            } else {
                None
            };

            self.command_markers.push(CommandMarker {
                marker_type,
                line_number: self.cursor.y + self.lines_above.len(),
                column_number: self.cursor.x,
                exit_code,
                command_text,
                timestamp,
            });
        }
    }
},
```

**Run tests**:
```bash
cargo test test_osc_133
```

**Expected**: All tests pass! ✅

### REFACTOR: Extract Helper Method

**File**: `zellij-server/src/panes/grid.rs`

Add before `impl Perform for Grid` (around line 2470):

```rust
impl Grid {
    // ... existing methods ...

    fn record_command_marker(&mut self, marker_type: CommandMarkerType, params: &[&[u8]]) {
        let timestamp = SystemTime::now()
            .duration_since(SystemTime::UNIX_EPOCH)
            .map(|d| d.as_millis() as u64)
            .unwrap_or(0);

        let exit_code = if marker_type == CommandMarkerType::CommandEnd && params.len() >= 3 {
            std::str::from_utf8(params[2])
                .ok()
                .and_then(|s| s.trim().parse::<i32>().ok())
        } else {
            None
        };

        let command_text = if marker_type == CommandMarkerType::CommandLine && params.len() >= 3 {
            std::str::from_utf8(params[2])
                .ok()
                .map(|s| s.to_string())
        } else {
            None
        };

        self.command_markers.push(CommandMarker {
            marker_type,
            line_number: self.cursor.y + self.lines_above.len(),
            column_number: self.cursor.x,
            exit_code,
            command_text,
            timestamp,
        });

        if self.debug {
            log::info!("Command marker recorded: {:?}", self.command_markers.last());
        }
    }
}
```

Update OSC handler to use helper:

```rust
b"133" => {
    if params.len() >= 2 {
        match params[1] {
            b"A" => self.record_command_marker(CommandMarkerType::PromptStart, params),
            b"B" => self.record_command_marker(CommandMarkerType::PromptEnd, params),
            b"C" => self.record_command_marker(CommandMarkerType::CommandExecutionStart, params),
            b"D" => self.record_command_marker(CommandMarkerType::CommandEnd, params),
            b"E" => self.record_command_marker(CommandMarkerType::CommandLine, params),
            _ => {
                if self.debug {
                    log::warn!("Unknown OSC 133 subcommand: {:?}", params[1]);
                }
            },
        }
    }
},
```

**Run tests again**:
```bash
cargo test test_osc_133
```

**Expected**: Still passing! ✅

---

## 🔴 CYCLE 3: Add Public API for Marker Access

### RED: Write Tests for Getter Methods

**File**: `zellij-server/src/panes/unit/grid_tests.rs`

```rust
#[test]
fn test_get_command_markers() {
    let mut grid = create_test_grid();

    // Add multiple markers
    let params_a = vec![b"133".as_ref(), b"A".as_ref()];
    let params_c = vec![b"133".as_ref(), b"C".as_ref()];
    grid.osc_dispatch(&params_a, true);
    grid.osc_dispatch(&params_c, true);

    let markers = grid.get_command_markers();
    assert_eq!(markers.len(), 2);
    assert_eq!(markers[0].marker_type, CommandMarkerType::PromptStart);
    assert_eq!(markers[1].marker_type, CommandMarkerType::CommandExecutionStart);
}

#[test]
fn test_get_markers_in_range() {
    let mut grid = create_test_grid();

    // Simulate cursor at different lines
    grid.cursor.y = 0;
    grid.osc_dispatch(&[b"133", b"A"], true);

    grid.cursor.y = 5;
    grid.osc_dispatch(&[b"133", b"C"], true);

    grid.cursor.y = 10;
    grid.osc_dispatch(&[b"133", b"D", b"0"], true);

    let markers_in_range = grid.get_markers_in_range(4, 8);
    assert_eq!(markers_in_range.len(), 1);
    assert_eq!(markers_in_range[0].marker_type, CommandMarkerType::CommandExecutionStart);
}
```

**Run tests** (they will fail):
```bash
cargo test test_get_command_markers
```

**Expected failures**: Method not found.

### GREEN: Implement Getter Methods

**File**: `zellij-server/src/panes/grid.rs`

Add to `impl Grid`:

```rust
impl Grid {
    // ... existing methods ...

    pub fn get_command_markers(&self) -> &[CommandMarker] {
        &self.command_markers
    }

    pub fn get_markers_in_range(&self, start_line: usize, end_line: usize) -> Vec<&CommandMarker> {
        self.command_markers
            .iter()
            .filter(|m| m.line_number >= start_line && m.line_number <= end_line)
            .collect()
    }
}
```

**Run tests**:
```bash
cargo test test_get_command_markers
```

**Expected**: Tests pass! ✅

### REFACTOR: No refactoring needed

---

## 🔴 CYCLE 4: Handle Scrollback Cleanup

### RED: Write Test for Marker Cleanup

**File**: `zellij-server/src/panes/unit/grid_tests.rs`

```rust
#[test]
fn test_clear_markers_before_line() {
    let mut grid = create_test_grid();

    // Add markers at lines 0, 5, 10
    grid.cursor.y = 0;
    grid.osc_dispatch(&[b"133", b"A"], true);

    grid.cursor.y = 5;
    grid.osc_dispatch(&[b"133", b"C"], true);

    grid.cursor.y = 10;
    grid.osc_dispatch(&[b"133", b"D", b"0"], true);

    assert_eq!(grid.command_markers.len(), 3);

    // Clear markers before line 6
    grid.clear_markers_before_line(6);

    assert_eq!(grid.command_markers.len(), 1);
    assert_eq!(grid.command_markers[0].line_number, 10);
}
```

**Run test** (will fail):
```bash
cargo test test_clear_markers_before_line
```

### GREEN: Implement Cleanup Method

**File**: `zellij-server/src/panes/grid.rs`

```rust
impl Grid {
    // ... existing methods ...

    pub fn clear_markers_before_line(&mut self, line: usize) {
        self.command_markers.retain(|m| m.line_number >= line);
    }
}
```

**Run test**:
```bash
cargo test test_clear_markers_before_line
```

**Expected**: Test passes! ✅

---

## 🔴 CYCLE 5: Integration Testing with VTE Parser

### RED: Write End-to-End Test

**File**: `zellij-server/src/panes/unit/grid_tests.rs`

```rust
#[test]
fn test_osc_133_via_vte_parser() {
    use vte::{Parser, Perform};

    let mut grid = create_test_grid();
    let mut parser = Parser::new();

    // Send complete OSC 133;A sequence through VTE parser
    let sequence = b"\x1b]133;A\x1b\\";
    for &byte in sequence {
        parser.advance(&mut grid, byte);
    }

    assert_eq!(grid.command_markers.len(), 1);
    assert_eq!(grid.command_markers[0].marker_type, CommandMarkerType::PromptStart);
}

#[test]
fn test_full_command_cycle_via_vte() {
    use vte::{Parser, Perform};

    let mut grid = create_test_grid();
    let mut parser = Parser::new();

    // Prompt start
    for &byte in b"\x1b]133;A\x1b\\" { parser.advance(&mut grid, byte); }

    // Command start
    for &byte in b"\x1b]133;C\x1b\\" { parser.advance(&mut grid, byte); }

    // Command end with exit code
    for &byte in b"\x1b]133;D;0\x1b\\" { parser.advance(&mut grid, byte); }

    assert_eq!(grid.command_markers.len(), 3);
    assert_eq!(grid.command_markers[2].exit_code, Some(0));
}
```

**Run test**:
```bash
cargo test test_osc_133_via_vte_parser
```

**Expected**: Should pass if VTE parser correctly calls `osc_dispatch()` ✅

---

## Manual Integration Testing

After all automated tests pass, create a manual test script:

**File**: `test-osc133.sh`

```bash
#!/bin/bash

echo "Testing OSC 133 Shell Integration"

# Prompt start
printf '\e]133;A\e\\'
echo -n "$ "

# Prompt end
printf '\e]133;B\e\\'

# Command line (optional)
printf '\e]133;E;echo "Hello World"\e\\'

# Command execution start
printf '\e]133;C\e\\'

# Actual command output
echo "Hello World"

# Command end with exit code
printf '\e]133;D;0\e\\'

echo ""
echo "Test complete - markers should be recorded"
```

Make executable and run:
```bash
chmod +x test-osc133.sh
zellij --debug  # Debug mode to see log messages
# In pane, run:
./test-osc133.sh
# Check logs for "Command marker recorded:" messages
```

---

## TDD Summary

**Cycles Completed:**
1. ✅ Basic OSC 133;A support
2. ✅ All marker types (A, B, C, D, E)
3. ✅ Public API for marker access
4. ✅ Scrollback cleanup
5. ✅ End-to-end VTE integration

**Test Coverage:**
- Unit tests for each marker type
- Tests for exit code parsing
- Tests for command text extraction
- Tests for range queries
- Tests for cleanup operations
- Integration tests with VTE parser

---

## Shell Integration Setup (Post-Implementation)

Once all tests pass and the feature is implemented, configure your shell to emit OSC 133 markers:

### Bash Configuration

Add to `~/.bashrc`:

```bash
# Zellij shell integration
if [ -n "$ZELLIJ" ]; then
    # Prompt start marker
    PS0=$'\e]133;C\e\\'

    # Add markers around PS1
    PS1='\[\e]133;A\e\\\]$ \[\e]133;B\e\\\]'

    # Command end marker with exit code
    __zellij_precmd() {
        local exit_code=$?
        printf '\e]133;D;%s\e\\' "$exit_code"
    }

    if [[ "$PROMPT_COMMAND" != *"__zellij_precmd"* ]]; then
        PROMPT_COMMAND="__zellij_precmd${PROMPT_COMMAND:+;$PROMPT_COMMAND}"
    fi
fi
```

### Zsh Configuration

Add to `~/.zshrc`:

```zsh
# Zellij shell integration
if [ -n "$ZELLIJ" ]; then
    precmd() {
        printf '\e]133;D;%s\e\\' $?
        printf '\e]133;A\e\\'
    }

    preexec() {
        printf '\e]133;C\e\\'
    }

    PS1='%{$(printf "\e]133;A\e\\")%}$ %{$(printf "\e]133;B\e\\")%}'
fi
```

### Fish Configuration

Add to `~/.config/fish/config.fish`:

```fish
# Zellij shell integration
if set -q ZELLIJ
    function __zellij_prompt_start --on-event fish_prompt
        printf '\e]133;A\e\\'
    end

    function __zellij_prompt_end --on-event fish_prompt
        printf '\e]133;B\e\\'
    end

    function __zellij_command_start --on-event fish_preexec
        printf '\e]133;C\e\\'
    end

    function __zellij_command_end --on-event fish_postexec
        printf '\e]133;D;%s\e\\' $status
    end
end
```

## Usage Example: Command Output Logger

Once implemented, you can create a logger that uses these markers:

```rust
// Example external tool or Zellij plugin
fn extract_commands_from_log(pane: &TerminalPane) {
    let markers = pane.get_command_markers();
    let content = pane.get_content();

    let mut commands = Vec::new();
    let mut current_command = None;

    for marker in markers {
        match marker.marker_type {
            CommandMarkerType::CommandExecutionStart => {
                current_command = Some(Command {
                    start_line: marker.line_number,
                    start_time: marker.timestamp,
                    output: String::new(),
                });
            },
            CommandMarkerType::CommandEnd => {
                if let Some(mut cmd) = current_command.take() {
                    cmd.end_line = marker.line_number;
                    cmd.end_time = marker.timestamp;
                    cmd.exit_code = marker.exit_code;

                    // Extract output between start_line and end_line
                    cmd.output = extract_lines(&content, cmd.start_line, cmd.end_line);

                    commands.push(cmd);
                }
            },
            _ => {},
        }
    }

    commands
}
```

## Development Workflow

Following TDD methodology, each cycle follows this pattern:

1. **🔴 RED**: Write a failing test
   - Run test: `cargo test <test_name>`
   - Confirm it fails with expected error

2. **🟢 GREEN**: Write minimal code to pass
   - Implement just enough to make test pass
   - Run test: `cargo test <test_name>`
   - Confirm it passes

3. **🔵 REFACTOR**: Clean up (if needed)
   - Extract common code
   - Improve readability
   - Run ALL tests: `cargo test test_osc_133`
   - Confirm nothing broke

4. **Commit** after each successful cycle
   ```bash
   git add .
   git commit -m "Add OSC 133 <feature> support"
   ```

## Implementation Checklist

- [ ] **Cycle 1**: Basic OSC 133;A support
  - [ ] RED: Write test for PromptStart marker
  - [ ] GREEN: Add minimal data structures and handler
  - [ ] Verify: `cargo test test_osc_133_prompt_start_marker`

- [ ] **Cycle 2**: All marker types
  - [ ] RED: Write tests for B, C, D, E markers
  - [ ] GREEN: Extend enums and handler
  - [ ] REFACTOR: Extract `record_command_marker()` helper
  - [ ] Verify: `cargo test test_osc_133`

- [ ] **Cycle 3**: Public API
  - [ ] RED: Write tests for getter methods
  - [ ] GREEN: Implement `get_command_markers()` and `get_markers_in_range()`
  - [ ] Verify: `cargo test test_get_command_markers`

- [ ] **Cycle 4**: Scrollback cleanup
  - [ ] RED: Write test for marker cleanup
  - [ ] GREEN: Implement `clear_markers_before_line()`
  - [ ] Verify: `cargo test test_clear_markers_before_line`

- [ ] **Cycle 5**: VTE integration
  - [ ] RED: Write end-to-end tests with VTE parser
  - [ ] GREEN: Verify existing implementation works
  - [ ] Verify: `cargo test test_osc_133_via_vte_parser`

- [ ] **Manual testing**: Run `test-osc133.sh` in Zellij

- [ ] **Final verification**: Run full test suite
  ```bash
  cargo test
  cargo build --release
  ```

## Benefits

- ✅ Non-invasive: No visible changes to terminal output
- ✅ Standards-compliant: Uses existing OSC 133 protocol
- ✅ Backward compatible: Older shells without integration continue to work
- ✅ Future-proof: Compatible with potential future Zellij shell integration features
- ✅ Rich metadata: Captures exit codes, timestamps, command text
- ✅ Efficient: Minimal memory overhead (only marker metadata, not duplicated content)

## Alternative: Custom OSC Code

If you want to avoid potential conflicts with future OSC 133 implementations, you could use a custom code like **OSC 9999**:

```rust
// In osc_dispatch(), use b"9999" instead of b"133"
b"9999" => {
    // Same implementation as OSC 133
}
```

Shell config would change to:
```bash
printf '\e]9999;C\e\\'  # instead of \e]133;C\e\\
```

This gives you complete control but loses standards compliance benefits.

## Questions to Consider

1. **Should markers be serialized** when sessions are saved/resurrected?
2. **Should markers be exposed** through the plugin API for custom tooling?
3. **Memory limits**: Should we cap the number of stored markers?
4. **Configuration**: Default on or off?

## Related Files to Review

- `zellij-server/src/panes/grid.rs` - Main implementation
- `zellij-server/src/panes/terminal_pane.rs` - API exposure
- `zellij-utils/src/input/options.rs` - Configuration
- `zellij-server/src/panes/unit/grid_tests.rs` - Testing

## Estimated Effort (TDD Approach)

- **Cycle 1** (Basic OSC 133;A): ~45 minutes
  - Write test: 10 min
  - Minimal implementation: 30 min
  - Verification: 5 min

- **Cycle 2** (All marker types): ~1.5 hours
  - Write tests: 20 min
  - Extend implementation: 45 min
  - Refactor: 20 min
  - Verification: 5 min

- **Cycle 3** (Public API): ~30 minutes
  - Write tests: 10 min
  - Implement getters: 15 min
  - Verification: 5 min

- **Cycle 4** (Scrollback cleanup): ~30 minutes
  - Write test: 10 min
  - Implement cleanup: 15 min
  - Verification: 5 min

- **Cycle 5** (VTE integration): ~30 minutes
  - Write integration tests: 15 min
  - Verify integration: 10 min
  - Debugging: 5 min

- **Manual testing**: ~30 minutes
- **Shell integration scripts**: ~30 minutes
- **Documentation**: ~30 minutes

**Total: ~5 hours of development time**

*Note: TDD often results in faster development despite writing tests first, because:*
- Less debugging time (tests catch issues immediately)
- Better design decisions (tests drive good API design)
- More confidence in changes (comprehensive test coverage)
- Easier refactoring (tests ensure nothing breaks)
