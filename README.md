# WordPress Theme Reviewer OJSpro

An automated WordPress block theme quality assurance skill for Claude Code. Catches the visual, structural, and functional issues that slip through manual testing.

## What It Checks

| # | Check | Severity | What It Catches |
|---|-------|----------|----------------|
| 1 | Post Grid Equal-Height Cards | Warning | Ragged card heights, metadata not pinned to bottom |
| 2 | Grid Gap / Spacing | Warning | Cards touching with no gap between columns/rows |
| 3 | Tab/Panel Visibility CSS Conflicts | Critical | Content disappearing due to mixed show/hide mechanisms |
| 4 | Scroll-to-Section Tab Navigation | Critical | Broken tab navigation, IntersectionObserver overriding click state |
| 5 | Sticky Element z-index/Background | Warning | Content showing through sticky headers/navs |
| 6 | CSS Version Cache Busting | Info | Static version strings causing stale cached assets |
| 7 | Redundant Content Sections | Info | Duplicate sections across patterns and templates |
| 8 | Clickable Author Names | Info | Author names as static text instead of archive links |
| 9 | Featured Image Presence | Warning | Single templates with no visual header/hero |
| 10 | Template-Pattern Wiring | Critical | Broken pattern slug references |
| 11 | Theme.json Token Completeness | Info | CSS tokens missing from theme.json (breaks Site Editor) |
| 12 | JS Event Listener Timing | Warning | Scripts running before DOM elements exist |
| 13 | Sidebar Layout Spacing | Info | Cramped sidebars, insufficient gaps |
| 14 | Accessibility Basics | Warning | Missing ARIA roles, alt text, focus styles |
| 15 | wp:query Block Configuration | Warning | Wrong postType, missing no-results fallback |

## Installation

### Claude Code (recommended)

```bash
claude skill install /path/to/wordpress-theme-reviewer
```

Or copy `SKILL.md` to your `.claude/skills/` directory.

### Manual

Place the `SKILL.md` file in any of these locations:
- Project: `.claude/skills/wp-theme-reviewer/SKILL.md`
- User: `~/.claude/skills/wp-theme-reviewer/SKILL.md`

## Usage

In Claude Code, just say:

```
Review my WordPress theme
```

```
Check my theme for issues
```

```
Run a theme QA pass on wp-content/themes/my-theme
```

```
Theme health check
```

The skill triggers automatically on these phrases and runs all 15 checks against your theme directory.

## Origin

Every check in this skill exists because it caught a real bug during theme development. Built from a multi-session development of the "Lexara" theme for The Law Brigade Publishers — a full-scale WordPress block theme with 13 page types, custom post types, tabbed interfaces, and complex grid layouts.

### Real bugs that inspired these checks:

- **WordPress generates TWO class formats** for post-template grids (`.is-layout-grid` space-separated AND hyphenated). Missing one = half your grids break silently.
- **CSS `display: none` on tab panels** + HTML `hidden` attribute = content vanishes with no error.
- **IntersectionObserver** for tab highlighting fires during smooth scroll, immediately overriding the tab you just clicked.
- **`ver=1.0.0`** static cache strings mean your CSS changes never reach the browser without hard refresh.

## Requirements

- WordPress block theme (must have `theme.json`, `templates/`, `patterns/`)
- Claude Code with file system access
- Optional: running local WordPress site (Studio CLI) for live verification

## License

MIT

## Author

[OJSpro](https://github.com/OJSpro)
