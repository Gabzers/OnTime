# Local AI Setup (Ollama + Continue)

Hub: [[OnTimeCRM]] · Maintenance & context rules: [[HOW-TO-USE]].

## Pre-flight checklist (do BEFORE installing Ollama)

Goal: be sure the project is healthy so the model has a known-good baseline. If any step fails,
fix it first — never let the model touch a broken setup.

**Environment:**
- [ ] Windows 11 + RTX 5070 with current NVIDIA driver (CUDA-capable).
- [ ] Docker Desktop installed and running.
- [ ] .NET 8 SDK (`dotnet --version` → 8.x).
- [ ] Node 20+ (`node -v`).
- [ ] VS Code (or Rider).
- [ ] Repo cloned; open the **monorepo root as the workspace** so code + `ONTIMECRM/` vault are both visible.

**Baseline (must all pass before installing):**
```bash
docker-compose up -d
cd OnTimeCRM_Backend && dotnet build && dotnet test
cd ../OnTimeCRM_Frontend && npm install && npm run build
```
- [ ] `dotnet test` is green (the known bugs in [[KNOWN-ISSUES]] don't currently fail tests — they're un-tested behavior; this changes as T1+ adds tests).
- [ ] `npm run build` is clean.
- [ ] `http://localhost:5173` loads.

**Read once (so the workflow is clear):**
- [ ] [[HOW-TO-USE]] · [[AI-WORKFLOW]] · [[AI-TASK-PLAN]] · [[CONVENTIONS]]

**Optional but recommended:**
- [ ] Create the `crm_ro` Postgres role (SQL in "Access policy" below) — lets the model read DB, never write.
- [ ] Decide Continue auto-approve list (the safe verbs in "Access policy").

Once everything above is ticked → proceed.

---

## The key idea
Ollama is just a model server. **The model does not inherently know your code or this vault.**
It only "knows" what the editor extension feeds it as context each request. So the setup is:
run the model in Ollama, open the whole `OnTimeCRM/` folder (code **and** this vault) in the
editor, and use **Continue** to pass the right files as context.

## Install
```bash
ollama pull qwen2.5-coder:14b     # coding model (~9GB VRAM, fits the 5070's 12GB)
ollama pull nomic-embed-text      # embeddings for codebase indexing (RAG)
```
Then install the **Continue** extension in VS Code (or Rider).

## Continue config (`~/.continue/config.json`)
```json
{
  "models": [
    { "title": "Qwen Coder", "provider": "ollama", "model": "qwen2.5-coder:14b" }
  ],
  "embeddingsProvider": { "provider": "ollama", "model": "nomic-embed-text" }
}
```

## How it "knows everything"
Open the repo root (`OnTimeCRM/`) as the workspace so both the .NET/React code AND `ONTIMECRM/` (this vault) are on disk for the extension. Then, per request, give context:

| Method | What it does | When |
|--------|--------------|------|
| `@file CONVENTIONS.md` | attaches one exact file | **preferred** — surgical, precise |
| `@codebase <question>` | embeds + retrieves relevant chunks (RAG via nomic-embed-text) | when you don't know which files matter |
| `@files` | attach several files at once | a feature touching a few files |

The vault is plain markdown, so Continue indexes/reads it like any file. **By default it's not Obsidian-aware** — it doesn't follow `[[wikilinks]]` or write notes; you point it at files and you (or Claude) keep them current.

> **Optional upgrade — Obsidian MCP.** Install the "Local REST API" Obsidian community plugin + the `mcp-obsidian` MCP server, wire it to Continue. The model then has real tools: `list_files_in_vault`, `search`, `read_file`, `patch_content`. Useful for vault-wide search and letting the model update [[STATUS]]/[[KNOWN-ISSUES]] itself. **Caveat:** `patch_content` lets the model autonomously edit notes — real risk of doc drift with a 14B model. Skip for the TDD coding workflow; revisit if `@file` feels clunky.

## Recommended workflow (best results on a 14B model)
Surgical context beats `@codebase`. For most tasks:
```
@file ONTIMECRM/00-PROJECT/CONVENTIONS.md
@file ONTIMECRM/00-PROJECT/DOMAIN.md          (or the relevant SCHEMA-*/feature note)
<concrete task with the exact endpoint/DTO/rule>
```
See [[HOW-TO-USE]] for the per-task context recipes.

## What to build, and how
- **What/order:** [[AI-TASK-PLAN]] — step-by-step tasks, each with the failing test to write first.
- **How every task runs:** [[AI-WORKFLOW]] — RED → GREEN → verify everything → update docs.

## Giving it more access (optional, recommended)
By default Continue only reads/writes files. You can extend it:

- **Agent mode** — lets the model run shell commands (e.g. `dotnet test`, `npm run test`). Continue
  asks for approval per command by default; auto-approve common safe ones in config. This is how
  the model can execute the [[AI-WORKFLOW]] gate itself instead of you copy-pasting output.
- **MCP servers** — Continue supports MCP. Hook it to: Postgres (run queries), a filesystem server
  (broader file access), GitHub, etc. Each one adds a specific capability.
- **Browser** — not built-in to Continue. Easiest path: keep the browser check on Claude Code's
  side (it has browser MCPs), or use Playwright MCP + the local model.

## Access policy — **read-only on anything critical**

The 14B model can confidently run a wrong command. The rule for this project:

| Resource | Local AI gets |
|----------|---------------|
| Repo files (code + vault) | ✅ read + write — main job |
| `dotnet build/test`, `npm run test/build`, `git status/diff/log` | ✅ auto-approve (safe verbs) |
| **Database (local Postgres)** | ✅ **read-only via Postgres MCP** — SELECT / `\d` only |
| `appsettings.Development.json`, `.env*` | ✅ read when needed; ❌ never write |
| `git commit/push`, schema/data writes, external API calls | ❌ require approval each time |
| Production DB / Supabase | ❌ never connect from the local model |

**How to set up the read-only DB access:**
```sql
CREATE ROLE crm_ro LOGIN PASSWORD '…';
GRANT CONNECT ON DATABASE ontimecrm TO crm_ro;
GRANT USAGE ON SCHEMA public TO crm_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO crm_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO crm_ro;
```
Point the Postgres MCP server at `crm_ro`. The model can answer "what's in this table?" and verify
fixes against real data, but **cannot mutate state** even if a prompt tells it to.

In Continue, allow-list the safe shell verbs by exact prefix; everything else asks for approval.

## Honest limits
- Finite context window — even with agent mode, curate inputs. `@file` the exact note/code.
- `@codebase` retrieval is approximate; for correctness-critical work, point at the right file directly.
- It won't plan architecture well — do that with Claude, record decisions in `04-DECISIONS/`, then hand the local model concrete, well-scoped tasks (e.g. a single [[KNOWN-ISSUES]] item).

## Related
[[HOW-TO-USE]] · [[CONVENTIONS]] · [[ROADMAP]] · [[OnTimeCRM]]
