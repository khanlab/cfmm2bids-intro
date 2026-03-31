# cfmm2bids-intro
Introduction and overview of the cfmm2bids workflow for reproducible and automated BIDS conversion.

## Contents

- [`presentation.md`](presentation.md) — Marp-based slide deck covering the cfmm2bids workflow, using the Trident studies (`config/trident`) as examples.
- [`draft_tutorial.md`](draft_tutorial.md) — Outline / notes used to build the presentation.

## Viewing the Presentation

The presentation is written for [Marp](https://marp.app/), a Markdown Presentation Ecosystem.

### Option 1 — Marp CLI

```bash
# Install Marp CLI
npm install -g @marp-team/marp-cli

# Export to HTML (recommended for sharing)
marp presentation.md --output presentation.html

# Export to PDF
marp presentation.md --pdf --output presentation.pdf

# Live preview in browser
marp presentation.md --preview
```

### Option 2 — VS Code Extension

Install the [Marp for VS Code](https://marketplace.visualstudio.com/items?itemName=marp-team.marp-vscode) extension to preview and export slides directly from the editor.

## Links

- 🔗 cfmm2bids GitHub: <https://github.com/khanlab/cfmm2bids>
