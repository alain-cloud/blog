# Enable Mermaid diagrams in Jekyll

Create `_includes/mermaid.html` with:

```html
<script type="module">
  import mermaid from "https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs";
  mermaid.initialize({ startOnLoad: true, theme: "neutral" });
  document.querySelectorAll("pre code.language-mermaid").forEach((code) => {
    const pre = code.parentElement;
    const div = document.createElement("div");
    div.className = "mermaid";
    div.textContent = code.textContent;
    pre.replaceWith(div);
  });
</script>
```

Copy the Minima post layout into `_layouts/post.html`, then add this before the closing body or at the end of the layout:

```liquid
{% raw %}
{% include mermaid.html %}
{% endraw %}
```

Security note: pin a reviewed Mermaid version or host the asset yourself if your content security policy does not allow a public CDN. Test rendering in a branch before merging.
