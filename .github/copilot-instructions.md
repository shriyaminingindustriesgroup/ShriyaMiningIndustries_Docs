# Shriya Mining Industries — Invoice Platform

This repo **is** the canonical architecture reference for the whole system — `ARCHITECTURE.md`
(source of truth) and `index.html` (diagram-first published view, via GitHub Pages at
https://shriyaminingindustriesgroup.github.io/ShriyaMiningIndustries_Docs/). The other two repos
are `ShriyaMiningIndustries_api` (C# backend, private) and `ShriyaMiningIndustries_UI` (Vue
frontend, private). This repo stays public and deliberately contains no secrets, so it can.

If you're working here, you're almost always updating the documentation itself in response to a
real change made in one of the other two repos.

Working norms for this repo:
- Never put an actual secret value in this repo — it's public. Name *where* a value lives (a
  GitHub secret, a named app setting), never the value itself.
- Keep `ARCHITECTURE.md` and `index.html` in sync — a change to one without the other is an
  incomplete update.
- Diagrams are Mermaid; keep them simple (one clear mental model per diagram) rather than
  cramming every true fact into one dense flowchart.
- Confirm real costs/prices via official sources before writing a number into the cost section.
- "CI is green" and "this was verified from a clean/empty state" are different claims — don't
  conflate them in what gets written here (a real migration-ordering bug went undocumented as a
  risk for a while because of exactly this conflation).
