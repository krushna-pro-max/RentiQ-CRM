---
name: Luminous Insight
colors:
  surface: '#faf8ff'
  surface-dim: '#d2d9f4'
  surface-bright: '#faf8ff'
  surface-container-lowest: '#ffffff'
  surface-container-low: '#f2f3ff'
  surface-container: '#eaedff'
  surface-container-high: '#e2e7ff'
  surface-container-highest: '#dae2fd'
  on-surface: '#131b2e'
  on-surface-variant: '#484456'
  inverse-surface: '#283044'
  inverse-on-surface: '#eef0ff'
  outline: '#797487'
  outline-variant: '#c9c3d9'
  surface-tint: '#6036ed'
  primary: '#4a09d9'
  on-primary: '#ffffff'
  primary-container: '#6339f0'
  on-primary-container: '#e0d7ff'
  inverse-primary: '#cabeff'
  secondary: '#0059b9'
  on-secondary: '#ffffff'
  secondary-container: '#0e71e4'
  on-secondary-container: '#fefcff'
  tertiary: '#494854'
  on-tertiary: '#ffffff'
  tertiary-container: '#61606c'
  on-tertiary-container: '#dedbea'
  error: '#ba1a1a'
  on-error: '#ffffff'
  error-container: '#ffdad6'
  on-error-container: '#93000a'
  primary-fixed: '#e6deff'
  primary-fixed-dim: '#cabeff'
  on-primary-fixed: '#1c0062'
  on-primary-fixed-variant: '#4800d6'
  secondary-fixed: '#d7e2ff'
  secondary-fixed-dim: '#acc7ff'
  on-secondary-fixed: '#001a40'
  on-secondary-fixed-variant: '#004591'
  tertiary-fixed: '#e3e1ef'
  tertiary-fixed-dim: '#c7c5d3'
  on-tertiary-fixed: '#1b1b25'
  on-tertiary-fixed-variant: '#464651'
  background: '#faf8ff'
  on-background: '#131b2e'
  surface-variant: '#dae2fd'
typography:
  headline-xl:
    fontFamily: Hanken Grotesk
    fontSize: 48px
    fontWeight: '700'
    lineHeight: '1.1'
    letterSpacing: -0.02em
  headline-lg:
    fontFamily: Hanken Grotesk
    fontSize: 32px
    fontWeight: '600'
    lineHeight: '1.2'
    letterSpacing: -0.01em
  headline-lg-mobile:
    fontFamily: Hanken Grotesk
    fontSize: 24px
    fontWeight: '600'
    lineHeight: '1.2'
  headline-md:
    fontFamily: Hanken Grotesk
    fontSize: 20px
    fontWeight: '600'
    lineHeight: '1.4'
  body-lg:
    fontFamily: Inter
    fontSize: 18px
    fontWeight: '400'
    lineHeight: '1.6'
  body-md:
    fontFamily: Inter
    fontSize: 16px
    fontWeight: '400'
    lineHeight: '1.5'
  label-sm:
    fontFamily: Hanken Grotesk
    fontSize: 14px
    fontWeight: '500'
    lineHeight: '1.2'
    letterSpacing: 0.02em
  label-xs:
    fontFamily: Hanken Grotesk
    fontSize: 12px
    fontWeight: '600'
    lineHeight: '1'
rounded:
  sm: 0.25rem
  DEFAULT: 0.5rem
  md: 0.75rem
  lg: 1rem
  xl: 1.5rem
  full: 9999px
spacing:
  base: 8px
  xs: 4px
  sm: 12px
  md: 24px
  lg: 48px
  xl: 80px
  gutter: 24px
  margin-mobile: 16px
  margin-desktop: 64px
---

## Brand & Style

This design system is built on a foundation of **Corporate Modernism** infused with **Glassmorphism**. It is designed for high-end SaaS and AI-driven platforms that need to feel both technologically advanced and deeply human. 

The brand personality is visionary and reliable. It utilizes generous whitespace to convey clarity, while vibrant gradients and translucent overlays represent the "energy" of artificial intelligence. The visual mood is calm yet high-energy, achieving a sophisticated balance between professional utility and creative expression. 

Key stylistic markers include:
- **Depth through layering:** Extensive use of soft shadows and background blurs to create a multi-dimensional workspace.
- **Vibrant Accents:** Using high-chroma purples and blues to guide attention and highlight intelligence.
- **Sophisticated Precision:** Tight, intentional typography paired with spacious layouts.

## Colors

The color palette centers on **Electric Violet** and **Azure Blue**, representing the intersection of logic and innovation. 

- **Primary (Electric Violet):** Used for primary actions, branding, and to denote "AI" or "active" states.
- **Secondary (Azure Blue):** Supports the primary color in gradients and used for secondary highlights or informational callouts.
- **Tertiary (Soft Lavender):** A low-saturation tint used for large surface areas and background containers to reduce visual fatigue.
- **Neutral (Slate):** A deep, cool-toned grey used for typography to maintain high legibility without the harshness of pure black.

**Gradients:** Use linear gradients (45-degree angle) transitioning from Primary to Secondary for high-impact cards or buttons.

## Typography

This design system utilizes **Hanken Grotesk** for headings and labels to provide a sharp, contemporary edge that feels tech-forward. **Inter** is utilized for body text to ensure maximum readability and a systematic, clean feel.

- **Headlines:** Use tight letter spacing for larger sizes to create a "compact" and authoritative look.
- **Color usage:** Primary headlines should use the Neutral-Slate color. Sub-headlines or "kicker" text should utilize the Primary-Violet or a gradient for emphasis.
- **Hierarchy:** Ensure a clear distinction between levels by significantly varying font weights (e.g., Bold headlines vs. Regular body).

## Layout & Spacing

The layout follows a **Fluid Grid** model with high density within components but generous "breathing room" between sections.

- **Grid Model:** A 12-column system for desktop, 8-column for tablet, and 4-column for mobile.
- **Gutter Strategy:** Fixed 24px gutters provide structural consistency, while outer margins expand fluidly.
- **Spacing Rhythm:** Based on an 8px scale. Use `lg` (48px) and `xl` (80px) for vertical section spacing to maintain the "airy" premium feel.
- **Component Padding:** Internal card padding should typically be `md` (24px) to ensure content doesn't feel cramped against rounded edges.

## Elevation & Depth

Visual hierarchy is established through **Tonal Layers** and **Ambient Shadows**. This design system avoids harsh borders in favor of depth-based separation.

- **Surfaces:** Main content lives on level 0 (Background). Primary containers and cards live on level 1 (Surface).
- **Shadows:** Use extra-diffused, low-opacity shadows. For a level 1 card, use a shadow with a 20px blur, 4px Y-offset, and 5% opacity of the Primary color to "lift" the element without adding visual clutter.
- **Glassmorphism:** For overlays, modals, or "floating" UI elements, use a 70% opacity white fill with a 12px backdrop blur. This maintains context while providing a focused interaction area.

## Shapes

The shape language is **Rounded**, conveying friendliness and modern software aesthetics.

- **Base Radius:** 0.5rem (8px) for small components like buttons and inputs.
- **Card Radius:** `rounded-xl` (1.5rem / 24px) is the standard for main content containers and feature cards, creating a soft, pill-like frame for imagery and text.
- **Iconography:** Use a consistent 2px stroke weight with rounded caps and joins to match the outer container radius.

## Components

### Buttons
- **Primary:** Solid gradient (Violet to Blue) with white text. Roundedness should be 8px or fully pill-shaped depending on the context.
- **Secondary:** Ghost style with a thin Violet border or a Soft Lavender background.

### Cards
- Standard cards use the `rounded-xl` radius.
- Use a subtle 1px border (#E2E8F0) in addition to soft shadows to define the edge on white backgrounds.
- Feature cards can use a subtle background gradient or a large, blurred decorative element in the background.

### Input Fields
- Backgrounds should be slightly off-white (#F1F5F9) to distinguish them from the card surface.
- Focus states should use a 2px Violet glow (ring).

### Chips & Tags
- Used for status and categories. High-roundedness (pill) with light-tinted backgrounds (e.g., 10% opacity of the status color) and bold text.

### Progress & Connectors
- Use rounded, thick lines (4px+) for "flow" indicators. When connecting elements (like AI nodes), use a gradient stroke to show direction and energy.