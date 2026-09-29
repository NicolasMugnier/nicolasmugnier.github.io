# Mermaid

This theme initialises Mermaid on `.mermaid` only (`_includes/custom-head.html`, mermaid@10, `startOnLoad: true`).

Use:

```html
<div class="mermaid">
flowchart TD
    A[Step] --> B["Label with: colon"]
</div>
```

Never a fenced ` ```mermaid ` block. Kramdown turns that into `<pre><code>`, so it stays a code listing.

Quote node labels that contain colons or commas.

Match existing posts (sorts, scaling jobs): unindented `<div class="mermaid">`, then the graph, then `</div>`.
