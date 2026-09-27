# Enkilabs Design System
## Enkihost-Inspired White & Orange Aesthetic + Ubuntu Typography

Design system for **Enkilabs**, aligned with [Enkihost.com](https://www.enkihost.com)'s modern visual identity: clean light surfaces, vibrant orange branding, and Ubuntu typography.

---

### 🎨 Color Palette

| Token | Hex / Value | Usage |
|---|---|---|
| **Background Body** | `#f5f5f7` | Global canvas background (clean Apple/Stripe-like light gray) |
| **Surface (Cards/Nav)** | `#ffffff` | Elevated cards, navigation bar, post containers |
| **Surface Subtle** | `#fafafa` | Subtle icon backgrounds, table headers |
| **Orange Primary** | `#ea580c` (Tailwind `orange-600`) | Main brand accent, CTAs, logo box, active states |
| **Orange Hover** | `#c2410c` (Tailwind `orange-700`) | Button hover, active link state |
| **Orange Light** | `#fff7ed` (Tailwind `orange-50`) | Tag backgrounds, hero pill badge, blockquotes |
| **Orange Border** | `#fed7aa` (Tailwind `orange-200`) | Tag borders, link underlines, subtle accent borders |
| **Orange Accent** | `#f97316` (Tailwind `orange-500`) | Text selection highlight, pulse indicators |
| **Border Subtle** | `#e4e4e7` (Tailwind `zinc-200`) | Card borders, dividers, header border |
| **Border Hover** | `#fdba74` (Tailwind `orange-300`) | Card hover border glow |
| **Text Heading** | `#09090b` (Tailwind `zinc-950`) | Primary titles and section headers |
| **Text Body** | `#27272a` (Tailwind `zinc-800`) | Article paragraphs, copy, content lists |
| **Text Secondary** | `#52525b` (Tailwind `zinc-600`) | Descriptions, subtitles, navigation items |
| **Text Muted** | `#71717a` (Tailwind `zinc-500`) | Dates, metadata, footer notes |

---

### 🔤 Typography

1. **Primary Font**: `Ubuntu` (Google Fonts: 300, 400, 500, 700, 800)
   - Fallback: `-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif`
   - Used for all titles, body text, buttons, and navigation.
2. **Code & Terminal Font**: `Ubuntu Mono` (Google Fonts: 400, 700)
   - Fallback: `ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace`
   - Used for code blocks, inline code, terminal outputs, and hashtags.

---

### 🧱 Component Blueprint

#### 1. Header (`.site-header`)
- **Sticky top** with `backdrop-filter: blur(12px)` and `border-bottom: 1px solid #e4e4e7`.
- **Brand Logo**:
  - 36x36px rounded square in `#ea580c` with white lab icon and soft shadow (`rgba(234, 88, 12, 0.28)`).
  - Uppercase title `ENKILABS` in `Ubuntu` font, weight 800, color `#09090b`.
- **Navigation Links**:
  - `font-weight: 600; color: #52525b;` -> hover `color: #ea580c; background: rgba(234, 88, 12, 0.06);`.
  - External CTA link to Enkihost with orange pill styling (`.nav-cta-link`).

#### 2. Hero Section (`.hero-section`)
- **Top Pill Badge**: `#fff7ed` with `#fed7aa` border and animated orange pulsing dot.
- **Hero Title**: Bold 2.8rem Ubuntu title with orange underlined accent (`.highlight-orange`).
- **CTAs**: Primary orange button (`.btn-primary`) and secondary bordered button (`.btn-secondary`).

#### 3. Cards (`.project-card`, `.post-list li`, `article.post`)
- **Background**: `#ffffff`
- **Border**: `1px solid #e4e4e7` (zinc-200)
- **Radius**: `16px`
- **Shadow**: `box-shadow: 0 1px 3px rgba(0, 0, 0, 0.03)`
- **Hover**: Subtle lift (`translateY(-2px)`), border shifts to `#fdba74`, and soft orange ambient shadow (`0 10px 25px -5px rgba(234, 88, 12, 0.08)`).

#### 4. Code Blocks & Terminal
- **Pre / Highlight**: Dark terminal theme `#09090b` with border `#27272a`, matching Enkihost's build log widget.
- **Inline Code**: `#fff7ed` background with `#c2410c` text and `#fed7aa` border.

#### 5. Footer (`.site-footer`)
- `#ffffff` background with `#e4e4e7` top border.
- Enkilabs mark with copyright notice and direct links to Blog, About, Enkihost, and GitHub.
