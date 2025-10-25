# Vim Passive Mode Implementation Summary

## Overview

Vim Passive Mode allows users to bind Vim text objects and motions to keyboard shortcuts without enabling full modal Vim editing. This provides access to Vim's powerful text manipulation capabilities while keeping the editor in its standard non-modal state.

## Current Status: WORKING ✅

The feature is now functional with the correct keybinding patterns.

---

## The Problem (SOLVED)

### Issue Description

When using text object selections in passive mode with keybindings like:

```json
"cmd-'": [
  "action::Sequence",
  [["vim::PushObject", { "around": false }], "vim::Quotes"]
]
```

**Problems encountered:**
1. ✗ Cursor would switch to block shape and not return to bar
2. ✗ Mode would transition but not switch back
3. ✗ The sync_vim_settings was skipping non-visual modes in passive mode

### Root Cause

The original implementation had `sync_vim_settings` skip all syncing for non-visual modes in passive mode. This meant:
- When switching to Visual mode → cursor updated to block ✓
- When switching back to Normal mode → cursor did NOT update back to bar ✗

The logic was:
```rust
if self.passive_mode && !self.mode.is_visual() {
    return; // Skip sync for non-visual modes
}
```

This prevented the cursor from updating when returning to Normal mode.

---

## The Solution

### 1. Fixed sync_vim_settings

**File:** `zed/crates/vim/src/vim.rs`

**Change:**
```rust
fn sync_vim_settings(&mut self, window: &mut Window, cx: &mut Context<Self>) {
    // In passive mode, we always sync cursor shape and selections to ensure
    // proper visual feedback when using text object keybindings
    if self.passive_mode {
        log::debug!(
            "[VIM] Passive mode: Syncing vim settings for mode {:?}",
            self.mode
        );
    }

    self.update_editor(cx, |vim, editor, cx| {
        editor.set_cursor_shape(vim.cursor_shape(cx), cx);
        editor.set_clip_at_line_ends(vim.clip_at_line_ends(), cx);
        editor.set_collapse_matches(true);
        editor.set_input_enabled(vim.editor_input_enabled());
        editor.set_autoindent(vim.should_autoindent());
        editor
            .selections
            .set_line_mode(matches!(vim.mode, Mode::VisualLine));

        let hide_edit_predictions = !matches!(vim.mode, Mode::Insert | Mode::Replace);
        editor.set_edit_predictions_hidden_for_vim_mode(hide_edit_predictions, window, cx);
    });
    cx.notify()
}
```

**Key change:** Removed the early return for non-visual modes. Now cursor shape and selections always sync in passive mode.

### 2. Correct Keybinding Pattern

Text object selections in passive mode require explicit mode switching:

```json
"cmd-'": [
  "action::Sequence",
  [
    "vim::SwitchToVisualMode",
    ["vim::PushObject", { "around": false }],
    "vim::Quotes",
    "vim::SwitchToNormalPreservingSelections"
  ]
]
```

**The four-step sequence:**
1. `vim::SwitchToVisualMode` - Enter visual mode
2. `["vim::PushObject", { "around": false }]` - Push the object operator
3. `vim::Quotes` - Execute the text object (makes the selection)
4. `vim::SwitchToNormalPreservingSelections` - Return to normal mode, keep selection

---

## How It Works

### Execution Flow

1. **User presses `cmd-'`**
   - Vim state: Normal mode, bar cursor

2. **`SwitchToVisualMode` executes**
   - Vim state: Visual mode
   - Cursor shape: Block (briefly)
   - `sync_vim_settings` is called → cursor updates to block

3. **`PushObject { around: false }` executes**
   - Operator stack: `[Operator::Object { around: false }]`
   - Mode: Still Visual

4. **`Quotes` executes**
   - Calls `visual_object` in `visual.rs`
   - Pops the operator from stack
   - Calculates text object range
   - Updates editor selections to highlight the text
   - Mode: Still Visual

5. **`SwitchToNormalPreservingSelections` executes**
   - Calls `switch_mode(Mode::Normal, true, window, cx)`
   - The `leave_selections: true` parameter preserves the selection
   - `sync_vim_settings` is called → cursor updates back to bar ✓
   - Mode: Normal

6. **Result**
   - Mode: Normal (non-modal)
   - Cursor: Bar shape ✓
   - Selection: Text inside quotes is highlighted ✓
   - User can immediately type to replace, copy, etc. ✓

---

## Supported Keybinding Patterns

### Pattern 1: Text Object Selection (Requires Mode Switching)

**Use case:** Select text inside/around delimiters

**Format:**
```json
"keybinding": [
  "action::Sequence",
  [
    "vim::SwitchToVisualMode",
    ["vim::PushObject", { "around": <true|false> }],
    "<TEXT_OBJECT>",
    "vim::SwitchToNormalPreservingSelections"
  ]
]
```

**Examples:**

```json
{
  "context": "Editor",
  "bindings": {
    // Select inside quotes
    "cmd-'": [
      "action::Sequence",
      [
        "vim::SwitchToVisualMode",
        ["vim::PushObject", { "around": false }],
        "vim::Quotes",
        "vim::SwitchToNormalPreservingSelections"
      ]
    ],

    // Select around quotes (including quotes)
    "cmd-shift-'": [
      "action::Sequence",
      [
        "vim::SwitchToVisualMode",
        ["vim::PushObject", { "around": true }],
        "vim::Quotes",
        "vim::SwitchToNormalPreservingSelections"
      ]
    ],

    // Select inside parentheses
    "cmd-9": [
      "action::Sequence",
      [
        "vim::SwitchToVisualMode",
        ["vim::PushObject", { "around": false }],
        "vim::Parentheses",
        "vim::SwitchToNormalPreservingSelections"
      ]
    ],

    // Select inside brackets
    "cmd-[": [
      "action::Sequence",
      [
        "vim::SwitchToVisualMode",
        ["vim::PushObject", { "around": false }],
        "vim::SquareBrackets",
        "vim::SwitchToNormalPreservingSelections"
      ]
    ],

    // Select inside curly braces
    "cmd-shift-[": [
      "action::Sequence",
      [
        "vim::SwitchToVisualMode",
        ["vim::PushObject", { "around": false }],
        "vim::CurlyBrackets",
        "vim::SwitchToNormalPreservingSelections"
      ]
    ],

    // Select inside word
    "cmd-w": [
      "action::Sequence",
      [
        "vim::SwitchToVisualMode",
        ["vim::PushObject", { "around": false }],
        "vim::Word",
        "vim::SwitchToNormalPreservingSelections"
      ]
    ],

    // Select around word (including whitespace)
    "cmd-shift-w": [
      "action::Sequence",
      [
        "vim::SwitchToVisualMode",
        ["vim::PushObject", { "around": true }],
        "vim::Word",
        "vim::SwitchToNormalPreservingSelections"
      ]
    ]
  }
}
```

### Pattern 2: Operator + Text Object (No Mode Switching Needed)

**Use case:** Delete, change, yank, comment text objects

**Format:**
```json
"keybinding": [
  "action::Sequence",
  [
    "<OPERATOR>",
    ["vim::PushObject", { "around": <true|false> }],
    "<TEXT_OBJECT>"
  ]
]
```

**Examples:**

```json
{
  "context": "Editor",
  "bindings": {
    // Delete inside quotes
    "cmd-d": [
      "action::Sequence",
      [
        "vim::PushDelete",
        ["vim::PushObject", { "around": false }],
        "vim::Quotes"
      ]
    ],

    // Change inside parentheses
    "cmd-c": [
      "action::Sequence",
      [
        "vim::PushChange",
        ["vim::PushObject", { "around": false }],
        "vim::Parentheses"
      ]
    ],

    // Yank inside word
    "cmd-y": [
      "action::Sequence",
      [
        "vim::PushYank",
        ["vim::PushObject", { "around": false }],
        "vim::Word"
      ]
    ],

    // Toggle comment inside quotes
    "cmd-/": [
      "action::Sequence",
      [
        "vim::PushToggleComments",
        ["vim::PushObject", { "around": false }],
        "vim::Quotes"
      ]
    ]
  }
}
```

---

## Available Text Objects

All standard Vim text objects are available:

### Paired Delimiters
- `vim::Quotes` - Single quotes `'text'`
- `vim::DoubleQuotes` - Double quotes `"text"`
- `vim::BackQuotes` - Backticks `` `text` ``
- `vim::Parentheses` - Parentheses `(text)`
- `vim::SquareBrackets` - Square brackets `[text]`
- `vim::CurlyBrackets` - Curly braces `{text}`
- `vim::AngleBrackets` - Angle brackets `<text>`
- `vim::VerticalBars` - Vertical bars `|text|`

### Word Objects
- `vim::Word` - Word (alphanumeric + underscore)
- `vim::WORD` - WORD (any non-whitespace)
- `vim::Sentence` - Sentence
- `vim::Paragraph` - Paragraph

### Code Objects
- `vim::Tag` - HTML/XML tag `<tag>content</tag>`
- `vim::Argument` - Function argument
- `vim::Comment` - Comment block

---

## Implementation Details

### Key Components

**1. Actions (`vim.rs`)**
- `SwitchToVisualMode` - Switches to visual mode
- `SwitchToNormalMode` - Switches to normal mode (clears selections)
- `SwitchToNormalPreservingSelections` - Switches to normal mode (keeps selections)
- `PushObject` - Pushes an Object operator onto the stack

**2. Mode Switching (`vim.rs::switch_mode`)**
- `leave_selections: bool` parameter controls whether selections are preserved
- `SwitchToNormalPreservingSelections` calls `switch_mode(Mode::Normal, true, ...)`
- Regular `SwitchToNormalMode` calls `switch_mode(Mode::Normal, false, ...)`

**3. Text Object Handling (`visual.rs::visual_object`)**
- Pops the Object operator from stack
- Calculates the range of the text object
- Updates editor selections to highlight the text
- Handles special cases (nested delimiters, empty objects, etc.)

**4. Sync Settings (`vim.rs::sync_vim_settings`)**
- Updates cursor shape based on current mode
- Updates editor settings (clip at line ends, input enabled, etc.)
- In passive mode: always syncs (fixed in our implementation)

---

## Testing

See `TESTING_GUIDE.md` for comprehensive test cases.

**Quick smoke test:**

1. Set `vim_passive_mode: true` and `vim_mode: false` in settings
2. Add this keybinding:
   ```json
   "cmd-'": [
     "action::Sequence",
     [
       "vim::SwitchToVisualMode",
       ["vim::PushObject", { "around": false }],
       "vim::Quotes",
       "vim::SwitchToNormalPreservingSelections"
     ]
   ]
   ```
3. Create a file with: `const x = "hello world";`
4. Place cursor inside the quotes
5. Press `cmd-'`
6. **Expected:** Text `hello world` is selected, cursor is bar shape
7. Type "goodbye" to replace
8. **Expected:** Result is `const x = "goodbye";`

---

## Design Decisions

### Why Explicit Mode Switching?

We chose to require explicit `SwitchToVisualMode` and `SwitchToNormalPreservingSelections` in keybindings rather than doing it automatically because:

1. **Transparency** - Users can see exactly what the keybinding does
2. **Flexibility** - Users can customize the behavior (e.g., stay in visual mode if desired)
3. **Simplicity** - No magic state management in the Vim implementation
4. **Debugging** - Easier to understand and debug issues
5. **Consistency** - Matches Vim's explicit mode model

### Why Not Make Text Objects Work Without Operators?

In traditional Vim, text objects require a preceding operator or visual mode (`diq`, `yaw`, `vib`, etc.). We maintain this constraint because:

1. **Architecture** - Text objects in the code expect an operator on the stack or visual mode
2. **Reusability** - Same code path for operator+object and visual+object patterns
3. **No duplication** - Avoids duplicating text object logic
4. **Vim semantics** - Preserves Vim's conceptual model

The trade-off is slightly more verbose keybindings, but it maintains code quality and correctness.

---

## Next Steps / Future Improvements

### 1. Template Keymaps (RECOMMENDED)

Create a default keymap template users can import:

```json
// keymap-templates/vim-passive-text-objects.json
{
  "context": "Editor",
  "bindings": {
    "cmd-'": ["action::Sequence", ["vim::SwitchToVisualMode", ["vim::PushObject", { "around": false }], "vim::Quotes", "vim::SwitchToNormalPreservingSelections"]],
    "cmd-shift-'": ["action::Sequence", ["vim::SwitchToVisualMode", ["vim::PushObject", { "around": true }], "vim::Quotes", "vim::SwitchToNormalPreservingSelections"]],
    "cmd-9": ["action::Sequence", ["vim::SwitchToVisualMode", ["vim::PushObject", { "around": false }], "vim::Parentheses", "vim::SwitchToNormalPreservingSelections"]],
    "cmd-shift-9": ["action::Sequence", ["vim::SwitchToVisualMode", ["vim::PushObject", { "around": true }], "vim::Parentheses", "vim::SwitchToNormalPreservingSelections"]],
    "cmd-[": ["action::Sequence", ["vim::SwitchToVisualMode", ["vim::PushObject", { "around": false }], "vim::SquareBrackets", "vim::SwitchToNormalPreservingSelections"]],
    "cmd-shift-[": ["action::Sequence", ["vim::SwitchToVisualMode", ["vim::PushObject", { "around": true }], "vim::SquareBrackets", "vim::SwitchToNormalPreservingSelections"]],
    "cmd-w": ["action::Sequence", ["vim::SwitchToVisualMode", ["vim::PushObject", { "around": false }], "vim::Word", "vim::SwitchToNormalPreservingSelections"]],
    "cmd-shift-w": ["action::Sequence", ["vim::SwitchToVisualMode", ["vim::PushObject", { "around": true }], "vim::Word", "vim::SwitchToNormalPreservingSelections"]]
  }
}
```

### 2. UI Improvements (OPTIONAL)

- Add a status bar indicator when passive mode is enabled
- Show a brief tooltip when first enabling passive mode
- Add "Vim Passive Mode" section to keymap panel with suggested bindings

### 3. Documentation (NEEDED)

- Add section to main Vim documentation
- Create a "Getting Started with Passive Mode" guide
- Add examples to the default keymap comments

### 4. Additional Testing (RECOMMENDED)

Create automated tests for:
- Text object selection in passive mode
- Cursor shape transitions
- Multiple cursors with text objects
- Nested text objects
- Edge cases (empty objects, overlapping delimiters)

### 5. Performance Monitoring (OPTIONAL)

- Verify no performance regression with mode switching
- Ensure no memory leaks from repeated mode transitions
- Profile the sync_vim_settings overhead

---

## Conclusion

The Vim Passive Mode feature is now **fully functional** with the correct keybinding patterns. The fix to `sync_vim_settings` ensures cursor shape properly updates when switching between modes, providing a seamless non-modal experience with Vim's powerful text objects.

**Key takeaway:** Text object selections in passive mode require explicit mode switching in keybindings. This is intentional and provides a clean, predictable implementation that doesn't alter Vim's core behavior.

**Status: Ready for use** ✅

Users can now bind Vim text objects to any keybinding and enjoy non-modal Vim power tools in Zed!
