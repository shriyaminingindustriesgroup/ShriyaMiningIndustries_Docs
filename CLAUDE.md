# Shriya Mining Industries — Invoice Platform

This repo **is** the canonical architecture reference for the whole system — `ARCHITECTURE.md`
(source of truth) and `index.html` (diagram-first published view, via GitHub Pages at
https://shriyaminingindustriesgroup.github.io/ShriyaMiningIndustries_Docs/). The other two repos
are `ShriyaMiningIndustries_api` (C# backend, **private**) and `ShriyaMiningIndustries_UI` (Vue
frontend, private). This repo stays public and deliberately contains no secrets, so it can.

## This repo's role

If you're working here, you're almost always updating the documentation itself in response to a
real change made in one of the other two repos. Keep `ARCHITECTURE.md` and `index.html` in sync —
a change to one without the other is an incomplete update. Diagrams are Mermaid; keep them simple
(one clear mental model per diagram) rather than cramming every true fact into one dense flowchart
— that produced an unreadable mess of crossing lines once already, see git history.

## How we work together

- **Never put an actual secret value in this repo** — it's public. Name *where* a value lives (a
  GitHub secret, a named app setting), never the value itself.
- Confirm real costs/prices via the Azure Retail Prices API or official docs before writing a
  number into the cost section — never guess from memory.
- Verify facts (resource existence, config values, runtime behavior) via actual tool calls against
  the real repos/Azure resources before documenting them as true.
- Update this documentation in the same session as the architectural change it describes, not
  "later."
- When documenting something as "verified working," make sure it actually was — a real bug
  (migration run order) went undocumented-as-a-risk for a long time because the live databases
  were seeded rather than built from these scripts, so CI passing against them never actually
  exercised it. "CI is green" and "this was verified from a clean/empty state" are different
  claims; don't conflate them in what gets written here.
