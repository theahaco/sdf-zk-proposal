---
layout: default
title: ZK Proofs on Soroban — Recovery + Privacy Bundle
---

{% capture proposal %}{% include_relative SDF_PROPOSAL.md %}{% endcapture %}
{{ proposal | markdownify }}

<script type="module">
  import mermaid from "https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs";
  mermaid.initialize({ startOnLoad: false, theme: "default", securityLevel: "loose" });
  document.querySelectorAll("pre code.language-mermaid").forEach((block) => {
    const div = document.createElement("div");
    div.className = "mermaid";
    div.textContent = block.textContent;
    block.parentElement.replaceWith(div);
  });
  await mermaid.run();
</script>

<style>
  .mermaid { display: flex; justify-content: center; margin: 1.5em 0; }
  .mermaid svg { max-width: 100%; height: auto; }
  /* Tighten Cayman's wide line-height for tables */
  table { font-size: 0.92em; }
  /* Make code blocks readable */
  pre, code { font-size: 0.88em; }
</style>
