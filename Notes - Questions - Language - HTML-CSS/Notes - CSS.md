# Notes - CSS

## 1. CSS Inclusion & Selectors

### Applying CSS: Inline, Internal & External Stylesheets

CSS can be applied to an HTML document in three ways:

| Method | Description | Example |
| :--- | :--- | :--- |
| **Inline CSS** | Applied directly inside an HTML element via the `style` attribute | `<p style="color:red;">Text</p>` |
| **Internal CSS** | Defined inside a `<style>` tag within the document `<head>` | `<style> p { color:red; } </style>` |
| **External CSS** | Written in a separate `.css` file and linked via `<link>` | `<link rel="stylesheet" href="styles.css">` |

#### Detailed Characteristics:
1. **Inline CSS:**
   - Applies strictly to that single element.
   - Difficult to maintain at scale and mixes structure with presentation.
   - Carries the **highest specificity** among standard CSS application methods.
2. **Internal CSS:**
   - Applies only to the single HTML page in which it is embedded.
   - Useful for small standalone pages or single-file prototypes.
3. **External CSS:**
   - **Industry Best Practice:** Reusable across multiple pages, cacheable by browsers, and cleanly separates styling from HTML markup for easy maintenance.

---

### CSS Selectors (Element, Class, ID, Descendant)

CSS selectors target HTML elements in the DOM to apply styling rules:

| Selector Type | Example | Target Behavior |
| :--- | :--- | :--- |
| **Element** | `p {}` | Targets all matching `<p>` elements |
| **Class** | `.card {}` | Targets all elements containing `class="card"` |
| **ID** | `#header {}` | Targets the unique element with `id="header"` |
| **Descendant** | `div p {}` | Targets all `<p>` elements nested anywhere inside a `<div>` |

---

## 2. Box Model & Display

### CSS Box Model (Content, Padding, Border, Margin)

The **CSS Box Model** defines how every HTML element is sized, spaced, and laid out on the page. Every box consists of four concentric layers from inside out:

`Content` $	o$ `Padding` $	o$ `Border` $	o$ `Margin`

```text
+---------------------------------------------------+
|                      Margin                       |
|   +-------------------------------------------+   |
|   |                  Border                   |   |
|   |   +-----------------------------------+   |   |
|   |   |              Padding              |   |   |
|   |   |   +---------------------------+   |   |   |
|   |   |   |          Content          |   |   |   |
|   |   |   +---------------------------+   |   |   |
|   |   +-----------------------------------+   |   |
|   +-------------------------------------------+   |
+---------------------------------------------------+
```

| Component | Purpose |
| :--- | :--- |
| **Content** | The inner text, image, or media rendered inside the element |
| **Padding** | Transparent inner spacing between the content and the border |
| **Border** | The visible or invisible boundary wrapping the padding and content |
| **Margin** | Transparent outer spacing separating the element from neighboring elements |

---

### Inspecting the Box Model in Browser DevTools

- To inspect and debug spacing issues on any live web page:
  - Open **Browser DevTools** (`F12` or Right-Click $	o$ **Inspect**) $	o$ select an element in the Elements/Inspector tab $	o$ view the **Box Model diagram** inside the Computed / Styles panel showing exact pixel dimensions for content, padding, border, and margin.

---

### Box Sizing: content-box vs border-box

The `box-sizing` CSS property controls how an element's declared `width` and `height` are calculated:

| `box-sizing` Value | Behavior |
| :--- | :--- |
| `content-box` | **Default CSS behavior:** Declared `width` and `height` apply *only* to the content area. Adding `padding` or `border` increases the total rendered outer box size. |
| `border-box` | Declared `width` and `height` **include** `content`, `padding`, and `border`, making responsive sizing predictable. |

```css
/* Best practice global reset for predictable layouts */
* {
    box-sizing: border-box;
}
```

---

### CSS Display Property (block, inline, inline-block, none, flex, grid)

The `display` property defines how an element is rendered visually and how it participates in the document layout flow:

| `display` Value | Layout Behavior |
| :--- | :--- |
| `block` | Starts on a new line and stretches across the full available width |
| `inline` | Sits inline with surrounding content and takes only required width (ignores explicit `width`/`height`) |
| `inline-block` | Flows inline like text, but respects explicit `width`, `height`, and vertical padding/margins |
| `none` | Completely hides the element and removes it from the document layout flow |
| `flex` | Turns the element into a 1D Flexbox layout container |
| `grid` | Turns the element into a 2D CSS Grid layout container |

---

## 3. Modern Layouts & Positioning

### Flexbox (1D Flexible Box Layout)

**Flexbox** is a **one-dimensional** layout system designed to align, distribute, and manage spacing between elements along either a single row or a single column.

#### Core Components:
| Component | Description |
| :--- | :--- |
| **Flex Container** | The parent element with `display: flex;` applied |
| **Flex Items** | Direct child elements inside the flex container |

#### Common Flexbox Properties:
| Property | Purpose |
| :--- | :--- |
| `flex-direction` | Sets the main axis (`row`, `row-reverse`, `column`, `column-reverse`) |
| `justify-content` | Aligns flex items along the **main axis** (horizontally in a row) |
| `align-items` | Aligns flex items along the **cross axis** (vertically in a row) |
| `gap` | Adds uniform spacing between flex items without margin hacks |

- **Common Use Cases:** Navigation bars, centering elements horizontally/vertically, card rows, and responsive toolbars.

---

### CSS Grid (2D Grid Layout)

**CSS Grid** is a **two-dimensional** layout system capable of arranging elements simultaneously across both **rows and columns**, offering precise structural alignment.

#### Core Components:
| Component | Description |
| :--- | :--- |
| **Grid Container** | The parent element with `display: grid;` applied |
| **Grid Items** | Direct child elements placed within grid tracks/cells |

#### Common Grid Properties:
| Property | Purpose |
| :--- | :--- |
| `grid-template-columns` | Defines column count and track sizes (e.g., `repeat(3, 1fr)`) |
| `grid-template-rows` | Defines row count and track heights |
| `gap` | Sets spacing (gutters) between rows and columns |
| `grid-area` | Assigns items to named grid areas or specific row/column spans |

- **Common Use Cases:** Full-page skeleton layouts, analytics dashboards, and complex multi-axis UI grids.

---

### CSS Positioning (static, relative, absolute, fixed, sticky)

The `position` property controls how an element is placed relative to the normal document flow, an ancestor element, or the browser viewport:

| `position` Value | Positioning Behavior |
| :--- | :--- |
| `static` | **Default:** Follows normal document flow (`top`, `left`, `z-index` have no effect) |
| `relative` | Offset relative to its original normal-flow position while preserving its original space |
| `absolute` | Removed from normal flow; positioned relative to its **nearest positioned ancestor** (an ancestor with `position` other than `static`) |
| `fixed` | Removed from normal flow; pinned relative to the **browser viewport** (stays fixed during scrolling) |
| `sticky` | Toggles between `relative` and `fixed` based on the user's scroll position until reaching a threshold (`top: 0`) |

- **Common Use Cases:** Tooltips (`absolute`), floating action buttons (`fixed`), sticky navigation headers (`sticky`), and modal overlays (`fixed`).

---

## 4. Responsive Design & Web Accessibility

### Responsive Web Design & Media Queries

**Responsive Web Design (RWD)** ensures a single web application adapts cleanly across desktops, tablets, and mobile devices without maintaining separate codebases.

| Technique | Purpose |
| :--- | :--- |
| **Flexible Layouts** | Containers resize dynamically relative to viewport width (Flexbox / Grid) |
| **Flexible Images** | Images scale proportionally within their parent containers (`max-width: 100%`) |
| **Media Queries** | Conditionally apply CSS rules based on viewport breakpoints or device capabilities |
| **Responsive Units** | Use relative units (`%`, `vw`, `vh`, `rem`, `em`) instead of rigid fixed pixels (`px`) |

#### Example Media Query:
```css
@media (max-width: 768px) {
    body {
        background-color: lightgray;
    }
}
```
*(Rules inside this media query activate when the viewport width is `768px` or narrower).*

- **Benefits:** Consistent layout behavior, mobile compatibility, and improved user experience across all device form factors.

---

### Web Accessibility (WCAG & Inclusive Practices)

**Web Accessibility (a11y)** is the practice of designing and engineering websites so people with disabilities—including users relying on screen readers, keyboard-only navigation, or assistive technologies—can perceive, operate, understand, and robustly interact with content.

| Accessibility Practice | Purpose |
| :--- | :--- |
| **Semantic HTML** | Provides structural landmarks (`<main>`, `<nav>`, `<header>`) for screen readers |
| **`alt` Attribute on Images** | Provides descriptive text equivalents for visually impaired users (`<img src="logo.png" alt="Company logo">`) |
| **Proper Form Labels** | Links `<label for="...">` to inputs so assistive technologies announce field purposes |
| **Keyboard Navigation** | Ensures all interactive elements are reachable and operable via `Tab` / `Enter` without a mouse |
| **ARIA Attributes** | Supplies supplemental accessibility state and roles (`aria-label`, `aria-expanded`) when native HTML semantics are insufficient |

- **Benefits:** Inclusive user experience, improved SEO, legal/standards compliance (**WCAG** — Web Content Accessibility Guidelines), and better usability for all users.

---
