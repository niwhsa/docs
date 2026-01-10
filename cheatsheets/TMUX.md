# tmux Cheatsheet

Keyboard-driven terminal multiplexer workflow for Neovim users.

## Prefix Key

**`Ctrl-Space`** (with Caps Lock → Control, it's `CapsLock + Space`)

All commands below: press prefix, release, then press the key.

## Sessions

| Keys | Action |
|------|--------|
| `tmux new -s name` | Create named session |
| `tmux ls` | List sessions |
| `tmux attach -t name` | Attach to session |
| `Ctrl-Space d` | Detach from session |
| `Ctrl-Space s` | Session picker |
| `Ctrl-Space )` | Next session |
| `Ctrl-Space (` | Previous session |
| `Ctrl-Space $` | Rename session |
| `Ctrl-Space :new -s name` | New session (from inside tmux) |

**When to use sessions:** Different projects (one session per project)

## Windows (Tabs)

| Keys | Action |
|------|--------|
| `Ctrl-Space c` | New window |
| `Ctrl-Space 1-9` | Jump to window |
| `Ctrl-Space n` | Next window |
| `Ctrl-Space p` | Previous window |
| `Ctrl-Space w` | Window picker |
| `Ctrl-Space ,` | Rename window |
| `Ctrl-Space &` | Close window |

**When to use windows:** Different tasks in same project (code, git, logs)

## Panes (Splits)

| Keys | Action |
|------|--------|
| `Ctrl-Space \|` | Split vertical (left/right) |
| `Ctrl-Space -` | Split horizontal (top/bottom) |
| `Ctrl-Space x` | Close pane |
| `Ctrl-Space z` | Zoom/unzoom pane |

**When to use panes:** Need to see both simultaneously (code + output)

## Navigation (No Prefix!)

| Keys | Action |
|------|--------|
| `Ctrl-h` | Move left |
| `Ctrl-j` | Move down |
| `Ctrl-k` | Move up |
| `Ctrl-l` | Move right |

Works seamlessly between tmux panes AND Neovim splits!

## Resize Panes

| Keys | Action |
|------|--------|
| `Ctrl-Space H` | Shrink left |
| `Ctrl-Space J` | Grow down |
| `Ctrl-Space K` | Shrink up |
| `Ctrl-Space L` | Grow right |

Note: Uppercase (Shift) = resize. Repeatable: `H H H`

## Common Workflows

### Start of day
```bash
tmux new -s project
Ctrl-Space |              # Split for terminal
nvim .                    # Code on left
```

### Code → Test cycle
```
Ctrl-h                    # Edit in nvim
:w                        # Save
Ctrl-l                    # Go to terminal
cargo run                 # Or: cargo watch -x run
Ctrl-h                    # Back to code
```

### End of day
```
Ctrl-Space d              # Detach (session persists!)
```

### Next day
```bash
tmux attach -t project    # Exactly where you left off
```

## Kill Sessions

```bash
tmux kill-session -t name     # Kill specific session
tmux kill-session -a          # Kill all except current
tmux kill-server              # Kill everything
```

## Hierarchy Mental Model

```
Sessions  = Browser windows (different sites/projects)
Windows   = Tabs (different tasks in same project)
Panes     = Split screen (see both at once)
```

## Typical Layout for Rust/Code

```
┌─────────────────────┬──────────────┐
│                     │              │
│       nvim          │  cargo run   │
│     (70-80%)        │   (20-30%)   │
│                     │              │
└─────────────────────┴──────────────┘
```

### Auto-Run on Save Setup

```bash
# 1. Create split layout
tmux new -s rust
Ctrl-Space |              # Split vertical

# 2. Left pane: open code
nvim src/main.rs

# 3. Right pane: start cargo watch
Ctrl-l                    # Go right
cargo watch -c -x run     # Auto-run on save (-c clears screen)

# 4. Code!
Ctrl-h                    # Back to nvim
# Edit, then :w to save → right pane auto-runs!
```

### cargo watch Options

```bash
cargo watch -x run              # Run on save
cargo watch -x test             # Test on save  
cargo watch -x check            # Compile check only (fastest)
cargo watch -c -x run           # Clear screen + run
cargo watch -x 'test -- --nocapture'  # Tests with output
```

**Install:** `cargo install cargo-watch`

