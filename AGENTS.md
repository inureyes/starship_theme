# Project notes for Claude

## Handling files with Nerd Font / PUA glyphs

`starship.toml` contains many Nerd Font glyphs in the Private Use Area
(U+E000–U+F8FF, U+F0000+). These may not render in the assistant's
display, which makes them look like empty strings — but the bytes are
real and load-bearing.

**Rule: never rewrite these files line-by-line or via full-file `Write`.**
If a glyph doesn't render in your view, retyping the line will silently
drop the character.

Instead, edit non-invasively:

- Prefer the `Edit` tool with `old_string` anchored on surrounding ASCII
  context, so the glyph passes through untouched.
- For bulk changes, write a Python/sed script that reads the existing
  bytes (or an upstream source like the starship preset at
  `https://starship.rs/presets/toml/gruvbox-rainbow.toml`) and patches
  the file in place — so glyphs are never round-tripped through the
  assistant's text output.
- After any edit touching these files, verify with a codepoint dump
  (e.g. `python3 -c "...ord(c)..."`) that no symbol slot became empty.

This applies to `starship.toml` today and to any future file in this
repo that embeds Nerd Font or other PUA characters.

## Powerline segment color chaining

The prompt is a chain of colored segments joined by powerline separator
glyphs (U+E0B0 right triangle, U+E0B6 left rounded cap, U+E0B4 right
rounded cap). Each separator sits **between** two segments and must
bridge their colors so the transition is seamless — no gap, no mismatch.

The rule for an inter-segment separator:

- `fg` = background color of the segment **on the separator's pointed
  side** (the color being "cut into")
- `bg` = background color of the segment **on the separator's flat side**
  (the color behind the separator)

For the right-triangle `` the point faces right, so:

```
[](fg:<prev_segment_bg> bg:<next_segment_bg>)
```

For the trailing cap after the last segment (nothing follows it), only
`fg` is set to the last segment's background — `bg` is omitted so it
blends into the terminal:

```
[ ](fg:<last_segment_bg>)
```

The opening cap (if used) mirrors this: `fg` = first segment's bg, no
`bg`.

**Consequences when editing:**

- If you reorder, insert, or remove a segment, every adjacent separator's
  `fg`/`bg` pair must be updated to match the new neighbors. A single
  stale reference breaks the visual chain (you'll see a hard edge or a
  wrong-colored wedge).
- If you rename a palette color, update both the segment's own style
  **and** every separator that references it.
- Segment modules themselves must set `bg:<their_own_bg>` in their
  `style`/`format` so the segment body matches what the separators
  expect on either side.
- When adding per-host palettes, ensure every `color_bg_*` key used by a
  separator exists in the new palette — missing keys cause starship to
  fall back to default colors and shatter the chain.

Before committing any layout change, mentally (or literally) walk the
format string left-to-right and confirm each separator's `fg`/`bg`
matches the backgrounds of its immediate neighbors.
