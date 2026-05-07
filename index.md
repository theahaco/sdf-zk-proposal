---
layout: default
title: ZK Proofs on Soroban — Recovery + Privacy Bundle
---

<div class="proposal-meta">
  $190k &middot; 14 weeks &middot; Tracks A + B &middot; May 2026
</div>

{% capture proposal %}{% include_relative SDF_PROPOSAL.md %}{% endcapture %}
{{ proposal | markdownify }}

<script type="module">
  import mermaid from "https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs";
  mermaid.initialize({
    startOnLoad: false,
    securityLevel: "loose",
    theme: "base",
    themeVariables: {
      fontFamily: "Inter, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif",
      fontSize: "13px",
      primaryColor: "#eef2f7",
      primaryBorderColor: "#155799",
      primaryTextColor: "#0b3d2e",
      lineColor: "#4a5568",
      tertiaryColor: "#fafbfc",
      actorBkg: "#155799",
      actorTextColor: "#ffffff",
      actorBorder: "#0f3a64",
      noteBkgColor: "#fef3c7",
      noteBorderColor: "#d97706",
      noteTextColor: "#451a03",
    },
    flowchart: { curve: "basis", nodeSpacing: 50, rankSpacing: 70, htmlLabels: true },
    sequence: { actorMargin: 60, boxMargin: 12, noteFontWeight: "500", messageFontWeight: "400" },
  });
  document.querySelectorAll("pre code.language-mermaid").forEach((block) => {
    const div = document.createElement("div");
    div.className = "mermaid";
    div.textContent = block.textContent;
    block.parentElement.replaceWith(div);
  });
  await mermaid.run();
</script>

<style>
  /* Inter for body, JetBrains Mono for code — loaded async-friendly via Google Fonts */
  @import url("https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=JetBrains+Mono:wght@400;500&display=swap");

  /* === Hero === */
  .page-header {
    background-image: linear-gradient(120deg, #0f3a64, #0b3d2e);
  }
  .proposal-meta {
    position: relative;
    z-index: 2;
    margin: -2.5rem auto 1.5rem;
    max-width: 760px;
    text-align: center;
    color: #b9d3e8;
    font-family: "Inter", system-ui, sans-serif;
    font-size: 0.78rem;
    letter-spacing: 0.14em;
    text-transform: uppercase;
  }

  /* === Body === */
  .main-content {
    max-width: 760px;
    margin: 0 auto;
    font-family: "Inter", -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  }
  .main-content p, .main-content li {
    line-height: 1.65;
    font-size: 1rem;
  }
  .main-content h1, .main-content h2, .main-content h3, .main-content h4 {
    font-family: "Inter", system-ui, sans-serif;
    font-weight: 600;
    letter-spacing: -0.01em;
    color: #0f1a2e;
  }
  .main-content h2 { border-bottom: 1px solid #e2e8f0; padding-bottom: 0.3em; }

  /* === Code & inline identifiers === */
  .main-content code {
    background: #f1f5f9;
    border: 1px solid #e2e8f0;
    border-radius: 4px;
    padding: 0.1em 0.4em;
    font-size: 0.86em;
    font-family: "JetBrains Mono", ui-monospace, SFMono-Regular, monospace;
    color: #0b3d2e;
  }
  .main-content pre {
    background: #f6f8fa;
    border: 1px solid #e2e8f0;
    border-radius: 6px;
  }
  .main-content pre code {
    background: transparent;
    border: 0;
    padding: 0;
    color: inherit;
    font-size: 0.85em;
  }

  /* === Tables === */
  .main-content table {
    display: block;
    overflow-x: auto;
    border-collapse: collapse;
    font-size: 0.92em;
    box-shadow: inset -8px 0 8px -8px rgba(0,0,0,0.12);
  }
  .main-content th {
    background: #f6f8fa;
    font-weight: 600;
    text-align: left;
  }
  .main-content th, .main-content td {
    padding: 0.5em 0.75em;
    border: 1px solid #e2e8f0;
    vertical-align: top;
  }

  /* === Mermaid diagrams === */
  .mermaid {
    display: flex;
    justify-content: center;
    margin: 1.5em 0;
    padding: 1.25em 1em;
    background: #fafbfc;
    border: 1px solid #e2e8f0;
    border-radius: 6px;
    overflow-x: auto;
  }
  .mermaid svg { max-width: 100%; height: auto; }

  /* === Mobile === */
  @media (max-width: 768px) {
    .main-content { font-size: 16px; padding: 1.2rem; }
    .main-content table { font-size: 0.84em; }
    .mermaid {
      padding: 0.5em;
      /* Allow horizontal scroll for wide diagrams; users can pinch-zoom too */
      overflow-x: scroll;
    }
    .mermaid svg { min-width: 700px; max-width: none; }
    .proposal-meta { font-size: 0.7rem; letter-spacing: 0.12em; }
  }

  /* === Links & misc === */
  .main-content a { color: #155799; }
  .main-content a:hover { color: #0b3d2e; }
  .main-content blockquote {
    border-left: 3px solid #155799;
    background: #f6f8fa;
    padding: 0.75em 1em;
    color: #2d3748;
  }
  hr { border: 0; border-top: 1px solid #e2e8f0; margin: 2.5em 0; }
</style>
