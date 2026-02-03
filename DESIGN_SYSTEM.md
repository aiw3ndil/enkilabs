# Enkilabs Design System
## Hacker Aesthetic + Glassmorphism UI

### 🎨 Color Palette

| Color | Hex | Usage |
|-------|-----|-------|
| **Background Primary** | `#212240` | Main background (dark, almost black blue) |
| **Background Secondary** | `#1a1a2e` | Secondary backgrounds |
| **Accent Cyan** | `#0BB1D3` | Primary action elements, links, accents |
| **Accent Purple** | `#9F2FFF` | Gradient endpoint for buttons |
| **Accent Blue** | `#0051B3` | Trust and navigation elements |
| **Text Primary** | `#ffffff` | Main text on dark backgrounds |
| **Text Secondary** | `#717285` | Secondary text, metadata, labels |
| **Alert Pink** | `#FF419C` | Critical notifications & security alerts |
| **Grid Color** | `rgba(11, 177, 211, 0.1)` | Subtle hacker grid background |

### 🎯 Key Features

#### 1. **Glassmorphism**
- Semi-transparent frosted glass effect on cards and components
- `backdrop-filter: blur(10px)` for depth and layering
- Subtle borders with cyan glow (`rgba(11, 177, 211, 0.15)`)

#### 2. **Gradient Elements**
- **Button Gradient**: `89.8deg` from Cyan to Purple
  ```
  linear-gradient(89.8deg, #0BB1D3 0%, #9F2FFF 100%)
  ```
- Applied to CTAs, titles, and interactive elements

#### 3. **Hacker Aesthetic**
- Grid background pattern inspired by "Game of Life"
- Animated grid using repeating linear gradients
- Cyan glow effects on hover and focus states

#### 4. **Animations**
- `float`: Subtle floating animation on hero titles (3s duration)
- `grid-pulse`: Pulsing opacity on background grid (2s duration)
- Smooth transitions on all interactive elements (0.3s)

### 📱 Component Styles

#### Header (.site-header)
- Glassmorphic background: `rgba(26, 26, 46, 0.4)`
- Blur filter: 10px
- Gradient border with cyan/purple accents

#### Buttons & CTAs
- **Gradient**: Cyan to Purple at 89.8°
- **Padding**: 12px 24px
- **Border-radius**: 12px
- **Hover Effect**: Lift animation + shadow enhancement
- **Active Effect**: Return to base position

#### Page Content Cards
- **Background**: `rgba(26, 26, 46, 0.3)` with blur
- **Border**: Cyan-tinted with 0.15 opacity
- **Border-radius**: 20px
- **Shadow**: Cyan glow: `0 8px 32px rgba(15, 177, 211, 0.1)`

#### Code Blocks
- **Background**: `rgba(26, 26, 46, 0.6)`
- **Text Color**: Accent cyan (`#0BB1D3`)
- **Font**: Courier New, monospace
- **Border**: Cyan subtle 1px

#### Form Inputs
- **Background**: `rgba(26, 26, 46, 0.4)` with blur
- **Border**: Cyan on focus
- **Shadow on focus**: Cyan glow `0 0 15px rgba(11, 177, 211, 0.3)`

### 🔗 Links & Hover States
- **Default**: Cyan
- **Hover**: Purple with text-shadow glow
- **Underline Gradient**: Animated from left to right on nav links

### 📐 Responsive Design
- **Desktop**: Full styling with all effects
- **Mobile** (≤800px): Reduced padding, adjusted font sizes

### ✨ Special Effects

1. **Title Glow**: Gradient text with floating animation
2. **Interactive Elevation**: Cards lift on hover
3. **Neon Borders**: Cyan/purple glowing effects
4. **Smooth Transitions**: All interactions: 0.3s ease

---

**Design Philosophy**: Modern hacker aesthetic meets premium glassmorphism, creating a tech-forward, trustworthy UI that stands out while maintaining readability and accessibility.
