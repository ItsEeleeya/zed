# Vim Passive Mode

Vim Passive Mode lets you use Vim’s motions, text objects, and operators via keybindings without enabling full modal editing. The editor stays non‑modal, so you can type immediately without switching modes.

## Overview

In passive mode:

- The editor stays in normal editing mode (no persistent Normal/Insert/Visual modes)
- The cursor remains a bar (not a block) except briefly during explicit visual operations
- No mode indicator is shown
- You can bind keys to Vim actions (operators, motions, text objects)
- Vim’s operator + motion/object pattern works the same as in full Vim mode

## Enabling Passive Mode

1. Via Settings UI: Settings → enable “Vim Passive Mode”
2. Via settings.json: add `"vim_passive_mode": true`

Note: You should have `"vim_mode": false` when using passive mode.

## How It Works

Vim’s power comes from its operator + motion/object composition. In passive mode, this works the same way:

- Operator + motion/object: push an operator (e.g., `vim::PushToggleComments`) then a motion or text object (e.g., `vim::CurrentLine` or `vim::Quotes`). The operator applies to the resulting range.
- Selection with text objects (no operator): requires an explicit 4‑step sequence in your keybinding:
  1. `vim::SwitchToVisualMode`
  2. `["vim::PushObject", { "around": <true|false> }]`
  3. `<TEXT_OBJECT>` (e.g., `vim::Quotes`)
  4. `vim::SwitchToNormalPreservingSelections`

This explicit sequence avoids duplicating logic and cleanly reuses the existing Vim implementation.

## Action Sequence Syntax

To combine multiple actions in a single keybinding, use `action::Sequence`:

```json
{
  "bindings": {
    "key": [
      "action::Sequence",
      [
        "first_action",
        ["second_action", { "parameter": "value" }],
        "third_action"
      ]
    ]
  }
}
```

### Important Rules

1. Each action in the sequence array must be:
   - A string: `"action_name"` (for actions without parameters)
   - A two-element array: `["action_name", { "param": value }]` (for actions with parameters)

2. INCORRECT ❌:

   ```json
   ["action::Sequence", ["vim::PushObject", { "around": false }, "vim::Quotes"]]
   ```

   This passes three separate items in one element which is invalid.

3. CORRECT ✅:
   ```json
   [
     "action::Sequence",
     [["vim::PushObject", { "around": false }], "vim::Quotes"]
   ]
   ```
   Each action is its own element in the array.

## Common Use Cases

### Commenting

#### Comment the current line (operator + motion)

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-/": [
      "action::Sequence",
      ["vim::PushToggleComments", "vim::CurrentLine"]
    ]
  }
}
```

#### Select inside quotes (inner, selection pattern: 4 steps)

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-'": [
      "action::Sequence",
      [
        "vim::SwitchToVisualMode",
        ["vim::PushObject", { "around": false }],
        "vim::Quotes",
        "vim::SwitchToNormalPreservingSelections"
      ]
    ]
  }
}
```

#### Comment inside quotes (inner, operator + object)

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-shift-/": [
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

#### Comment around quotes (including quotes, operator + object)

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-alt-/": [
      "action::Sequence",
      [
        "vim::PushToggleComments",
        ["vim::PushObject", { "around": true }],
        "vim::Quotes"
      ]
    ]
  }
}
```

### Deleting Text

#### Delete inside quotes (operator + object)

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-'": [
      "action::Sequence",
      [
        "vim::PushDelete",
        ["vim::PushObject", { "around": false }],
        "vim::Quotes"
      ]
    ]
  }
}
```

#### Delete around parentheses (operator + object)

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-9": [
      "action::Sequence",
      [
        "vim::PushDelete",
        ["vim::PushObject", { "around": true }],
        "vim::Parentheses"
      ]
    ]
  }
}
```

### Selecting Text

Selections without an operator require the explicit 4‑step sequence:

- `vim::SwitchToVisualMode`
- `["vim::PushObject", { "around": <true|false> }]`
- `<TEXT_OBJECT>`
- `vim::SwitchToNormalPreservingSelections`

#### Select inside quotes (inner)

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-'": [
      "action::Sequence",
      [
        "vim::SwitchToVisualMode",
        ["vim::PushObject", { "around": false }],
        "vim::Quotes",
        "vim::SwitchToNormalPreservingSelections"
      ]
    ]
  }
}
```

#### Select inside brackets (inner)

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-[": [
      "action::Sequence",
      [
        "vim::SwitchToVisualMode",
        ["vim::PushObject", { "around": false }],
        "vim::SquareBrackets",
        "vim::SwitchToNormalPreservingSelections"
      ]
    ]
  }
}
```

#### Select around parentheses (including parentheses)

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-9": [
      "action::Sequence",
      [
        "vim::SwitchToVisualMode",
        ["vim::PushObject", { "around": true }],
        "vim::Parentheses",
        "vim::SwitchToNormalPreservingSelections"
      ]
    ]
  }
}
```

Note: The cursor may briefly flash to block style during selection—this is expected while the action executes. The selection persists after returning to normal mode.

### Changing Text (Delete + Insert Mode)

#### Change inside word (operator + object)

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-w": [
      "action::Sequence",
      [
        "vim::PushChange",
        ["vim::PushObject", { "around": false }],
        ["vim::Word", { "ignore_punctuation": false }]
      ]
    ]
  }
}
```

### Yanking (Copying)

#### Yank inside curly brackets (operator + object)

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-shift-c": [
      "action::Sequence",
      [
        "vim::PushYank",
        ["vim::PushObject", { "around": false }],
        "vim::CurlyBrackets"
      ]
    ]
  }
}
```

## Available Operators

These actions push operators onto the stack:

- `vim::PushDelete` — Delete text
- `vim::PushChange` — Delete text and enter insert mode
- `vim::PushYank` — Copy text
- `vim::PushToggleComments` — Toggle comments
- `vim::PushIndent` — Indent text
- `vim::PushOutdent` — Outdent text
- `vim::PushUppercase` — Convert to uppercase
- `vim::PushLowercase` — Convert to lowercase
- `vim::PushOppositeCase` — Toggle case
- `vim::PushReplace` — Replace with a character
- `vim::PushReplaceWithRegister` — Replace with register contents

## Available Text Objects

Use `vim::PushObject` with `{ "around": false }` for “inner” or `{ "around": true }` for “around”:

- `vim::Word` — Word (requires `ignore_punctuation` parameter)
- `vim::Subword` — Subword (requires `ignore_punctuation` parameter)
- `vim::Quotes` — Single quotes `'...'`
- `vim::DoubleQuotes` — Double quotes `"..."`
- `vim::BackQuotes` — Backticks `` `...` ``
- `vim::AnyQuotes` — Any quotes
- `vim::MiniQuotes` — Nearest quotes
- `vim::Parentheses` — Parentheses `(...)`
- `vim::SquareBrackets` — Square brackets `[...]`
- `vim::CurlyBrackets` — Curly brackets `{...}`
- `vim::AngleBrackets` — Angle brackets `<...>`
- `vim::AnyBrackets` — Any brackets
- `vim::MiniBrackets` — Nearest brackets
- `vim::VerticalBars` — Vertical bars `|...|`
- `vim::Tag` — HTML/XML tag
- `vim::Argument` — Function argument
- `vim::Sentence` — Sentence
- `vim::Paragraph` — Paragraph
- `vim::Comment` — Comment block
- `vim::Method` — Method/function
- `vim::Class` — Class definition
- `vim::EntireFile` — Entire file
- `vim::IndentObj` — Same indentation level (requires `include_below` parameter)

## Available Motions

These can be used without pushing an object:

- `vim::CurrentLine` — The current line
- `vim::Left` — One character left
- `vim::Right` — One character right
- `vim::Up` — One line up
- `vim::Down` — One line down
- `vim::WordForward` — Forward to next word
- `vim::WordBackward` — Backward to previous word
- `vim::EndOfLine` — End of line
- `vim::StartOfLine` — Start of line
- `vim::FirstNonWhitespace` — First non‑whitespace character

And many more—see `crates/vim/src/motion.rs` for the full list.

## Troubleshooting

### “Incorrect type. Expected string”

This error occurs when action parameters are not properly formatted. Make sure:

1. Actions with parameters are in a two‑element array: `["action", { "param": value }]`
2. Boolean values are not quoted: `{ "around": false }` not `{ "around": "false" }`
3. Each action in a sequence is its own element in the array

### Action doesn’t do anything

Check that:

1. For operations (delete, change, yank, comment): the operator comes before the motion/object
2. For selections: use the 4‑step sequence (Visual → PushObject → TextObject → NormalPreservingSelections)
3. You’re using `vim::PushObject` when working with text objects
4. The action exists (check the lists above)

### How do I know what parameters an action takes?

Look at the action definition in the source code:

- Text object actions: `crates/vim/src/object.rs`
- Motion actions: `crates/vim/src/motion.rs`
- Operator actions: `crates/vim/src/vim.rs`

Or check the JSON schema in the action’s `#[derive]` attributes.

## Advanced Examples

### Delete everything inside the nearest quotes (operator + object)

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-alt-d": [
      "action::Sequence",
      [
        "vim::PushDelete",
        ["vim::PushObject", { "around": false }],
        "vim::MiniQuotes"
      ]
    ]
  }
}
```

### Change entire function/method (operator + object)

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-shift-m": [
      "action::Sequence",
      [
        "vim::PushChange",
        ["vim::PushObject", { "around": true }],
        "vim::Method"
      ]
    ]
  }
}
```

### Select entire file and toggle comments (operator + object)

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-shift-a": [
      "action::Sequence",
      [
        "vim::PushToggleComments",
        ["vim::PushObject", { "around": false }],
        "vim::EntireFile"
      ]
    ]
  }
}
```

## How Selections Work in Passive Mode

Selections triggered by keybindings explicitly:

1. Switch to visual mode (`vim::SwitchToVisualMode`)
2. Execute the text object (`["vim::PushObject", { "around": <true|false> }]` + `<TEXT_OBJECT>`)
3. Switch back to normal mode while preserving the selection (`vim::SwitchToNormalPreservingSelections`)

This runs very quickly; you may notice a brief block cursor while the action executes. The selection remains visible and active, and you can immediately type to replace it, copy it, or run other actions.

## Why Not Full Vim Mode?

Passive mode is ideal if you:

- Want Vim’s text objects without modal editing
- Primarily use standard editor keybindings
- Only need specific Vim motions/objects for certain operations
- Find modal editing disruptive
- Want to learn Vim concepts gradually

If you want the full Vim experience with modes, use `"vim_mode": true` instead.

## Combining with Other Keybindings

Passive mode keybindings work alongside all standard Zed keybindings. You can mix and match:

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-/": [
      "action::Sequence",
      ["vim::PushToggleComments", "vim::CurrentLine"]
    ],
    "cmd-d": "editor::SelectNext",
    "cmd-shift-k": "editor::DeleteLine"
  }
}
```

This gives you the best of both worlds.
