# Questions - HTML

### What is the purpose of `<!DOCTYPE html>`?

> **Direct Answer:**
> - `<!DOCTYPE html>` declares to the browser that the document conforms to the HTML5 specification.
> - It forces the browser to render the page in **standards mode**, preventing **quirks mode** (legacy backward-compatibility rendering behavior that breaks modern layouts).
>
> **Key Nuance / Trade-off:**
> - `<!DOCTYPE html>` is not an HTML tag; it is a mandatory document type declaration instruction placed at the very top of the HTML file.
>
> **Reference Concept:** `<!DOCTYPE html> Declaration & Rendering Modes`

---

### Difference between `div` and `section`?

> **Direct Answer:**
> - `<div>` is a non-semantic, generic container used purely for layout grouping and CSS styling hooks with no structural meaning for accessibility or SEO.
> - `<section>` is a semantic HTML5 element representing a logical, thematic grouping of content within a document, providing structural context to screen readers and search engines.
>
> **Key Nuance / Trade-off:**
> - Rule of thumb: if the grouped content has thematic meaning and typically contains a heading, use `<section>`; if the wrapper exists strictly for CSS layout or styling, use `<div>`.
>
> **Reference Concept:** `Generic Containers vs Semantic Sections (div vs section)`

---

### What is the DOM?

> **Direct Answer:**
> - The Document Object Model (DOM) is a hierarchical, tree-like in-memory representation of an HTML document constructed by the browser during parsing.
> - Every HTML element, attribute, and text block becomes an object (node) in the DOM tree, exposing a programmatic interface that JavaScript uses to read and dynamically modify the page.
>
> **Key Nuance / Trade-off:**
> - Manipulating the DOM directly triggers browser style recalculations and layout reflows; understanding the DOM tree helps developers minimize expensive DOM mutations.
>
> **Reference Concept:** `Document Object Model (DOM) & Tree Structure`

---

### What is a DOM node?

> **Direct Answer:**
> - A DOM node is any individual object within the DOM tree created when the browser parses an HTML document.
> - Common node types include **Element Nodes** (HTML tags like `<div>`, `<p>`, `<section>`), **Text Nodes** (text strings nested inside elements), and **Attribute Nodes** (`id`, `class`, `href`).
>
> **Key Nuance / Trade-off:**
> - Every part of an HTML document—even plain text inside tags—becomes a distinct node in the tree hierarchy, meaning JavaScript methods like `childNodes` return text nodes (including whitespace) whereas `children` returns only element nodes.
>
> **Reference Concept:** `DOM Nodes & Node Types`

---

### What are parent, child and sibling nodes?

> **Direct Answer:**
> - In the DOM hierarchy, a **Parent** is an element directly containing another element, a **Child** is an element directly nested inside another element, and **Siblings** are elements that share the same parent element.
> - For example, if `<section>` contains `<article>` and `<aside>`, then `<section>` is the parent, while `<article>` and `<aside>` are children of `<section>` and siblings to each other.
>
> **Key Nuance / Trade-off:**
> - DOM traversal APIs distinguish between direct children (`parentElement`, `children`) and arbitrary descendants (`querySelector`), which matters for both JavaScript traversal and CSS combinator performance.
>
> **Reference Concept:** `DOM Hierarchy: Parent, Child & Sibling Relationships`

---

### How does JavaScript interact with the DOM?

> **Direct Answer:**
> - JavaScript accesses and manipulates HTML documents through the browser's `document` DOM API using methods such as `document.getElementById()`, `document.querySelector()`, and `document.createElement()`.
> - Through these APIs, JavaScript dynamically updates text/HTML content, modifies inline styles or CSS classes, creates/removes elements, and attaches event listeners.
>
> **Key Nuance / Trade-off:**
> - Prefer `querySelector()` / `getElementById()` and batch DOM updates (e.g., via `DocumentFragment`) to avoid repeated layout thrashing during dynamic rendering.
>
> **Reference Concept:** `JavaScript DOM Interaction & Manipulation`

---

### What is the meaning of semantic?

> **Direct Answer:**
> - "Semantic" means having explicit meaning or describing the role and purpose of something.
> - In HTML, semantic elements describe *what* the enclosed content represents (e.g., `<nav>` for navigation links, `<article>` for independent content, `<header>` for introductory content), rather than just how it should look visually.
>
> **Key Nuance / Trade-off:**
> - Presentation should always be separated from meaning: use semantic tags for document meaning and CSS for visual appearance.
>
> **Reference Concept:** `Semantic HTML Meaning & Purpose`

---

### What are semantic HTML tags?

> **Direct Answer:**
> - Semantic HTML tags are elements that clearly communicate the role of their content to browsers, assistive technologies, and developers.
> - Core semantic tags include `<header>`, `<footer>`, `<nav>`, `<main>`, `<section>`, `<article>`, and `<aside>`.
>
> **Key Nuance / Trade-off:**
> - Use `<main>` once per document for primary unique content, `<article>` for self-contained distributable items (like blog posts), and `<aside>` for tangentially related content like sidebars.
>
> **Reference Concept:** `Semantic HTML5 Elements & Use Cases`

---

### Why is semantic HTML important?

> **Direct Answer:**
> - Semantic HTML is essential for four key reasons: **Accessibility** (screen readers rely on semantic landmarks to navigate pages), **SEO** (search engines infer content hierarchy and primary topics), **Maintainability** (self-documenting code over nested `<div>` tags), and a cleaner logical **DOM structure**.
>
> **Key Nuance / Trade-off:**
> - Native semantic elements (`<button>`, `<nav>`, `<main>`) provide built-in keyboard navigation and ARIA landmark roles out of the box without requiring extra JavaScript or manual ARIA overrides.
>
> **Reference Concept:** `Importance of Semantic HTML (Accessibility, SEO, Maintainability)`

---

### What are the important attributes of a form?

> **Direct Answer:**
> - The core attributes of an HTML `<form>` element are `action` (destination URL where form data is submitted), `method` (HTTP method such as `GET` or `POST`), `name` (identifier for the form), and `autocomplete` (controls browser autofill behavior).
>
> **Key Nuance / Trade-off:**
> - Modern Single Page Applications (SPAs) often intercept form submission via JavaScript (`event.preventDefault()`) and send JSON payloads asynchronously via `fetch`, rather than relying solely on native `action` page reloads.
>
> **Reference Concept:** `HTML Form Attributes (action, method, name, autocomplete)`

---

### Difference between GET and POST in forms?

> **Direct Answer:**
> - `GET` appends form data to the URL as query parameters, has strict size limits, exposes data in browser history/URLs, and is intended for idempotent data retrieval.
> - `POST` transmits form data inside the HTTP request body, supports large payloads and file uploads, keeps data out of the URL, and is intended for state-changing submissions.
>
> **Key Nuance / Trade-off:**
> - Sensitive credentials (passwords, tokens) must never be submitted via `GET` because URLs are logged in browser history, proxy servers, and web access logs.
>
> **Reference Concept:** `Form Submission Methods: GET vs POST`

---

### What is the purpose of `label` in forms?

> **Direct Answer:**
> - The `<label>` element binds a descriptive text caption to a specific form input, improving **accessibility** for screen readers and **usability** because clicking the label automatically focuses (or toggles) the linked input.
> - It is linked to the input using the `for` attribute matching the input's `id`.
>
> **Key Nuance / Trade-off:**
> - Using `placeholder` text as a replacement for `<label>` is an accessibility anti-pattern because placeholders disappear once typing starts and are not consistently announced by assistive technologies.
>
> **Reference Concept:** `Form Labels & Accessibility (for Attribute)`

---

### What is the purpose of the `for` attribute in `<label>`?

> **Direct Answer:**
> - The `for` attribute links a `<label>` explicitly to a specific form control by matching the target input element's `id` attribute (e.g., `<label for="email">` paired with `<input id="email">`).
> - This programmatic association allows screen readers to announce the label when the input is focused and lets users focus the input by clicking the label text.
>
> **Key Nuance / Trade-off:**
> - Because `for` binds to `id` (not `name`), every form control on a page linked via `for` must have a unique `id`.
>
> **Reference Concept:** `Form Labels & Accessibility (for Attribute)`

---

### Difference between `id` and `name` in forms?

> **Direct Answer:**
> - `id` uniquely identifies an element within the client-side **DOM** (used by CSS `#id` selectors, JavaScript `getElementById()`, and `<label for="...">`).
> - `name` specifies the **key** in the key-value pair sent to the backend server during form submission (e.g., `name=Vinayak&email=test@example.com`).
>
> **Key Nuance / Trade-off:**
> - An `<input>` without a `name` attribute will not be included in an HTML form's submitted payload; conversely, multiple radio buttons share the same `name` to form a mutually exclusive group while maintaining distinct `id`s for their labels.
>
> **Reference Concept:** `Form Identifiers: id vs name Attributes`

---

### What does `required` attribute do?

> **Direct Answer:**
> - The `required` attribute enforces browser-level validation, preventing the form from being submitted unless the user provides a non-empty value for that input field.
> - It works alongside HTML5 input types (`text`, `email`, `password`, `number`, `radio`, `checkbox`, `file`) to provide instant client-side feedback.
>
> **Key Nuance / Trade-off:**
> - Client-side HTML5 validation (`required`, `type="email"`) improves user experience, but can easily be bypassed via DevTools or direct HTTP requests—authoritative validation must always be enforced on the backend server.
>
> **Reference Concept:** `HTML5 Form Validation & Input Types`

---
