# Implementation Plan: Command Output Separation in Zellij

## Overview

**Goal**: Add support for command output separators to enable logging terminal output with clear boundaries between commands.

**Approach**: Implement OSC 133 (Shell Integration Protocol) support in Zellij's terminal emulator to mark command boundaries invisibly.

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

## Implementation Steps

### Step 1: Define Data Structures

**File**: `zellij-server/src/panes/grid.rs`

Add a new module for command markers:

```rust
// Around line 40, after existing imports
use std::time::SystemTime;

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum CommandMarkerType {
    PromptStart,           // OSC 133;A
    PromptEnd,             // OSC 133;B
    CommandExecutionStart, // OSC 133;C
    CommandEnd,            // OSC 133;D
    CommandLine,           // OSC 133;E
}

#[derive(Debug, Clone)]
pub struct CommandMarker {
    pub marker_type: CommandMarkerType,
    pub line_number: usize,        // Line in viewport + scrollback
    pub column_number: usize,      // Column position
    pub exit_code: Option<i32>,    // For CommandEnd markers
    pub command_text: Option<String>, // For CommandLine markers
    pub timestamp: u64,            // Unix timestamp in milliseconds
}
```

### Step 2: Add Marker Storage to Grid

**File**: `zellij-server/src/panes/grid.rs`

In the `Grid` struct (around line 314), add:

```rust
pub struct Grid {
    // ... existing fields ...
    pub command_markers: Vec<CommandMarker>,
}
```

In `Grid::new()` (around line 503), initialize:

```rust
Grid {
    // ... existing fields ...
    command_markers: Vec::new(),
}
```

### Step 3: Implement OSC 133 Handler

**File**: `zellij-server/src/panes/grid.rs`

In `impl Grid`, add helper methods (around line 2470, before `impl Perform`):

```rust
impl Grid {
    // ... existing methods ...

    fn record_command_marker(&mut self, marker_type: CommandMarkerType, params: &[&[u8]]) {
        let line_number = self.cursor.y + self.lines_above.len();
        let column_number = self.cursor.x;

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

        let marker = CommandMarker {
            marker_type,
            line_number,
            column_number,
            exit_code,
            command_text,
            timestamp,
        };

        self.command_markers.push(marker);

        if self.debug {
            log::info!("Command marker: {:?}", marker);
        }
    }
}
```

### Step 4: Add OSC 133 Dispatch Handler

**File**: `zellij-server/src/panes/grid.rs`

In `osc_dispatch()` method (around line 2575), add new case before the default case:

```rust
fn osc_dispatch(&mut self, params: &[&[u8]], bell_terminated: bool) {
    // ... existing code ...

    match params[0] {
        // ... existing cases (b"0", b"2", b"4", b"8", etc.) ...

        // OSC 133 - Shell Integration / Command Markers
        b"133" => {
            if params.len() >= 2 {
                match params[1] {
                    b"A" => {
                        self.record_command_marker(CommandMarkerType::PromptStart, params);
                    },
                    b"B" => {
                        self.record_command_marker(CommandMarkerType::PromptEnd, params);
                    },
                    b"C" => {
                        self.record_command_marker(CommandMarkerType::CommandExecutionStart, params);
                    },
                    b"D" => {
                        self.record_command_marker(CommandMarkerType::CommandEnd, params);
                    },
                    b"E" => {
                        self.record_command_marker(CommandMarkerType::CommandLine, params);
                    },
                    _ => {
                        if self.debug {
                            log::warn!("Unknown OSC 133 subcommand: {:?}", params[1]);
                        }
                    },
                }
            }
        },

        _ => {
            if self.debug {
                log::warn!("Unhandled osc: {:?}", params);
            }
        },
    }
}
```

### Step 5: Add Public API to Access Markers

**File**: `zellij-server/src/panes/grid.rs`

Add getter methods in `impl Grid`:

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

    pub fn clear_markers_before_line(&mut self, line: usize) {
        self.command_markers.retain(|m| m.line_number >= line);
    }
}
```

### Step 6: Handle Scrollback Cleanup

**File**: `zellij-server/src/panes/grid.rs`

When scrollback is trimmed, clean up old markers. Find where `lines_above` is modified (search for `lines_above.drain` or similar) and add marker cleanup.

Example location might be in a method that manages scrollback buffer. Add:

```rust
// After lines are removed from scrollback
let removed_lines = number_of_lines_removed;
self.command_markers.retain(|m| m.line_number >= removed_lines);
for marker in &mut self.command_markers {
    marker.line_number -= removed_lines;
}
```

### Step 7: Expose Markers Through Terminal Pane API

**File**: `zellij-server/src/panes/terminal_pane.rs`

Add methods to expose markers:

```rust
impl TerminalPane {
    // Add these methods (find a good location in the impl block)

    pub fn get_command_markers(&self) -> &[CommandMarker] {
        self.grid.get_command_markers()
    }

    pub fn get_markers_in_viewport(&self) -> Vec<&CommandMarker> {
        let viewport_start = self.grid.lines_above.len();
        let viewport_end = viewport_start + self.grid.height;
        self.grid.get_markers_in_range(viewport_start, viewport_end)
    }
}
```

### Step 8: Add Configuration Option (Optional)

**File**: `zellij-utils/src/input/options.rs`

Add a configuration option to enable/disable marker tracking:

```rust
pub struct Options {
    // ... existing fields ...

    /// Enable shell integration markers (OSC 133)
    #[clap(long, value_parser)]
    #[serde(default)]
    pub shell_integration: Option<bool>,
}
```

**File**: `zellij-server/src/panes/grid.rs`

Add field to Grid:

```rust
pub struct Grid {
    // ... existing fields ...
    shell_integration_enabled: bool,
    pub command_markers: Vec<CommandMarker>,
}
```

Update `record_command_marker` to check the flag:

```rust
fn record_command_marker(&mut self, marker_type: CommandMarkerType, params: &[&[u8]]) {
    if !self.shell_integration_enabled {
        return;
    }
    // ... rest of implementation ...
}
```

## Testing Strategy

### Unit Tests

**File**: `zellij-server/src/panes/unit/grid_tests.rs`

Add tests for OSC 133 handling:

```rust
#[test]
fn test_osc_133_prompt_markers() {
    let mut grid = create_test_grid();

    // Send OSC 133;A (prompt start)
    let params = vec![b"133".as_ref(), b"A".as_ref()];
    grid.osc_dispatch(&params, true);

    assert_eq!(grid.command_markers.len(), 1);
    assert_eq!(grid.command_markers[0].marker_type, CommandMarkerType::PromptStart);
}

#[test]
fn test_osc_133_command_end_with_exit_code() {
    let mut grid = create_test_grid();

    // Send OSC 133;D;127 (command end with exit code 127)
    let params = vec![b"133".as_ref(), b"D".as_ref(), b"127".as_ref()];
    grid.osc_dispatch(&params, true);

    assert_eq!(grid.command_markers.len(), 1);
    assert_eq!(grid.command_markers[0].marker_type, CommandMarkerType::CommandEnd);
    assert_eq!(grid.command_markers[0].exit_code, Some(127));
}

#[test]
fn test_osc_133_command_line_text() {
    let mut grid = create_test_grid();

    // Send OSC 133;E;ls -la
    let params = vec![b"133".as_ref(), b"E".as_ref(), b"ls -la".as_ref()];
    grid.osc_dispatch(&params, true);

    assert_eq!(grid.command_markers.len(), 1);
    assert_eq!(grid.command_markers[0].marker_type, CommandMarkerType::CommandLine);
    assert_eq!(grid.command_markers[0].command_text, Some("ls -la".to_string()));
}
```

### Integration Testing

Create a test script to emit OSC 133 sequences:

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
echo "Test complete"
```

Run in Zellij with debug mode:
```bash
zellij --debug
# In pane, run:
./test-osc133.sh
# Check logs for "Command marker:" messages
```

## Shell Integration Setup

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

## Rollout Plan

1. **Phase 1**: Implement core functionality (Steps 1-5)
2. **Phase 2**: Add tests (Step 8 - Testing Strategy)
3. **Phase 3**: Add configuration option (Step 8 - Optional)
4. **Phase 4**: Document in Zellij docs
5. **Phase 5**: Create shell integration scripts for common shells
6. **Phase 6**: Submit PR to Zellij project

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

## Estimated Effort

- Core implementation: ~4 hours
- Testing: ~2 hours
- Shell integration scripts: ~1 hour
- Documentation: ~1 hour

**Total: ~8 hours of development time**
