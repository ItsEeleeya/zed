# Vim Passive Mode — Testing Guide

This document outlines a minimal set of tests for Vim Passive Mode. It exists as a placeholder referenced by other docs and should be expanded as the implementation evolves.

Goal:
- Verify that passive mode uses the existing Vim actions (motions, text objects, operators) without duplicating logic.
- Validate that explicit mode switching via keybindings yields correct editor state (selections, cursor shape, input, no mode indicator).

## Scope

Covered:
- Text object selections using explicit mode switching (Visual → PushObject → Object → NormalPreservingSelections)
- Operator + object sequences (no explicit mode switching)
- Motions bound directly
- Multi-cursor behavior
- Cursor shape transitions and selection persistence
- Non-invasive UX (no mode indicator, no subscriptions intercepting unrelated input)

Not covered (yet):
- Exhaustive per-object edge cases
- Performance profiling
- Telemetry

## Quick Smoke Test

1) Setup
- Settings: set `"vim_passive_mode": true` and `"vim_mode": false`.
- Add a selection keybinding (select inside quotes):

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

- Add an operator + object keybinding (toggle comments inside quotes):

```json
{
  "context": "Editor",
  "bindings": {
    "cmd-/ cmd-'": [
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

2) Select inside quotes
- Text: const x = "hello world";
- Place cursor anywhere inside hello world.
- Press cmd-'.
- Expect: “hello world” selected; cursor shape returns to bar; mode indicator hidden; typing replaces selection.

3) Operator + object
- Using the same text, press cmd-/ followed by cmd-'.
- Expect: comment toggled for the “inside quotes” range (operator applied to object); cursor remains a bar afterwards.

4) Motions-only binding (optional)
- Bind a simple motion (e.g., w) to a convenient key and verify navigation works without switching modes.

5) Multiple cursors
- Create 2–3 cursors inside different quoted strings.
- Press cmd-'.
- Expect: selections created independently under all cursors and cursor shape returns to bar.

6) Nested/ambiguous delimiters
- Example: foo("a 'b' c")
- Place cursor inside the inner 'b'.
- Press cmd-'.
- Expect: selects the inner quotes content. Verify MiniQuotes/AnyQuotes variants as desired.

7) Visual line targets
- Bind selection around parentheses using “around: true” and Parentheses.
- Place cursor inside (a line-spanning) parentheses range.
- Expect: selection target mode matches the object’s target (sometimes VisualLine), selection persists, cursor returns to bar.

## Manual Test Matrix

Selections (4-step sequence):
- Quotes: inner/around
- DoubleQuotes/BackQuotes/AnyQuotes/MiniQuotes
- Parentheses/SquareBrackets/CurlyBrackets/AngleBrackets
- AnyBrackets/MiniBrackets
- Word/Subword (requires ignore_punctuation param)
- Sentence/Paragraph (note target visual mode behavior)
- Tag/Argument/Method/Class/Comment/EntireFile
- IndentObj (requires include_below param)

Operators (operator → PushObject → Object):
- Delete (di", da(), etc.)
- Change (ci", ca(), etc.)
- Yank (yi", ya(), etc.)
- ToggleComments (gc i", gc a(), etc.)
- Indent/Outdent/AutoIndent
- Case conversions (upper/lower/opposite)
- Replace/ReplaceWithRegister

Motions:
- A representative subset (Left/Right/Up/Down, WordForward/Backward, Start/EndOfLine, FirstNonWhitespace)
- Motion + operator flows (e.g., PushDelete + CurrentLine)

Behavioral checks:
- Cursor shape transitions: brief block in Visual, bar in Normal after switching back
- Selection persists after SwitchToNormalPreservingSelections
- No mode indicator is shown in passive mode
- Editor input remains enabled for normal typing after actions
- Selection history behaves reasonably (no runaway entries for transient operations)

## Regression Checklist

- Non-passive users unaffected (passive is opt-in; no global key interception without bindings)
- Operator stack cleared appropriately across actions
- Counts still work where expected
- VisualBlock edge cases do not regress normal passive flows
- Edit prediction visibility behaves as expected across temporary mode changes

## Automated Testing (Initial Plan)

Add focused tests under the Vim crate to validate core invariants:
- Text object selection:
  - Setup an editor buffer and Vim instance with passive mode
  - Simulate the 4-step sequence programmatically:
    - SwitchToVisualMode
    - PushObject { around: X }
    - Object (e.g., Quotes)
    - SwitchToNormalPreservingSelections
  - Assert:
    - Final mode is Normal
    - Selections match expected ranges
- Operator + object:
  - PushDelete/PushChange/PushYank + PushObject + Object
  - Verify resulting buffer content and selection/cursor state
- Multi-cursor:
  - Create multiple selections, run the same flows, verify independent outcomes
- Cursor shape:
  - Ensure sync path sets block in Visual and returns to bar in Normal
  - Where direct cursor-shape inspection is difficult, assert the state transitions and that sync is invoked across mode switches

Note:
- Use existing GPUI test harness utilities where available (there are examples of #[gpui::test] in the codebase). Keep tests deterministic and avoid relying on timers.

## Troubleshooting

- “Action does nothing”
  - For selections: ensure the 4-step sequence is used (Visual → PushObject → Object → NormalPreservingSelections)
  - For edits: ensure operator precedes PushObject + object
  - Verify boolean params are real booleans, not strings
  - Ensure objects like Word/Subword include required params (ignore_punctuation)

- “Cursor remains block”
  - Confirm SwitchToNormalPreservingSelections is the final step
  - Confirm passive mode always syncs in the cursor settings path

- “JSON error about types”
  - Remember that each action is either a string or a two-element array [name, { params }], and each action is its own element in the Sequence array

## References

- Passive Mode overview and examples: docs/vim/llm/PASSIVE_MODE.md
- Implementation summary and rationale: docs/vim/llm/PASSIVE_MODE_IMPLEMENTATION.md
- Example keymap snippet: docs/vim/llm/PASSIVE_MODE_KEYMAP_EXAMPLE.json
- Core code paths:
  - crates/vim/src/vim.rs (mode switching, syncing)
  - crates/vim/src/object.rs (text objects)
  - crates/vim/src/visual.rs (visual selections)
  - crates/vim/src/normal.rs (operator + object workflows)

---

This is a minimal, living guide to prevent broken references and to establish a consistent baseline for testing Vim Passive Mode. Please extend it with concrete test cases, fixtures, and automation as the feature matures.
