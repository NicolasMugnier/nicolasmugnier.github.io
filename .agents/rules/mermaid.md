# Mermaid

GitHub renders fenced mermaid blocks. The blog does too: `_includes/custom-head.html` promotes `pre code.language-mermaid` to `.mermaid`, then `mermaid.run()`.

Use a fence in the markdown file:

    ```mermaid
    flowchart TD
        A[Step] --> B["Label with: colon"]
    ```

Do not use `<div class="mermaid">` for new diagrams. GitHub will not draw those.

Quote node labels that contain colons or commas. Close every parenthesis in sequence messages. A truncated line kills the whole graph.

Existing posts that still use `<div class="mermaid">` keep working.
