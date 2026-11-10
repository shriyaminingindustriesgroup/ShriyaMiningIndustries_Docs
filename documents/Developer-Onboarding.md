# Welcome — Developer Onboarding

This is a short, practical guide to getting set up on the Shriya Mining Industries invoice
platform and shipping your first change, including how to work with AI coding assistants
(Claude Code or GitHub Copilot) here. For the full architecture — every Azure resource, both
CI/CD pipelines, the database, auth model, and a debugging playbook — see
**[ARCHITECTURE.md](../ARCHITECTURE.md)** (or the [published page](https://shriyaminingindustriesgroup.github.io/ShriyaMiningIndustries_Docs/)).
This guide is the quick path to your first commit; that one is the deep reference you'll come
back to.

## What you're working on

A web app for managing export invoices: create/edit invoices, import purchase orders from Excel,
generate Excel/PDF/print exports, manage company profile data, administer users and roles.
Backend is C# (.NET, Azure Functions). Frontend is Vue 3. They're two separate repos, two
separate pipelines, meeting only at the HTTP boundary.

## Step 1 — Get access

Both repos are private. Ask whoever's managing the project to add you as a collaborator on:
- `shriyaminingindustriesgroup/ShriyaMiningIndustries_api` (the backend)
- `shriyaminingindustriesgroup/ShriyaMiningIndustries_UI` (the frontend)

You'll likely need both, even if your first task is on just one side — most features touch both.

## Step 2 — Install prerequisites

| Tool | Why |
|---|---|
| .NET 10 SDK | Build/run the API |
| Node.js (for `npm`) | Build/run the UI |
| Azure Functions Core Tools v4 (`npm install -g azure-functions-core-tools@4`) | Run the API locally |
| Azurite (`npm install -g azurite`) | Local storage emulator the API needs |
| Docker Desktop | Recommended — gives you a fully isolated local database, nothing to request from anyone |
| Git | Obviously |

## Step 3 — Clone and set up locally

```bash
git clone https://github.com/shriyaminingindustriesgroup/ShriyaMiningIndustries_api.git
git clone https://github.com/shriyaminingindustriesgroup/ShriyaMiningIndustries_UI.git
```

Each repo's own `README.md` has a self-contained quick start. In short, for the API:

```bash
cd ShriyaMiningIndustries_api/local-dev
docker compose up -d          # starts a local SQL Server, isolated to just your machine

# create the database (any SQL client, localhost,1433, login sa, password LocalDevOnly!2026)
#   CREATE DATABASE ShriyaMiningIndustriesLocal

# apply the schema
cd ..
dotnet run --project src/ShriyaMiningIndustries.Migrations --configuration Release -- apply "Server=tcp:localhost,1433;Initial Catalog=ShriyaMiningIndustriesLocal;User ID=sa;Password=LocalDevOnly!2026;TrustServerCertificate=True;Encrypt=False;" database/migrations

# copy the settings template (already pre-filled for the setup above)
cd src/ShriyaMiningIndustries.Api
cp local.settings.json.example local.settings.json

# run it
dotnet run --configuration Release
```

Open **http://localhost:7071/api/swagger/ui** — every endpoint is listed with an editable sample
request and a "Try it out" button that hits your real local API.

For the UI, in a separate terminal:

```bash
cd ShriyaMiningIndustries_UI
npm install
npm run dev
```

Open **http://localhost:3000** — it's already configured to talk to your local API on port 7071.

Make sure Azurite is running too (`azurite` in its own terminal — the API needs it).

## Step 4 — Start a branch, off `test`

```bash
git checkout test
git pull
git checkout -b feature/short-description-of-what-youre-doing
```

`test` is the real integration branch — a validated, deployed environment, not just a label.
Don't branch off `main` directly.

## Step 5 — Work with an AI assistant

Both repos are already set up so Claude Code and GitHub Copilot understand this project
automatically — you don't need to paste in any context yourself.

- **Claude Code:** open the repo folder and start a session. It reads `CLAUDE.md` automatically.
- **GitHub Copilot (VS Code):** open the repo folder, open Copilot Chat. It reads
  `.github/copilot-instructions.md` automatically (check VS Code's setting
  `github.copilot.chat.codeGeneration.useInstructionFiles` is on — it is by default in recent
  versions). You can confirm it actually used the file by expanding the references list under a
  Copilot Chat response.

Just describe what you want in plain language, for example:

> "Add a new endpoint to export an invoice as a plain CSV, alongside the existing Excel/PDF
> export."

For anything beyond a trivial change, ask it to **outline a plan before writing code** — review
that plan, correct anything that doesn't fit how this codebase actually works, then say go. The
AI has been told this is a live, deployed system to extend within existing patterns, not a
greenfield project to design freely.

## Step 6 — Test before you push

- API: `dotnet test` (unit tests) and manually exercise your change via Swagger UI.
- UI: `npm run test` (Vitest) and `npm run type-check`.

All three run in CI on every push too, but catching a problem locally is faster.

## Step 7 — Push, and open a PR into `test`

```bash
git push -u origin feature/short-description-of-what-youre-doing
```

Open a PR targeting `test`, not `main`. CI deploys it there automatically, in an environment
fully isolated from production (its own database, its own backend) — validate your change on the
real test URL before going further.

## Step 8 — Promote to production

Once you (and whoever reviews it) are satisfied with how it behaves on `test`, merge `test` into
`main` via a normal PR. The exact commit you validated is what reaches production — not a
rebuild. CI handles the rest.

## If something goes wrong

- A failed CI run with `Error Number: 40613` or a "degraded" health status — that's a known,
  harmless SQL cold-start flake (the database auto-pauses when idle). Just retry the run.
- Anything else unexpected — check [ARCHITECTURE.md §12, the debugging
  playbook](../ARCHITECTURE.md#12-debugging-playbook--things-that-have-actually-gone-wrong)
  first; most "weird" problems here have already happened once and are written down.

Welcome aboard.
