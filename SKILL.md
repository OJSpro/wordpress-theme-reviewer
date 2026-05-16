---
name: wp-theme-reviewer
description: "Automated WordPress block theme reviewer that checks for common visual, structural, and functional issues. Use this skill whenever the user asks to review a WordPress theme, audit theme quality, check for CSS/JS issues, validate templates, or wants a pre-launch theme QA pass. Also trigger when the user says 'review my theme', 'check my theme', 'theme audit', 'theme QA', 'find theme bugs', or 'theme health check'."
---

# WordPress Theme Reviewer OJSpro

Automated quality assurance for WordPress block themes. This skill performs a comprehensive review covering layout consistency, CSS correctness, JavaScript functionality, template structure, pattern rendering, and accessibility — catching the issues that slip through manual testing.

Born from real-world theme development sessions where these exact bugs were found, diagnosed, and fixed. Each check exists because it caught a real problem.

## When to run

Run this skill against any WordPress block theme directory. The theme must be a **block theme** (has `theme.json`, `templates/`, `patterns/`). Classic themes are out of scope.

## How to run

1. User provides theme path (or you detect it from context)
2. Read key files: `style.css`, `theme.json`, `functions.php`, all templates, all patterns, CSS, JS
3. Run each check category below
4. Report findings grouped by severity: **Critical** (breaks functionality), **Warning** (visual/UX issue), **Info** (improvement suggestion)

---

## Check Categories

### 1. Post Grid Equal-Height Cards

**Why:** WordPress post grids (`wp:post-template` with grid layout) don't guarantee equal-height cards. Cards with different content lengths create ragged layouts, and metadata (author, date) floats up instead of pinning to the bottom.

**What to check:**

- Look for `wp:post-template` blocks with `layout.type: "grid"` in all templates
- Check if the theme CSS targets BOTH class name formats WordPress generates:
  - `.wp-block-post-template.is-layout-grid` (space-separated)
  - `.wp-block-post-template-is-layout-grid` (hyphenated)
- Verify these CSS rules exist:
  ```css
  /* Both selectors must be present */
  .wp-block-post-template.is-layout-grid,
  .wp-block-post-template-is-layout-grid {
    align-items: stretch;
    gap: /* any value, but must be set */;
  }
  /* Cards must be flex columns */
  .wp-block-post-template.is-layout-grid > li,
  .wp-block-post-template-is-layout-grid > li {
    display: flex;
    flex-direction: column;
  }
  /* Inner group must flex-grow */
  .wp-block-post-template > li > .wp-block-group {
    flex: 1;
    display: flex;
    flex-direction: column;
  }
  /* Last child pins to bottom */
  .wp-block-post-template .wp-block-group > *:last-child {
    margin-top: auto;
  }
  ```

**Common miss:** Only targeting `.wp-block-post-template.is-layout-grid` and missing the hyphenated variant. WordPress generates both — if you miss one, half your grids break.

**Severity:** Warning (visual inconsistency)

---

### 2. Grid Gap / Spacing

**Why:** `wp:post-template` grid blocks often render cards touching each other with no gap, even when `blockGap` is set in the block markup, because WordPress doesn't always apply the gap CSS correctly.

**What to check:**

- Verify explicit `gap` value in CSS for post-template grids
- Check that `blockGap` in template JSON matches CSS gap value
- If using `wp:post-template` without explicit gap, flag it

**Severity:** Warning

---

### 3. Tab/Panel Visibility CSS Conflicts

**Why:** When using tabbed interfaces, CSS selectors for showing/hiding panels can conflict. Common pattern: CSS uses `.is-active` class but HTML uses `hidden` attribute (or vice versa), causing content to disappear.

**What to check:**

- Find all tab-related CSS (`.tlb-tab`, `.tab`, `[role="tab"]`, etc.)
- Find all panel-related CSS (`.tlb-tabpanel`, `.tabpanel`, `[role="tabpanel"]`, etc.)
- Verify the show/hide mechanism is consistent:
  - If CSS uses `[hidden] { display: none }`, HTML must use `hidden` attribute
  - If CSS uses `.is-active { display: block }`, JS must toggle `.is-active` class
  - Mixed approaches = content disappears
- Check for blanket `display: none` rules on panel classes without corresponding show rules

**Severity:** Critical (content disappears)

---

### 4. Scroll-to-Section Tab Navigation

**Why:** For long-form content with section tabs (like case analyses, articles with TOC), hide/show tabs are wrong — users expect to see all content with tabs as navigation anchors that scroll to sections.

**What to check:**

- If tabs link to `#section-*` anchors, verify:
  - All sections have matching `id` attributes
  - No `hidden` attribute on any section
  - `scroll-margin-top` set on sections (to account for sticky headers)
  - JS uses `scrollIntoView({ behavior: 'smooth' })` on tab click
  - `e.preventDefault()` on anchor clicks to prevent default jump
- If using IntersectionObserver for active tab highlighting:
  - Check for click-guard flag to prevent observer from overriding tab highlight during smooth scroll animation
  - Observer should be suppressed for ~800ms after click
  - `rootMargin` should account for sticky header height

**Example of the click-guard pattern:**
```javascript
let clicking = false;
tab.addEventListener('click', (e) => {
  clicking = true;
  // ... scroll logic ...
  setTimeout(() => { clicking = false; }, 800);
});
observer = new IntersectionObserver((entries) => {
  if (clicking) return; // Skip during programmatic scroll
  // ... highlight logic ...
});
```

**Severity:** Critical (broken navigation) / Warning (observer override)

---

### 5. Sticky Element z-index and Background

**Why:** Sticky nav bars, tab bars, and headers need explicit `background` and `z-index` or content scrolls behind/through them, creating visual chaos.

**What to check:**

- Find all `position: sticky` elements
- Verify each has:
  - `background` (not `transparent` — must match page background)
  - `z-index` (typically 10-50, must not conflict with modals/overlays)
  - `top: 0` (or appropriate offset)
- Check for `border-bottom` on sticky elements (visual separation when stuck)

**Severity:** Warning

---

### 6. CSS Version Cache Busting

**Why:** Static version strings like `ver=1.0.0` cause browsers to cache old CSS/JS indefinitely. Changes don't appear until hard refresh.

**What to check:**

- In `functions.php`, find `wp_enqueue_style` and `wp_enqueue_script` calls
- Check the version parameter:
  - **Bad:** Hard-coded string like `'1.0.0'`
  - **Better:** `wp_get_theme()->get('Version')` (at least changes on version bump)
  - **Best:** `filemtime(get_template_directory() . '/assets/css/file.css')` (changes on every file save)
- Flag any enqueue call with a static version string

**Severity:** Info (development friction)

---

### 7. Redundant Content Sections

**Why:** When patterns and templates evolve, content sections can end up duplicated — e.g., "Related Cases" both inline in a pattern AND as a separate `wp:query` block in the template.

**What to check:**

- For each template, trace all included patterns (`wp:pattern`)
- Check if the template has sections that duplicate content already in a pattern
- Common duplicates:
  - Related posts (in pattern sidebar AND template footer)
  - Citations (in header pattern AND separate template section)
  - Author info (in article pattern AND template)

**Severity:** Info

---

### 8. Clickable Author Names

**Why:** Author names in bylines should link to the author archive page. Static text author names are a missed navigation opportunity.

**What to check:**

- Find author name displays in patterns and templates
- Verify they're wrapped in `<a href="/author/.../">` or use `wp:post-author` block with `isLink: true`
- Check that the link `href` uses a valid author URL pattern

**Severity:** Info

---

### 9. Featured Image Presence

**Why:** Templates without featured images look incomplete, especially for article/case/journal single pages. Even a gradient placeholder is better than nothing.

**What to check:**

- For each `single-*.html` template, check for:
  - `wp:post-featured-image` block, OR
  - Custom hero/header pattern with image, OR
  - Gradient placeholder div
- Flag single templates with no visual header element

**Severity:** Warning

---

### 10. Template-Pattern Wiring

**Why:** Patterns referenced in templates must exist, have correct slugs, and be registered in the right category.

**What to check:**

- Extract all `wp:pattern {"slug":"theme-name/pattern-name"}` references from templates
- Verify each pattern file exists in `patterns/` directory
- Verify the pattern's `Slug` header matches the reference
- Check pattern `Categories` header matches registered categories in `functions.php` or `inc/pattern-categories.php`

**Severity:** Critical (broken template rendering)

---

### 11. Theme.json Token Completeness

**Why:** Design tokens defined in CSS but not in `theme.json` can't be used in the Site Editor, breaking the visual editing experience.

**What to check:**

- Parse CSS custom properties from the main stylesheet
- Parse `theme.json` `settings.custom`, `settings.color.palette`, `settings.typography.fontSizes`, `settings.spacing.spacingSizes`
- Flag CSS tokens that have no `theme.json` equivalent
- Key areas: color palette, font sizes, spacing scale, border radii

**Severity:** Info

---

### 12. JavaScript Event Listener Timing

**Why:** Scripts enqueued in the footer (`true` as 5th param to `wp_enqueue_script`) run after DOM is ready. But scripts enqueued in the header may run before elements exist. IIFE patterns need DOM elements to be present.

**What to check:**

- Check if JS is enqueued with `true` (footer) parameter
- If JS uses IIFE `(function() { ... })()`, verify elements it queries exist in the DOM at script execution time
- If JS queries elements from `wp:html` blocks, those render inline (before footer scripts) — this is fine
- If JS queries elements from dynamic blocks or patterns with PHP rendering, there could be timing issues
- Check for `DOMContentLoaded` wrapper if script is in header

**Severity:** Warning (silent failures)

---

### 13. Sidebar Layout Spacing

**Why:** Two-column layouts with sidebars often have cramped sidebars — too narrow, insufficient padding, sections crammed together.

**What to check:**

- Find `grid-template-columns` definitions in patterns/templates
- Sidebar column should be at least 240px (280px preferred)
- Gap between sidebar and main content should be at least 32px (48px preferred)
- Sidebar sections should have `border-top` or `margin-top` separation (at least 20px)
- `line-height` in sidebar text should be at least 1.5

**Severity:** Info

---

### 14. Accessibility Basics

**Why:** WCAG AA compliance is a baseline expectation. Common issues in custom themes: missing alt text, poor color contrast, no keyboard navigation for custom components.

**What to check:**

- Tab components: `role="tab"`, `role="tabpanel"`, `aria-selected`, `aria-controls`
- Images: `alt` attributes present (including placeholder/decorative images with `alt=""`)
- Color contrast: Check key color pairs from `theme.json` palette (e.g., body text on background, link colors)
- Focus styles: Check if custom components have `:focus-visible` styles
- Skip to content: Check for skip link in header

**Severity:** Warning (a11y violations)

---

### 15. wp:query Block Configuration

**Why:** Query blocks with wrong `postType`, missing `inherit: false`, or no `query-no-results` fallback cause silent failures or unexpected content.

**What to check:**

- All `wp:query` blocks in templates
- Verify `postType` matches an existing registered CPT
- If the query should NOT inherit from the main query (e.g., "Related posts" section), `inherit` must be `false`
- Check for `wp:query-no-results` block inside each query
- Verify `perPage` is reasonable (not 100+ for a card grid)

**Severity:** Warning

---

## Output Format

```markdown
# Theme Review: [Theme Name] v[Version]

## Summary
- **Critical:** X issues
- **Warning:** Y issues  
- **Info:** Z suggestions

## Critical Issues
### [Check Name]
**File:** `path/to/file`
**Line:** XX
**Issue:** Description of what's wrong
**Fix:** How to fix it

## Warnings
...

## Suggestions
...

## Passed Checks
- [x] Check name — OK
```

---

## Running Against a Live Site (Optional)

If a local WordPress site is running (via Studio or otherwise):

1. Get site URL: `studio site status`
2. Curl key pages and check rendered HTML:
   - Homepage: grid cards rendered correctly?
   - Single post: featured image, author link, TOC?
   - Custom post types: correct template applied?
   - Search: results render with fallback?
3. Check browser console for JS errors: use Chrome MCP `read_console_messages`
4. Verify CSS/JS version strings in rendered `<link>` and `<script>` tags

This live verification catches issues that static file analysis misses (like WordPress class name generation quirks).
