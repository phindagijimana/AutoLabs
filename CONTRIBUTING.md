# Contributing to AutoLabs

Thanks for being interested. **AutoLabs** is a layered handbook on autonomous labs and literature synthesis. It grows by accretion — small additions, corrections, and worked examples are exactly what makes it useful.

## Quick start

```bash
pip install mkdocs-material
mkdocs serve              # preview locally at http://127.0.0.1:8000
mkdocs build --strict     # build the site; --strict turns warnings into errors
```

## What good contributions look like

- **Pages stay focused.** A page does one thing. If it grows past ~800 lines it probably wants to split.
- **Pages respect the level.** Beginner pages stay in plain language; PhD pages can use math; engineer pages assume production context. If you're not sure which level a contribution belongs at, ask in the issue.
- **Concepts are anchored in something concrete.** When you introduce a term, give an example. The hippocampal-sclerosis case study is the running neuroscience example; if you contribute a section that uses a different modality or domain, please do the same with that domain.
- **Honest warnings stay.** Every chapter ends with a "honest warnings" section. Do not delete one; do extend it.
- **Stubs are fine.** A page with a clean scope statement and links to good external references is better than an empty file.

## Style

- Markdown is rendered with **MkDocs Material**. Admonitions (`!!! note`, `!!! tip`, `!!! warning`) are encouraged.
- Headings: `H1` is the page title; sub-sections use `H2` and below.
- Code blocks use language fences (` ```python `, ` ```bash `, ` ```sql `, etc.).
- Mermaid diagrams for any architecture; tables for any landscape comparison; bullet lists for short enumerations.
- Citations use the `[^slug]` footnote pattern with a `## References` section at the bottom of the page.
- Cross-references between chapters use relative links (`[Architecture](../engineer/architecture.md)`).

## Adding a new chapter

1. Pick the right level directory (`beginner/`, `intermediate/`, `phd/`, `engineer/`).
2. Add the file. Start with the scope statement: a single-sentence blockquote at the top.
3. Link from the level's `index.md`.
4. Add a `nav:` entry to `mkdocs.yml`.
5. Run `mkdocs build --strict` and fix any warnings.

## Filing issues

Use GitHub Issues. Helpful issue contents:

- Page link or section heading you're asking about.
- What's confusing, missing, or wrong.
- A suggestion (even a rough one) if you have it.

## License

By contributing, you agree your contribution is licensed under [MIT](LICENSE) — the same license as the rest of AutoLabs.
