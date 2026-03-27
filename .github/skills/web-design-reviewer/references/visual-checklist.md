# Visual Inspection Checklist

Comprehensive checklist for web design visual inspection, updated for modern browsers, devices, and standards.

---

## 1. Layout Verification

### Structural Integrity

- [ ] Header is correctly fixed/sticky at the top of the screen
- [ ] Footer is positioned at the bottom of the screen or end of content
- [ ] Main content area is center-aligned with appropriate `max-width` / container
- [ ] Sidebar (if present) is positioned correctly and collapses appropriately
- [ ] Navigation is displayed in the intended position
- [ ] Sticky elements do not overlap or obscure content when stacked
- [ ] CSS `@layer` cascade order (if used) produces the expected specificity

### Overflow

- [ ] Horizontal scrollbar is not unintentionally displayed
- [ ] Content does not overflow from parent elements
- [ ] Images and media fit within parent containers
- [ ] Tables do not exceed container width (consider horizontal scroll wrapper)
- [ ] Code blocks wrap or scroll appropriately
- [ ] Container query-based components don't overflow in narrow contexts

### Alignment

- [ ] Grid and subgrid items are evenly distributed
- [ ] Flex item alignment is correct
- [ ] Text alignment (start/center/end) is consistent with reading direction
- [ ] Icons and text are vertically aligned
- [ ] Form labels and input fields are correctly positioned
- [ ] Anchor-positioned elements (popovers, tooltips) stay within viewport

---

## 2. Typography Verification

### Readability

- [ ] Body text font size is sufficient (minimum 16px / 1rem recommended)
- [ ] Line height is appropriate (1.4–1.6 for body, 1.1–1.3 for headings)
- [ ] Characters per line is appropriate (45–75 characters recommended)
- [ ] Spacing between paragraphs is sufficient
- [ ] Heading size hierarchy is clear and uses a consistent scale
- [ ] `clamp()` or fluid typography scales smoothly across viewport sizes

### Text Handling

- [ ] Long words and URLs wrap appropriately (`overflow-wrap: anywhere` or similar)
- [ ] No text clipping occurs
- [ ] Ellipsis / line-clamp displays correctly
- [ ] Language-specific line-breaking and hyphenation rules work correctly
- [ ] Text remains legible over background images (overlay or text-shadow)

### Fonts

- [ ] Variable fonts load correctly and axes (weight, width, slant) render as intended
- [ ] Fallback fonts are appropriate and size-adjusted (`size-adjust`, `ascent-override`)
- [ ] Font weights and optical sizes are as intended
- [ ] Special characters, emoji, and icon fonts display correctly
- [ ] `font-display: swap` or `optional` is set to avoid invisible text

---

## 3. Color & Contrast Verification

### Accessibility (WCAG 2.2 Standards)

- [ ] Body text: contrast ratio ≥ 4.5:1 (AA)
- [ ] Large text (18px+ bold or 24px+): contrast ratio ≥ 3:1
- [ ] Non-text UI components and graphical objects: contrast ratio ≥ 3:1
- [ ] Focus indicators: ≥ 3:1 contrast against adjacent colors (WCAG 2.2 enhanced)
- [ ] Adjacent color contrast between interactive elements and their surroundings

### Color Consistency

- [ ] Design tokens / CSS custom properties are used consistently
- [ ] Link colors are consistent and distinguishable from body text
- [ ] Semantic state colors (error, success, warning, info) are unified
- [ ] Hover/active/focus-visible state colors are appropriate
- [ ] Color transitions between states are smooth

### Color Spaces & Gamut

- [ ] Wide-gamut colors (`oklch`, `oklab`, `display-p3`) render correctly on supported displays
- [ ] Fallback sRGB values are defined for older browsers
- [ ] Colors behave consistently across light and dark themes

### Color Vision Diversity

- [ ] Information conveyed by shape, text, or pattern — not just color
- [ ] Charts and data visualizations consider color vision diversity
- [ ] Error messages don't rely solely on red color

---

## 4. Dark Mode & Theming

### Theme Support

- [ ] `prefers-color-scheme: dark` is respected
- [ ] Manual theme toggle works and persists user preference
- [ ] Transition between themes is smooth (no flash of wrong theme on load)
- [ ] Images and media adapt to theme (dark-mode variants or `filter` adjustments)

### Dark Mode Specific

- [ ] Background uses dark grays (not pure #000) to avoid halation
- [ ] Text contrast meets WCAG 2.2 AA in dark theme
- [ ] Shadows and elevation are replaced with lighter borders or subtle glows
- [ ] Brand colors are adjusted for legibility on dark backgrounds
- [ ] Embedded content (iframes, third-party widgets) doesn't create bright rectangles
- [ ] Scrollbar styling adapts to theme (`color-scheme: dark`)

---

## 5. Responsive Verification

### Mobile (up to 767px)

- [ ] Content fits within screen width including narrow devices (280–428px)
- [ ] Touch targets are ≥ 24×24 CSS px minimum (WCAG 2.2 AA), 44×44px recommended
- [ ] Text is readable without zooming
- [ ] No unintended horizontal scrolling
- [ ] Navigation is mobile-friendly (bottom sheet, hamburger, etc.)
- [ ] Form inputs trigger appropriate virtual keyboard (`inputmode`, `type`)
- [ ] Dynamic viewport units (`dvh`, `svh`) are used instead of `vh` where needed

### Tablet (768px – 1023px)

- [ ] Layout adapts to tablet width (intermediate layout, not stretched mobile)
- [ ] Multi-column layouts display appropriately
- [ ] Image sizes are appropriate for density
- [ ] Sidebar show/hide behavior is appropriate
- [ ] Works in both portrait and landscape orientations

### Desktop (1024px+)

- [ ] Content max-width is set and centered (typically 1200–1440px)
- [ ] Layout doesn't break or look sparse on large / ultra-wide displays (2560px+)
- [ ] Spacing and whitespace scale appropriately
- [ ] Multi-column layouts function correctly
- [ ] Hover and focus-visible states are implemented
- [ ] High-DPI assets (`@2x`, `@3x`, `srcset` with density descriptors) are served

### Container Queries

- [ ] Components using `@container` resize correctly inside different layout contexts
- [ ] Container query breakpoints produce sensible layouts (card in sidebar vs. main)
- [ ] No layout thrashing between container size thresholds

### Breakpoint Transitions

- [ ] Layout transitions smoothly when viewport changes size
- [ ] No content disappears, duplicates, or jumps at any intermediate size
- [ ] Resize handles and drag interactions work across breakpoints

---

## 6. Interactive Element Verification

### Buttons

- [ ] Default state is clear and visually distinct
- [ ] Hover state exists (pointer devices)
- [ ] `:focus-visible` state is visually clear (keyboard only, not on click)
- [ ] Active (pressed) state provides tactile feedback
- [ ] Disabled state is visually distinct and conveys unavailability
- [ ] Loading / pending state prevents double submission

### Links

- [ ] Links are visually identifiable (underline or clear affordance)
- [ ] Visited links are distinguishable (where appropriate)
- [ ] Hover and focus-visible states exist
- [ ] Link text is descriptive (not "click here")

### Form Elements

- [ ] Input field boundaries are clear against the background
- [ ] Placeholder text has appropriate contrast (≥ 4.5:1 not required, but legible)
- [ ] `:focus-visible` provides clear visual ring
- [ ] Validation states (error, success) are shown with icon + text + color
- [ ] Required field indication is accessible (not color-only)
- [ ] Dropdowns, date pickers, and custom selects work with keyboard
- [ ] Autocomplete attributes are set for common fields

### Popovers, Dialogs & Overlays

- [ ] Native `<dialog>` / Popover API elements render correctly
- [ ] Backdrop dims background content appropriately
- [ ] Escape key and outside click dismiss behavior works
- [ ] Focus is trapped within modal dialogs
- [ ] Scroll is locked on body when modal is open
- [ ] Anchor-positioned elements stay attached during scroll and resize

---

## 7. Images & Media Verification

### Images

- [ ] Images display at appropriate size and are responsive
- [ ] Aspect ratio is maintained (`aspect-ratio` or intrinsic sizing)
- [ ] Responsive images use `srcset` + `sizes` with density and width descriptors
- [ ] AVIF / WebP formats are served with fallback
- [ ] Broken image state is handled gracefully (alt text visible, no broken icon)
- [ ] Lazy loading (`loading="lazy"`) works for below-fold images
- [ ] Above-fold hero/LCP images are eagerly loaded with `fetchpriority="high"`

### Video & Embeds

- [ ] Videos are responsive and maintain aspect ratio
- [ ] Embedded content (iframes) is responsive
- [ ] Video controls are accessible via keyboard
- [ ] Captions / subtitles are available when applicable
- [ ] Autoplay respects user preference (muted only if autoplay is needed)

### SVG & Icons

- [ ] SVG icons scale correctly and respect `currentColor`
- [ ] Decorative SVGs have `aria-hidden="true"`
- [ ] Meaningful SVGs have accessible labels
- [ ] Icon-only buttons have visible or `aria-label` text

---

## 8. Accessibility Verification (WCAG 2.2)

### Keyboard Navigation

- [ ] All interactive elements accessible via Tab key
- [ ] Focus order is logical and follows DOM order
- [ ] `:focus-visible` is styled — `:focus` ring is not suppressed globally
- [ ] Focus traps are appropriate (modals) and escapable
- [ ] Skip-to-content link exists and works
- [ ] No keyboard traps in custom widgets

### Target Size (WCAG 2.2 — 2.5.8)

- [ ] Interactive targets are ≥ 24×24 CSS px (AA minimum)
- [ ] Inline links in text are exempt but surrounded by adequate spacing
- [ ] Spacing between adjacent targets prevents accidental activation

### Dragging Movements (WCAG 2.2 — 2.5.7)

- [ ] Drag-and-drop has a single-pointer alternative (buttons, menus)
- [ ] Reorder/sort interactions work via keyboard

### Consistent Help (WCAG 2.2 — 3.2.6)

- [ ] Help mechanisms (chat, FAQ link, contact) appear in consistent location

### Redundant Entry (WCAG 2.2 — 3.3.7)

- [ ] Previously entered information is auto-populated or selectable in multi-step flows

### Screen Reader Support

- [ ] Images have descriptive `alt` text (empty `alt=""` for decorative)
- [ ] Forms have associated `<label>` elements
- [ ] ARIA is used sparingly and correctly (no ARIA is better than bad ARIA)
- [ ] Heading hierarchy is correct and sequential (h1 → h2 → h3…)
- [ ] Live regions (`aria-live`) announce dynamic content changes
- [ ] Landmarks (`<main>`, `<nav>`, `<aside>`, `<header>`, `<footer>`) are present

### Motion & Preferences

- [ ] `prefers-reduced-motion: reduce` disables non-essential animations
- [ ] `prefers-contrast: more` increases contrast when requested
- [ ] `forced-colors` / high contrast mode renders controls correctly
- [ ] `prefers-reduced-transparency` is respected where applicable

---

## 9. Performance-related Visual Issues

### Core Web Vitals

- [ ] **LCP** (Largest Contentful Paint): hero element loads within 2.5s
- [ ] **CLS** (Cumulative Layout Shift): score ≤ 0.1 — no unexpected jumps
- [ ] **INP** (Interaction to Next Paint): interactions respond within 200ms
- [ ] Font FOUT/FOIT is minimized (preload critical fonts, `font-display`)
- [ ] No layout shift when images, ads, or embeds load (dimensions reserved)
- [ ] Skeleton / placeholder states are appropriate for async content

### Animation & Transitions

- [ ] Animations run at 60fps+ (use `transform` and `opacity`, avoid layout triggers)
- [ ] Scroll-driven animations degrade gracefully on unsupported browsers
- [ ] View Transitions API (if used) animates smoothly between states
- [ ] No performance issues during scroll (no forced reflows, passive listeners used)
- [ ] `will-change` is used sparingly and removed after animation completes

### Loading Strategy

- [ ] Critical CSS is inlined or loaded first
- [ ] Non-critical CSS/JS is deferred or lazy-loaded
- [ ] Third-party scripts don't block rendering
- [ ] Resource hints (`preconnect`, `preload`, `prefetch`) are set for critical assets
- [ ] `content-visibility: auto` is used for heavy off-screen sections

---

## 10. PWA & Cross-platform Considerations

### Progressive Web App (if applicable)

- [ ] App installs correctly and splash screen displays properly
- [ ] Status bar and safe area insets (`env(safe-area-inset-*)`) are handled
- [ ] Standalone display mode renders without browser chrome issues
- [ ] Offline fallback page is styled and useful

### Cross-browser

- [ ] Tested in Chromium, Firefox, and Safari (WebKit)
- [ ] CSS features use fallbacks or `@supports` for progressive enhancement
- [ ] Scrollbar styling works across browsers or degrades gracefully
- [ ] `<select>`, `<input>`, and native controls are acceptable across browsers

---

## Priority Matrix

| Priority | Category | Examples |
|----------|----------|----------|
| P0 (Critical) | Functionality breaking | Complete element overlap, content disappearance, broken interactivity |
| P1 (High) | Accessibility / UX blockers | Unreadable text, unreachable controls, missing focus states, failed WCAG 2.2 AA |
| P2 (Medium) | Visual / responsive issues | Alignment problems, breakpoint breakage, dark mode inconsistencies |
| P3 (Low) | Polish | Slight spacing differences, minor color variations, animation smoothness |

---

## Verification Tools

### Browser DevTools

- Elements panel: DOM, styles, computed values, container query debugging
- Performance panel: Core Web Vitals, layout shifts, long tasks
- Lighthouse / PageSpeed Insights: audits for performance, accessibility, SEO
- Device toolbar: responsive testing across device presets
- CSS overview: color and font inventory
- Rendering panel: emulate `prefers-color-scheme`, `prefers-reduced-motion`, `forced-colors`

### Accessibility Tools

- axe DevTools / axe-core
- WAVE
- WCAG Color Contrast Checker (supports APCA and WCAG 2.2)
- Accessibility Insights for Web
- Screen readers: VoiceOver (macOS/iOS), NVDA (Windows), TalkBack (Android)

### Visual Regression & Automation

- Playwright (screenshot comparison, cross-browser)
- Percy / Chromatic (visual regression testing in CI)
- Storybook (component-level visual testing)
- BackstopJS (full-page visual diffing)
