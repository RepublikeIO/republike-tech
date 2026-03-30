# Republike Design System — Theme Reference

Extracted from Figma file: [Republike - WIP HABILIS](https://www.figma.com/design/pAxMQ77go1ftdFeokH6ZSV/Republike---WIP-HABILIS)

## Figma Structure

| Section | Contents |
|---------|----------|
| **Brand** | Typography scale, color palette swatches, logo variants |
| **Atoms** | Buttons (Primary/Secondary, Medium/Small, Default/Disabled/Line), Tags |
| **Molecules** | Composed UI patterns |
| **Organismes** | Full layout components |
| **Assets** | Icons (remixicon-based) |
| **iOS** | iOS-specific components (Status Bar, Keyboards, Alerts, etc.) |

---

## Colors

### Brand Colors (Tailwind classes)

| Token | Hex | Tailwind Class | Usage |
|-------|-----|----------------|-------|
| **Purple Main** | `#7B61FF` | `text-purple-main` / `bg-purple-main` | Primary brand color, CTA buttons, links, active states |
| **Purple Secondary** | `#8570F2` | `text-purple-secondary` / `bg-purple-secondary` | Secondary purple for hover/alternative |
| **Purple Light** | `#9055ED` | `text-purple-light` / `bg-purple-light` | Lighter purple accent |
| **Purple Background** | `#775EF7` | `bg-purple-background` | Purple tinted backgrounds |
| **Purple Soft** | `#CAC5E7` | `text-purple-soft` / `bg-purple-soft` | Muted purple for subtle UI |
| **Purple Extra Light** | `#E9E8FF` | `bg-purple-extralight` | Very light purple tints |
| **Black Main** | `#131721` | `text-black-main` / `bg-black-main` | Primary text, dark backgrounds |
| **Gray Main** | `#C7C8D0` | `text-gray-main` | Secondary/meta text |
| **Gray Text** | `#9597A1` | `text-gray-text` | Tertiary text, timestamps |
| **Gray Title** | `#E3E1E6` | `border-gray-title` | Borders, dividers |
| **Gray Background** | `#F9F9F9` | `bg-gray-background` | Page background |
| **Gray Post** | `#F5F5F5` | `bg-gray-post` | Post card background variant |
| **Gray Disabled** | `#DDDDDD` | `text-gray-disabled` | Disabled states |
| **Red Main** | `#FF0E83` | `text-red-main` | Moderation badges |
| **Red Secondary** | `#FF8686` | `text-red-secondary` | Soft warnings |
| **Pink Main** | `#FF6161` | `text-pink-main` | Alerts, errors |
| **Pink Error** | `#FF0062` | `text-pink-error` | Error states |
| **Pink Secondary** | `#E2DDFF` | `bg-pink-secondary` | Light purple-pink backgrounds (translation button, badges) |
| **Consul Gold** | `#FBDF9C` | `bg-consul` | Founding Consul badge |
| **Consul Foreground** | `#F0CB66` | `text-consul-foreground` | Consul text/border |

### Semantic Colors (CSS Variables — HSL)

| Token | Light Mode | Dark Mode | Tailwind Class |
|-------|-----------|-----------|----------------|
| `--background` | `0 0% 97.6%` (#F9F9F9) | `222.2 84% 4.9%` (#080C14) | `bg-background` |
| `--foreground` | `222.2 84% 4.9%` (#080C14) | `210 40% 98%` (#F5F8FA) | `text-foreground` |
| `--primary` | `249 100% 69%` (#7B61FF) | `210 40% 98%` (#F5F8FA) | `bg-primary` / `text-primary` |
| `--primary-foreground` | `210 40% 98%` | `222.2 47.4% 11.2%` | `text-primary-foreground` |
| `--secondary` | `246.32 82.61% 95.49%` (#EDEBFE) | `217.2 32.6% 17.5%` | `bg-secondary` |
| `--secondary-foreground` | `249 100% 69%` (#7B61FF) | `210 40% 98%` | `text-secondary-foreground` |
| `--muted` | `210 40% 96.1%` | `217.2 32.6% 17.5%` | `bg-muted` / `text-muted-foreground` |
| `--destructive` | `0 84.2% 60.2%` (#EF4444) | `0 62.8% 30.6%` | `bg-destructive` |
| `--border` | `214.3 31.8% 91.4%` (#E2E8F0) | `217.2 32.6% 17.5%` | `border-border` |
| `--card` | `0 0% 100%` (#FFFFFF) | `222.2 84% 4.9%` | `bg-card` |
| `--ring` | `215 20.2% 65.1%` | `217.2 32.6% 17.5%` | `ring-ring` |
| `--radius` | `0.5rem` | `0.5rem` | `rounded-lg/md/sm` |

---

## Typography

### Font Families

| Token | Font | Tailwind Class | Usage |
|-------|------|----------------|-------|
| **Main** | TT Firs Neue | `font-main` | All body text, headings, UI |
| **Secondary** | Replica | `font-secondary` | Special display text |
| **Light Replica** | Replica Light | `font-lightReplica` | Light weight display |

### Font Weights (TT Firs Neue)

| Weight | CSS Value | File |
|--------|-----------|------|
| Light | 300 | TTFirsNeue-Light.woff |
| Medium (Regular) | 400 | TTFirsNeue-Medium.woff |
| DemiBold (Semi Bold) | 600 | TTFirsNeue-DemiBold.woff |
| Bold | 700 | TTFirsNeue-Bold.woff |

### Type Scale (from Figma Brand section)

> **Note:** Figma uses "Graphik Trial" for Heading 1 and "SF Pro" for all other styles.
> In the webapp, all text uses **TT Firs Neue** (`font-main`). Map Figma weights as:
> Regular (400) → `font-normal`, Medium (510) → `font-medium`, Semibold (590/600) → `font-semibold`

| Style | Font Size | Line Height | Letter Spacing | Weight | Usage |
|-------|-----------|-------------|----------------|--------|-------|
| **Heading 1** | 32px | 40px | -0.16px | Semibold (600) | Page titles |
| **Heading 2 - Semi Bold** | 24px | 32px | -0.12px | Semibold (590) | Section titles |
| **Heading 2 - Regular** | 24px | 32px | -0.12px | Regular (400) | Section subtitles |
| **Heading 3 - Semi Bold** | 18px | 26px | -0.045px | Medium (510) | Card titles |
| **Heading 3 - Regular** | 18px | 26px | -0.045px | Regular (400) | Card subtitles |
| **Body 1 - Semi Bold** | 15px | 24px | -0.225px | Medium (510) | Emphasized body text |
| **Body 1 - Regular** | 15px | 24px | -0.225px | Regular (400) | Primary body text |
| **Body 2 - Semi Bold** | 15px | 21px | -0.225px | Medium (510) | Secondary emphasized text |
| **Body 2 - Regular** | 15px | 21px | -0.225px | Regular (400) | Secondary body text |
| **CTA Medium** | 15px | 16px | -0.225px | Medium (510) | Button labels (medium) |
| **CTA Small** | 12px | 16px | 0.12px | Semibold (590) | Button labels (small) |
| **Caption Medium** | 12px | 18px | 0.25px | Medium (510) | Captions, labels |
| **Caption Regular** | 12px | 18px | 0px | Regular (400) | Meta text, timestamps |

### Custom Font Size

| Token | Value | Tailwind Class |
|-------|-------|----------------|
| sx | 0.8125rem (13px) | `text-sx` |

---

## Shadows

| Token | Value | Tailwind Class | Usage |
|-------|-------|----------------|-------|
| **RPB Shadow** | `0px 0px 20px rgba(0,0,0,0.05)` | `shadow-rpb` | Post cards, elevated surfaces |
| **RPB Shadow 2** | `0px 0px 3px rgba(0,0,0,0.15)` | `shadow-rpb2` | Subtle elevation (avatars, small cards) |

---

## Border Radius

| Token | Value | Tailwind Class |
|-------|-------|----------------|
| Large | `0.5rem` (8px) | `rounded-lg` |
| Medium | `calc(0.5rem - 2px)` (6px) | `rounded-md` |
| Small | `calc(0.5rem - 4px)` (4px) | `rounded-sm` |
| Post cards | `0.75rem` (12px) | `rounded-xl` |

---

## Animations

| Token | Value | Tailwind Class | Usage |
|-------|-------|----------------|-------|
| Slide In | `translateY(-10px) → 0, opacity 0→1, 0.5s ease-out` | `animate-slideIn` | Dropdown menus, notifications |
| Scale In | `scale(0.85) → 1, opacity 0→1, 0.5s ease-out` | `animate-scaleIn` | Modals, popovers |
| Accordion Down | Height `0 → auto, 0.2s ease-out` | `animate-accordion-down` | Expandable sections |
| Accordion Up | Height `auto → 0, 0.2s ease-out` | `animate-accordion-up` | Collapsible sections |

---

## Button Variants (from Figma Atoms)

| Type | Size | State | Description |
|------|------|-------|-------------|
| **Primary** | Medium / Small | Default, Disabled, Line | Solid purple background, white text |
| **Secondary** | Medium / Small | Default, Disabled, Line | Outlined/ghost, purple text |
| Both | Both | Icon Only | Square button with icon, no text |

---

## Logo Variants (from Figma Brand)

| Variant | Description |
|---------|-------------|
| **Logo Mark** | Full horizontal wordmark |
| **Logo Symbol** | Square icon only (63x63) |
| **Variant 3** | Stacked/alternative layout |

---

## Icons

The app uses **Remix Icon** (`react-icons/ri`) for web and a custom **RepublIcon** font for mobile. Icons referenced in Figma Assets section follow the `icons/{name}` naming pattern (e.g., `icons/body-scan-fill`, `icons/palette-line`).

---

## File References

| File | Purpose |
|------|---------|
| `tailwind.config.js` | Color tokens, font families, border radius, animations |
| `src/styles/globals.css` | CSS variables (light/dark mode), shadows, base styles |
| `src/constants/font.constant.ts` | Local font loading (TT Firs Neue, Replica) |
| `src/lib/tailwindTheme.ts` | Runtime access to resolved Tailwind theme |
