# Questions - CSS

### What are the different ways to apply CSS?

> **Direct Answer:**
> - CSS can be applied to HTML in three ways: **Inline CSS** (via the `style` attribute directly on an element), **Internal CSS** (inside a `<style>` block within `<head>`), and **External CSS** (defined in a standalone `.css` file linked via `<link rel="stylesheet" href="styles.css">`).
> - **External CSS** is the industry best practice because it separates presentation from structure, enables reuse across multiple pages, and improves maintainability.
>
> **Key Nuance / Trade-off:**
> - Inline CSS has higher specificity than external or internal stylesheets and applies only to a single element, making large applications hard to maintain. External stylesheets can also be cached by the browser across page navigations.
>
> **Reference Concept:** `Applying CSS: Inline, Internal & External Stylesheets`

---

### What are CSS selectors?

> **Direct Answer:**
> - CSS selectors are patterns used to match and target HTML elements in the DOM so styling rules can be applied to them.
> - Fundamental selector types include **Element** (`p {}`), **Class** (`.card {}`), **ID** (`#header {}`), and **Descendant** (`div p {}`).
>
> **Key Nuance / Trade-off:**
> - Class selectors (`.card`) are preferred over ID selectors (`#header`) for styling because classes are reusable and keep specificity low, preventing specificity wars in large stylesheets.
>
> **Reference Concept:** `CSS Selectors (Element, Class, ID, Descendant)`

---

### What is the CSS Box Model?

> **Direct Answer:**
> - The CSS Box Model defines the rectangular layout and spacing structure of every HTML element, composed of four concentric layers from inside out: **Content** $\to$ **Padding** $\to$ **Border** $\to$ **Margin**.
> - Content holds the actual text or image, padding adds inner space around the content, border wraps the padding, and margin creates outer space separating the element from neighbors.
>
> **Key Nuance / Trade-off:**
> - Vertical margins between adjacent block elements can undergo "margin collapsing," whereas padding never collapses and expands the element's background area.
>
> **Reference Concept:** `CSS Box Model (Content, Padding, Border, Margin)`

---

### How to inspect the Box Model?

> **Direct Answer:**
> - You inspect the Box Model using **Browser DevTools**: right-click an element $\to$ click **Inspect** $\to$ open the **Computed / Styles** panel to view the interactive visual Box Model diagram displaying exact pixel measurements for content, padding, border, and margin.
>
> **Key Nuance / Trade-off:**
> - Hovering over each layer inside the DevTools Box Model diagram highlights the exact corresponding region directly on the rendered viewport, making layout debugging immediate.
>
> **Reference Concept:** `Inspecting the Box Model in Browser DevTools`

---

### What is `box-sizing`?

> **Direct Answer:**
> - The `box-sizing` property controls how an element's total rendered width and height are calculated.
> - `content-box` (default) applies `width`/`height` only to the content, meaning padding and border add to total size; `border-box` includes content, padding, and border within the declared `width` and `height`.
>
> **Key Nuance / Trade-off:**
> - Applying `* { box-sizing: border-box; }` globally is a standard CSS reset practice because it prevents padding and borders from unexpectedly breaking multi-column percentage layouts.
>
> **Reference Concept:** `Box Sizing: content-box vs border-box`

---

### What is the CSS `display` property?

> **Direct Answer:**
> - The `display` property controls how an element is rendered visually and how it participates in the document layout flow.
> - Core values include `block` (full width, starts on a new line), `inline` (shrinks to content, stays inline), `inline-block` (stays inline while accepting `width`/`height`), `none` (hides element and removes it from layout), `flex` (1D flex container), and `grid` (2D grid container).
>
> **Key Nuance / Trade-off:**
> - Unlike `visibility: hidden` (which hides an element visually while preserving its empty space in the layout), `display: none` removes the element completely from layout calculations.
>
> **Reference Concept:** `CSS Display Property (block, inline, inline-block, none, flex, grid)`

---

### What is Flexbox?

> **Direct Answer:**
> - Flexbox (Flexible Box Layout) is a **one-dimensional** CSS layout model used to align, distribute, and space items along a single row or column.
> - It operates via a parent **Flex Container** (`display: flex`) and child **Flex Items**, controlled by properties such as `flex-direction`, `justify-content` (main-axis alignment), `align-items` (cross-axis alignment), and `gap`.
>
> **Key Nuance / Trade-off:**
> - Flexbox excels at single-axis alignment (e.g., navigation bars, centering items, dynamic wrapping cards), whereas two-dimensional page skeletons requiring simultaneous row and column alignment are better suited for CSS Grid.
>
> **Reference Concept:** `Flexbox (1D Flexible Box Layout)`

---

### What is CSS Grid?

> **Direct Answer:**
> - CSS Grid is a **two-dimensional** layout system that arranges elements into both **rows and columns** simultaneously for precise page-level and component-level layouts.
> - A parent **Grid Container** (`display: grid`) defines tracks using `grid-template-columns`, `grid-template-rows`, `gap`, and `grid-area` to place child **Grid Items**.
>
> **Key Nuance / Trade-off:**
> - Flexbox and CSS Grid are complementary rather than mutually exclusive: use Grid for overall macro dashboard/page structure and Flexbox inside individual grid cells for micro alignment.
>
> **Reference Concept:** `CSS Grid (2D Grid Layout)`

---

### What is CSS Positioning?

> **Direct Answer:**
> - CSS positioning (`position` property) determines how an element is placed in the document relative to normal flow, an ancestor, or the browser viewport.
> - Standard values include `static` (default flow), `relative` (offset from its normal position), `absolute` (positioned relative to its nearest positioned ancestor), `fixed` (pinned to the browser viewport), and `sticky` (switches between `relative` and `fixed` upon scrolling).
>
> **Key Nuance / Trade-off:**
> - An `absolute` child aligns to the nearest ancestor whose `position` is not `static` (commonly `position: relative` on the parent); if none exists, it falls back to the initial containing block (viewport).
>
> **Reference Concept:** `CSS Positioning (static, relative, absolute, fixed, sticky)`

---

### What is Responsive Design?

> **Direct Answer:**
> - Responsive Design is a web engineering approach ensuring a single webpage adapts gracefully across desktops, tablets, and mobile screens without building separate sites.
> - It relies on **flexible layouts** (Flexbox/Grid), **flexible images**, **CSS media queries** (e.g., `@media (max-width: 768px)`), and **responsive relative units** (`%`, `vw`, `vh`, `rem`).
>
> **Key Nuance / Trade-off:**
> - Designing mobile-first (`min-width` media queries) progressively enhances layouts from small screens upward, typically yielding cleaner CSS and lighter mobile payloads than desktop-first overrides.
>
> **Reference Concept:** `Responsive Web Design & Media Queries`

---

### What is Web Accessibility?

> **Direct Answer:**
> - Web Accessibility (a11y) is the practice of designing and coding websites so people with disabilities—including screen reader and keyboard-only users—can perceive, operate, understand, and robustly interact with content.
> - Core practices include **Semantic HTML**, descriptive `alt` attributes on images, explicit `<label for="...">` form bindings, full keyboard navigation, and appropriate **ARIA** attributes following **WCAG** standards.
>
> **Key Nuance / Trade-off:**
> - Always prefer native semantic HTML controls (`<button>`, `<input>`, `<nav>`) over clickable `<div>` elements with custom ARIA roles, because native elements include keyboard focus, Enter/Space activation, and screen reader semantics automatically.
>
> **Reference Concept:** `Web Accessibility (WCAG & Inclusive Practices)`

---
