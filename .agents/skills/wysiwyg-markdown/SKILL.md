# WYSIWYG Markdown Editor Development

This skill covers working on the WYSIWYG markdown rendering feature in this Zed fork.

## Repository & Branch Structure

- **Fork**: `Feel-ix-343/zed` (fork of `zed-industries/zed`)
- **Main WYSIWYG branch**: `devin/1773098637-wysiwyg-markdown-preview` — all WYSIWYG feature branches should be based off this
- **Upstream**: `zed-industries/zed` (the original Zed editor)

## Building

```bash
# Check the editor crate compiles (fastest feedback loop)
cargo check -p editor

# Full debug build (takes ~20 min first time)
cargo build

# Run the editor locally
cargo run

# Nix build (used by CI, takes ~3.5 hrs on GitHub Actions)
nix build --accept-flake-config
```

## Key Files

### Core WYSIWYG Module
- **`crates/editor/src/markdown_wysiwyg.rs`** — The main module (~1500 lines). Contains:
  - `MarkdownWysiwygState` struct — per-editor state (active flag, block IDs, cached references, saved editor settings)
  - `toggle_markdown_wysiwyg()` — `impl Editor` method that toggles WYSIWYG on/off
  - `maybe_auto_enable_wysiwyg()` — auto-enables WYSIWYG for markdown files when the global flag is set
  - `is_markdown_file()` — checks file extension (.md, .markdown, .mdx)
  - `WYSIWYG_GLOBALLY_ENABLED` — global `AtomicBool` for persistence across editor instances
  - `parse_markdown_decorations()` — parses markdown into decoration structs using `pulldown_cmark`
  - `apply_highlights()` — applies text styling (bold, italic, strikethrough, inline code, headings)
  - `apply_marker_folds()` — folds syntax markers (`**`, `*`, `` ` ``, `#`, `[[`, `]]`)
  - `apply_blocks()` — renders tables, images, and headings as editor blocks
  - `apply_references_block()` — renders backlinks/references block
  - `fetch_references()` — scans workspace for files linking to current file
  - `on_selection_changed()` — refreshes decorations when cursor moves (reveals syntax on cursor line)
  - `try_handle_wikilink_click()` — handles go-to-definition on wikilinks
  - `try_handle_image_paste()` — handles pasting images in WYSIWYG mode
  - `schedule_wysiwyg_refresh()` — debounced re-parse (200ms)
  - `clear_wysiwyg_decorations()` — removes all WYSIWYG decorations

### Editor Integration Points
- **`crates/editor/src/editor.rs`**:
  - `MarkdownWysiwygState` is a field on `Editor` struct
  - `maybe_auto_enable_wysiwyg()` is called at end of `new_internal()` (in the `full_mode` block, after buffer initialization)
  - Action registered in `element.rs` line ~503: `register_action(editor, window, Editor::toggle_markdown_wysiwyg)`
- **`crates/editor/src/element.rs`**:
  - WYSIWYG centering offset calculation (~line 9766)
  - Content offset adjustment for WYSIWYG mode
- **`crates/editor/src/actions.rs`**: `ToggleMarkdownWysiwyg` action definition

## Architecture

### How WYSIWYG Rendering Works
1. User triggers `ToggleMarkdownWysiwyg` action
2. `toggle_markdown_wysiwyg()` saves current editor settings (gutter, line numbers, soft wrap), sets font to serif, enables WYSIWYG state
3. `refresh_wysiwyg_decorations()` is called:
   - Gets buffer text snapshot
   - `parse_markdown_decorations()` parses all markdown syntax into decoration structs
   - Gets cursor position and active line range
   - `apply_highlights()` — sets text styles, hides syntax markers on non-cursor lines
   - `apply_marker_folds()` — folds syntax markers using `FoldPlaceholder`
   - `apply_blocks()` — renders tables/images/headings as custom blocks
4. On cursor move, `on_selection_changed()` triggers `refresh_wysiwyg_decorations()` to reveal/hide syntax

### Key Design Patterns
- **Cursor reveal**: The line(s) the cursor is on show raw markdown; all other lines show rendered output
- **Per-editor state**: `MarkdownWysiwygState` is per `Editor` instance (each open file gets its own)
- **Global persistence**: `WYSIWYG_GLOBALLY_ENABLED` (`AtomicBool`) persists the user's preference across file navigation. Set on toggle, checked during `Editor::new_internal()`
- **Debounced reparsing**: Text changes trigger `schedule_wysiwyg_refresh()` with 200ms debounce
- **Block rendering**: Tables, images, and heading decorations use Zed's `CustomBlock` system
- **Fold-based hiding**: Syntax markers (`**`, `#`, `` ` ``, `[[`, `]]`) are hidden using the fold system with zero-width placeholders

### Markdown File Detection
Files are detected as markdown by extension: `.md`, `.markdown`, `.mdx`

## Cachix CI Pipeline

- **Workflow**: `.github/workflows/cachix.yml`
- **Triggers**: All pushes to the fork
- **Cache**: `oxmd` on cachix.org — auth token stored as `CACHIX_AUTH_TOKEN` GitHub Actions secret
- **Upstream caches used**: `zed.cachix.org`, `cache.garnix.io` (read-only, for pulling pre-built deps)
- **Runner caching**: `DeterminateSystems/magic-nix-cache-action` caches nix store paths in GitHub Actions cache between runs
- **Build time**: ~3.5 hours first run, faster with runner cache populated

### NixOS Integration
- User's nixos config (`Feel-ix-343/nixos`) has `oxmd` flake input pointing to `main` branch
- `oxmd.cachix.org` substituter configured in `flake.nix` nixConfig
- The `oxmd` command launches the WYSIWYG-enabled zed binary

## Common Tasks

### Adding a New Decoration Type
1. Add a struct in `markdown_wysiwyg.rs` (e.g., `struct MyDecoration { range: Range<usize>, ... }`)
2. Add field to `MarkdownDecorations` struct
3. Parse it in `parse_markdown_decorations()` using pulldown_cmark events
4. Apply rendering in `apply_highlights()` (for inline styles) or `apply_blocks()` (for block-level elements)
5. Add syntax markers to hide in `apply_marker_folds()`

### Modifying Toggle Behavior
- Edit `toggle_markdown_wysiwyg()` in the `impl Editor` block
- The on-toggle logic saves/restores: gutter visibility, line numbers, soft wrap mode, text style

### Testing Changes
1. `cargo check -p editor` for fast compilation check
2. `cargo run` to launch the editor
3. Open a `.md` file, toggle WYSIWYG (action: `ToggleMarkdownWysiwyg`)
4. Verify: headings render at correct sizes, bold/italic/strikethrough styled, syntax markers hidden, tables render as grids, cursor reveals raw syntax on current line
5. Navigate between markdown files to test persistence
