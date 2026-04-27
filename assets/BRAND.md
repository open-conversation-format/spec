# OCF Brand Guidelines

Style description for the Open Conversation Format identity. This document is the source of truth for how the OCF wordmark, typography, and colors are used across documentation, the GitHub org profile, and any downstream tooling.

The brand stays deliberately small: one wordmark, one typeface, two colors. No mascot, no auxiliary symbol, no tagline. Spec standards live longer than design trends - minimalism is the goal.

---

## Wordmark

The OCF mark is **`[ OCF ]`** - the format name enclosed by square brackets, with breathing space inside.

Two readings exist for the brackets, both intentional and neither needs to be explained to the viewer:

1. **Format frame** - the square brackets signal "structured data / format definition", in the same family as JSON's `{}` or YAML's flat wordmark.
2. **Two abstract speakers** - the brackets face each other, framing the OCF text as the conversation that lives between them. This is the conversation-format reading.

The mark is intentionally pure typography. No icon, no underline, no decoration. The brackets carry both meanings; adding more would dilute either.

### Variants

| File | Use case |
|---|---|
| `assets/ocf-logo.svg` | Primary - black on transparent, for light backgrounds. |
| `assets/ocf-logo-inverse.svg` | Inverse - white on transparent, for dark backgrounds. |
| `assets/ocf-org-avatar.svg` / `.png` | Square 500×500, white background - GitHub organization avatar. |
| `assets/ocf-social-preview.svg` / `.png` | Wide 1280×640, wordmark + "Open Conversation Format" + URL - GitHub repo social preview (open-graph card). |

For automatic dark/light switching in GitHub markdown:

```html
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/ocf-logo-inverse.svg">
  <img src="assets/ocf-logo.svg" alt="OCF" width="200">
</picture>
```

For markdown without `<picture>` support, default to `ocf-logo.svg` (black). Most GitHub READMEs render on a white-ish surface either way.

---

## Typography

### Wordmark font

**JetBrains Mono Bold** (weight 800).

- License: SIL Open Font License (OFL) - freely usable, freely redistributable.
- Source: <https://www.jetbrains.com/lp/mono/> · <https://github.com/JetBrains/JetBrainsMono>

The wordmark in `ocf-logo.svg` and `ocf-logo-inverse.svg` is **outlined as SVG paths** - no font dependency at render time. The files are self-contained (~1.6 KB each) and render identically on any system, regardless of installed fonts.

### Adjacent text

When OCF appears in running text adjacent to the logo (README headings, navigation, code-style references), use this font stack:

```css
font-family: "JetBrains Mono", ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
font-weight: 700;
```

For body text in OCF documentation, no specific font is mandated - system defaults are acceptable. If a brand-coherent body font is desired, **Inter** pairs cleanly with JetBrains Mono and is widely available.

### Why JetBrains Mono

Considered alternatives: IBM Plex Mono (slightly more neutral but very similar), Iosevka (narrower, more distinctive but less common), Inter (sans, less code-flavored). JetBrains Mono won on three counts:

- Strong dev-community recognition - widely used in IDEs, terminals, conference slides.
- Distinctive letterforms (the `C` terminals, the slightly elongated bracket arms) without becoming idiosyncratic.
- Permissive license, actively maintained, complete weight range.

---

## Colors

| Token | Hex | Usage |
|---|---|---|
| **Primary** | `#0E1116` | Wordmark fill on light backgrounds. Matches GitHub's dark-theme background - visually anchored, not "pure black". |
| **Inverse** | `#F0F6FC` | Wordmark fill on dark backgrounds. Matches GitHub's light-theme text - visually anchored, not "pure white". |
| **Accent · Purple** *(optional)* | `#7C3AED` | Reserved for distinct decorative elements (markers, indicator dots). Never as wordmark fill. Tailwind violet-600. |
| **Accent · Blue** *(optional)* | `#0969DA` | Same usage as purple, for GitHub-native contexts. GitHub's primary blue. |

### Color rules

- **The wordmark is monochrome.** Never a gradient, never multi-colored. Either Primary `#0E1116` or Inverse `#F0F6FC`.
- **Accents are markers, not fills.** If an accent color appears, it's on a small element (a dot, a separator) - not on the OCF letters or the brackets.
- **No off-palette colors.** Avoid red, green, orange, etc. - they don't belong in the OCF identity.
- **Contrast is mandatory.** Combinations must satisfy WCAG AA at minimum. Primary on Inverse and Inverse on Primary both satisfy this trivially.

---

## Sizing

| Context | Recommended width | Notes |
|---|---|---|
| Favicon | 32 × 32 (square crop) | Tight crop around the wordmark. Below 32 px the brackets become illegible - use a square mark variant if needed. |
| Inline documentation | 80 - 120 px | Beside headings, in navigation. |
| README header | 200 - 240 px | Centered above the H1. |
| Open Graph / social card | 480 - 600 px | Plenty of clear space around. |
| Print / merchandise | scale freely | SVG vector - no quality loss at any size. |

**Minimum render size: 80 px wide.** Below this the wordmark becomes hard to read.

**Clear space:** Maintain padding equal to the cap-height of the wordmark on all sides. The `viewBox` of `ocf-logo.svg` already includes built-in padding (the wordmark is centered with ~30 px on each side at the source viewBox).

---

## File specifications

### `ocf-logo.svg` and `ocf-logo-inverse.svg`

| Property | Value |
|---|---|
| Format | SVG (paths, no fonts) |
| viewBox | `0 0 328 100` |
| Height ratio | wordmark sits centered around `y = 50`, baseline at ~`y = 73` |
| File size | ~1.6 KB each |
| Outlining source | JetBrains Mono Bold (master branch, OFL-licensed) |
| Tooling | Generated via `fontTools.pens.svgPathPen` - see regeneration notes below |

### Regenerating the logo files

If the wordmark needs regeneration (e.g., JetBrains Mono updates, or a new variant is needed), the conversion uses Python + fontTools:

```python
from fontTools.ttLib import TTFont
from fontTools.pens.svgPathPen import SVGPathPen

# Load font, extract glyph paths for "[ OCF ]",
# transform each path with translate + scale + Y-flip,
# emit as SVG <path> elements with the chosen fill color.
```

Full script lives in the repository tooling (or can be reconstructed from the example above). The output is deterministic given the same input font and parameters.

---

## Don'ts

- ❌ Don't stretch, squash, or skew the wordmark.
- ❌ Don't change the bracket style - no `{OCF}`, no `<OCF>`, no `(OCF)`. Square brackets are the mark.
- ❌ Don't remove the breathing space - `[OCF]` (tight) was considered and rejected. The mark is `[ OCF ]`.
- ❌ Don't add decorative effects: no drop shadow, glow, gradient, outline, or 3D extrude.
- ❌ Don't substitute the typography. Rendering OCF in a different typeface is not the OCF mark.
- ❌ Don't recolor with off-palette colors.
- ❌ Don't combine the wordmark with another logo such that they appear as one entity.

## Do's

- ✅ Use SVG (not PNG) wherever possible - vector scaling preserves quality.
- ✅ Use `ocf-logo-inverse.svg` on dark backgrounds rather than forcing the dark variant.
- ✅ Honor the minimum size (80 px wide).
- ✅ Use accent colors only for marker elements, not as wordmark fill.
- ✅ For HTML/markdown contexts, use `<picture>` for automatic dark/light switching.
- ✅ When in doubt, default to **black wordmark on light background** - this is the canonical form.

---

## Files

```
assets/
├── ocf-logo.svg              Primary wordmark - black on transparent (light backgrounds)
├── ocf-logo-inverse.svg      Inverse wordmark - white on transparent (dark backgrounds)
├── ocf-org-avatar.svg        Square 500×500 wordmark with white bg - GitHub org avatar source
├── ocf-org-avatar.png        Same, rasterized for upload to GitHub Organization Settings
├── ocf-social-preview.svg    1280×640 wordmark + tagline + URL - open-graph card source
├── ocf-social-preview.png    Same, rasterized for upload to GitHub Repo Settings
├── BRAND.md                  This document
└── explorations/             Earlier design exploration variants (kept for reference)
```

The `explorations/` folder contains earlier design candidates that informed the current mark - alternate typefaces, with-circle versions, accent-color experiments. They're retained as a record of the design process; downstream tooling should ignore them.

## Upload instructions for GitHub

GitHub avatars and social preview images are uploaded through the web UI - there is no API or git path to push them. Upload steps:

### Organization avatar

1. Go to <https://github.com/organizations/open-conversation-format/settings/profile>
2. Under "Profile picture", click "Upload a photo"
3. Select `assets/ocf-org-avatar.png`
4. Save changes

The image renders inside a circle in most GitHub contexts. The wordmark is centered with adequate padding so the circle crop never clips the brackets.

### Repository social preview

For each repo (start with `spec`, repeat for `.github` and future repos):

1. Go to repository Settings (e.g. <https://github.com/open-conversation-format/spec/settings>)
2. Scroll to "Social preview"
3. Click "Edit" → "Upload an image…"
4. Select `assets/ocf-social-preview.png`
5. The image will be displayed when the repo is shared on Twitter, Slack, Discord, LinkedIn, etc.

Both images are deterministically regenerable from `ocf-logo.svg` plus the brand tokens documented above; the tooling lives in `tools/` (or can be reconstructed from the spec - see the regeneration recipe in [File specifications](#file-specifications)).

---

## License

The OCF wordmark and these brand assets are released under the [Apache License 2.0](../LICENSE), the same license as the OCF specification. JetBrains Mono - used to outline the wordmark - is separately OFL-licensed by JetBrains s.r.o. See the typography section above for source links.
