# Notes - HTML

## 1. HTML Basics & Document Structure

### <!DOCTYPE html> Declaration & Rendering Modes

- **Definition:** `<!DOCTYPE html>` declares that the document follows the **HTML5** standard.
- **Purpose:**
  - Informs the browser which HTML version the document uses.
  - Forces the browser to render the page in **standards mode** (consistent modern CSS and layout behavior).
  - Prevents **quirks mode** (legacy browser backward-compatibility rendering behavior).
- **Important Note:** `<!DOCTYPE html>` is **not** an HTML tag; it is a document type declaration instruction for the browser placed as the very first line of an HTML file.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Document</title>
</head>
<body>
</body>
</html>
```

---

### Generic Containers vs Semantic Sections (div vs section)

| Feature | `<div>` | `<section>` |
| :--- | :--- | :--- |
| **Type** | Non-semantic element | Semantic element |
| **Meaning** | Generic container | Logical, thematic section of a document |
| **Usage** | Layout, styling, visual grouping | Structuring related thematic content |
| **Accessibility** | No structural meaning for screen readers | Exposes landmark/document structure |
| **SEO** | No structural value | Helps search engines understand document outline |

- **Rule of Thumb:**
  - If content has thematic meaning and usually has a heading $	o$ use `<section>`.
  - If the element is used purely for layout wrappers or styling hooks $	o$ use `<div>`.

---

## 2. Document Object Model (DOM)

### Document Object Model (DOM) & Tree Structure

- **Definition:** The **DOM (Document Object Model)** is a tree-like in-memory representation of an HTML document created by the browser when parsing HTML.
- **Key Mechanics:**
  - The browser parses raw HTML markup and converts it into a hierarchical **DOM tree**.
  - Each HTML element, attribute, and text segment becomes an object (**node**) in the tree.
  - JavaScript interacts with, queries, and dynamically updates the rendered page through the DOM API.

#### Example HTML Page Structure as a DOM Tree:

```text
html
├── head
└── body
     ├── header
     ├── main
     │     ├── section
     │     └── article
     └── footer
```

---

### DOM Nodes & Node Types

- **Definition:** A **DOM node** is any individual object in the DOM tree created from an HTML document. Every part of the document becomes a node in the hierarchical tree structure.

#### Example HTML & Node Representation:

```html
<section>
    <h2>Title</h2>
    <p>Hello World</p>
</section>
```

```text
section  (element node)
├── h2  (element node)
│     └── "Title" (text node)
└── p   (element node)
      └── "Hello World" (text node)
```

#### Common Types of DOM Nodes:

| Node Type | Description / Example |
| :--- | :--- |
| **Element Node** | HTML tags such as `<div>`, `<p>`, `<section>` |
| **Text Node** | Text content inside elements (e.g., `"Title"`, `"Hello World"`) |
| **Attribute Node** | Element attributes such as `id`, `class`, `href` |

---

### DOM Hierarchy: Parent, Child & Sibling Relationships

| Term | Meaning |
| :--- | :--- |
| **Parent** | An element directly containing another element |
| **Child** | An element directly nested inside another element |
| **Sibling** | Elements that share the exact same parent element |

#### Example:
```text
section
├── article
└── aside
```
Here, `section` is the **parent**, `article` and `aside` are **children** of `section`, and `article` and `aside` are **siblings** to each other.

---

### JavaScript DOM Interaction & Manipulation

JavaScript accesses and dynamically manipulates the HTML document through the browser's `document` DOM interface:

```javascript
// Selecting elements from the DOM
const header = document.getElementById("main-header");
const card = document.querySelector(".card");

// Creating and modifying elements
const newParagraph = document.createElement("p");
newParagraph.textContent = "Updated content";
card.appendChild(newParagraph);
```

- **Capabilities enabled by DOM manipulation:**
  - Changing text and HTML content dynamically.
  - Modifying CSS styles and classes.
  - Registering and handling user events (`click`, `submit`, `input`).

---

## 3. Semantic HTML

### Semantic HTML Meaning & Purpose

- **Meaning of Semantic:** "Semantic" means having meaning or describing the purpose of something.
- In HTML, **semantic elements** explicitly describe *what* the content represents to both browsers and developers, rather than merely *how* it looks visually.

| Element | Semantic Meaning |
| :--- | :--- |
| `<nav>` | Navigation links |
| `<article>` | Self-contained, independent content |
| `<header>` | Introductory/top part of a page or section |

---

### Semantic HTML5 Elements & Use Cases

Common semantic HTML5 tags include `<header>`, `<footer>`, `<article>`, `<section>`, `<nav>`, `<main>`, and `<aside>`.

| Tag | Purpose | Typical Use Case |
| :--- | :--- | :--- |
| `<header>` | Top/introductory area of page or section | Site logo, page title, top navigation |
| `<footer>` | Bottom/closing area of page or section | Copyright notices, legal links, contact information |
| `<nav>` | Navigation section | Primary navigation menu, sidebar table of contents |
| `<main>` | Dominant main content of the `<body>` | Primary page content excluding repeated headers/footers |
| `<section>` | Thematic grouping of content | Content chapters or grouped blocks with headings |
| `<article>` | Self-contained, independently distributable content | Blog post, news article, forum post, product card |
| `<aside>` | Tangentially related secondary content | Sidebars, related links, callout boxes, advertisements |

---

### Importance of Semantic HTML (Accessibility, SEO, Maintainability)

| Reason | Technical & Architectural Explanation |
| :--- | :--- |
| **Accessibility** | Screen readers and assistive technologies parse semantic landmarks (`<nav>`, `<main>`, `<header>`) to help users navigate efficiently. |
| **SEO** | Search engine crawlers better understand content hierarchy, primary topics (`<main>`, `<article>`), and page structure. |
| **Maintainability** | Codebases are immediately readable and self-documenting compared to deeply nested "div soup". |
| **Better DOM Structure** | Produces a cleaner, logically meaningful document outline. |

---

## 4. HTML Forms & Validation

### HTML Form Attributes (action, method, name, autocomplete)

| Attribute | Purpose |
| :--- | :--- |
| `action` | Destination URL endpoint where the form data is submitted |
| `method` | HTTP method used to transmit data (`GET` or `POST`) |
| `name` | Identifier for the form element (used in scripts or form collections) |
| `autocomplete` | Controls whether the browser may autofill input values (`on` or `off`) |

---

### Form Submission Methods: GET vs POST

| Dimension | `GET` | `POST` |
| :--- | :--- | :--- |
| **Data Transport** | Appended as query parameters in the URL | Sent inside the HTTP request body |
| **Data Size Limit** | Limited by maximum URL length | Large payloads and binary files allowed |
| **Visibility** | Visible in browser address bar and history | Not visible in the URL |
| **Primary Use Case** | Data retrieval, search filters, bookmarkable queries | Data submission, mutations, sensitive payloads (passwords) |

---

### Form Labels & Accessibility (for Attribute)

- **Purpose of `<label>`:**
  - Explicitly associates descriptive text with a form control.
  - **Accessibility:** Screen readers announce the label text when the input receives focus.
  - **Usability:** Clicking the `<label>` automatically focuses (or toggles, for checkboxes/radios) the associated input field.
- **Linking via `for`:** The `<label>` element's `for` attribute must match the target input element's `id` attribute.

```html
<label for="email">Email</label>
<input id="email" name="email" type="email" required>
```

---

### Form Identifiers: id vs name Attributes

| Attribute | Purpose |
| :--- | :--- |
| `id` | Uniquely identifies the element within the client-side **DOM** (used by CSS `#id`, JavaScript `getElementById()`, and `<label for="...">`) |
| `name` | Defines the **key** in key-value pairs transmitted to the server when the form is submitted |

#### Example Form Submission Payload:
```text
name=Vinayak&email=test@example.com
```
*(If an `<input>` lacks a `name` attribute, its value is omitted from the submitted form data).*

---

### HTML5 Form Validation & Input Types

- **`required` Attribute:** A boolean attribute enforcing browser-level validation that prevents form submission until the user fills in the field.

#### Common HTML5 Input Types:

| Input `type` | Behavior & Use Case |
| :--- | :--- |
| `text` | Single-line plain text input (default) |
| `email` | Built-in browser syntax validation for email addresses |
| `password` | Masks entered characters for sensitive secrets |
| `number` | Numeric input with optional `min`, `max`, and `step` constraints |
| `radio` | Mutually exclusive single selection within a shared `name` group |
| `checkbox` | Independent boolean / multiple-choice selections |
| `file` | Opens OS file picker for file uploads |

---
