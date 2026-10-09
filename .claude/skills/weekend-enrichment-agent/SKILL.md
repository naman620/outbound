---
name: weekend-enrichment-agent
description: Cloud-routine variant of the outbound-agent pipeline for unattended weekend runs
---

# Weekend Cloud Enrichment Agent — Operating Spec

This is the cloud-routine variant of `outbound-agent` (see that skill for the canonical interactive process). It exists for one reason: the interactive `outbound-agent` session depends on local MCP servers (Smartlead) that are only reachable from the user's own machine, so it cannot run unattended while the device is off. This spec runs the **research + enrichment** half of the same pipeline as a claude.ai scheduled cloud routine (`RemoteTrigger`), so a batch of accounts can be fully researched and verified over a weekend — with the Smartlead push always left for a live interactive session afterward.

**Do not run this from memory of a prior conversation.** Every fire of the routine is a fresh cloud session with no memory of earlier fires except what it reads back from its own Artifact. Treat every instruction below as something the prompt itself must restate each time, not something the agent "already knows."

## 1. Scope: what this agent does and does not do

Does:
- Company-level research (web search/fetch) for the enrichment variables listed in §7.
- Contact discovery + email verification via Clay (`search-contacts`, `run_subroutine` Work Email, `add-contact-data-points` Summarize Work History).
- Assembly of the full per-lead merge-variable set, with every field set to either a real sourced value or the exact documented fallback — never a missing key.
- Writing all of this to a resumable Claude Artifact so progress survives across hourly fires and is retrievable from any device Monday.

Never does:
- Call any Smartlead tool, or push a single lead anywhere. Smartlead is a local-only MCP server and is never attached as a connector on this routine — structurally, not just by instruction, it cannot reach it.
- Guess or invent a value for any field. Every hard no-fabrication rule in `outbound-agent` §2 applies unchanged here — a candidate without a verified email is dropped, a company signal without a real source is left as its documented fallback, nothing is fabricated to look fuller than it is.

## 2. Why a cloud routine needs different plumbing than the interactive agent

- **No shared filesystem.** A cloud routine session does not share local disk with the machine that created it. Local scratchpad files, CSV exports, or anything written to `/private/tmp/...` disappears with that session and is not retrievable later. All durable state must live in a **Claude Artifact**, which is hosted on claude.ai and reachable from any later session regardless of device.
- **No local MCP servers.** Only `claude.ai` connectors (the account's own connected apps, e.g. Clay) can be attached to a routine. Smartlead, being configured locally via `claude mcp add`, cannot be attached at all — this is the structural guarantee that this agent can never push anything.
- **Every fire is a new session.** A recurring routine's cron fire has no memory of the previous fire. The only way it "remembers" what's been done is by reading its own previously-published Artifact back at the start of each fire.
- **Minimum fire interval is 1 hour.** Batch size per fire should be sized so one fire's work reliably finishes well inside that hour (empirically, ~50 companies of full research + Clay enrichment takes well under an hour sequentially — see §6).

## 3. Resumable state — the Artifact is the only memory this agent has

Use the fixed Artifact title `"weekend_enrichment"` (there is no campaign name — Smartlead is not used for this run) and use it exactly, every fire.

**At the start of every fire:**
1. `Artifact({action: "list", scope: "mine", limit: 50})` — find the artifact with that exact title.
2. If found: `Artifact({action: "read", url: <that url>})`, and parse the hidden state blob embedded as `<script type="application/json" id="state">...</script>` in the page source. If not found, this is the first fire — initialize fresh state.
3. State shape:
   ```json
   {
     "done": [ /* one entry per fully-processed company, see §7 schema */ ],
     "remaining_domains": [ /* domains not yet attempted */ ],
     "quota_baseline_percent": null,
     "complete": false,
     "stopped_reason": null
   }
   ```
4. If `state.complete` is already `true`, stop immediately — there is nothing left to do this fire.

**At the end of every fire:** republish the *same* artifact (pass its existing `url` so it updates in place) with an updated state blob and a human-readable table of everything in `state.done` so far — company, domain, contacts, and a compact view of the researched variables. This page is what gets opened Monday to review before pushing.

**The contacts sheet — rebuilt and published on every fire.** Alongside the page, publish one cumulative `contacts.csv` through the publish call's `files` (e.g. `files: {"contacts.csv": "<scratchpad>/contacts.csv"}`), and put a prominent "Download contacts (CSV)" link to `contacts.csv` at the top of the page. Rules:
- Rebuild the whole file from `state.done` every fire — never append to the previous fire's file — so it always holds every company processed so far, with no duplicates.
- One row per verified contact (a contact with a verified email). A company with no verified contact still gets one row with the contact columns empty, so nothing processed is missing from the sheet.
- Columns, in this order: `company_name, domain, contact_name, title, email, linkedin_url, confidence_flag`, then every `merge_variables` key from §7 in the order listed there, then `notes`.
- Write it with a real CSV writer (Python's `csv` module): UTF-8, header row, every field quoted where needed, so commas, quotes and newlines in researched text can't break the columns.

## 4. Usage budget — no cap

There is no usage cap for this run (decided by the user, 2026-10-09). `mcp__ccd_session_mgmt__get_usage` does not exist inside a cloud routine session, so do not try to call it and do not stop or ask for confirmation over usage. The run ends only when `state.remaining_domains` is empty (§8). `state.quota_baseline_percent` stays `null`.

## 5. Company research (per `outbound-agent` §5 — unchanged)

Company-level research does **not** use Clay. For each company, in order of preference:
1. Use any tech-stack/traffic/installed-apps data already supplied in the input list — treat it as sufficient, no need to re-verify.
2. Otherwise do genuine web research (`WebSearch`/`WebFetch`) — the company's own site, job postings, press, etc.
3. A case-study quote, growth stat, or named competitor benchmark must be sourced and citable. If none exists, the corresponding field gets its documented empty/sentinel fallback — never an invented one.

## 6. Contact + email discovery (per `outbound-agent` §4 — unchanged)

Run the standard Clay filter per batch of up to 10 domains per `search-contacts` call:
```
select from people where
  experiences.any(
    is_current = true
    and seniority in ("Founder", "Owner", "VP", "Head", "Director", "C-suite")
    and job_title is_similar_to ("customer success", "customer experience", "Marketing", "Sales", "ecommerce", "e-commerce", "digital")
  )
limit 3 by clay_company_id
```
Broadened-seniority fallback (add "Manager") only after a zero-result first pass; no bare company-name fallback (confirmed broken). Cap at 3 enriched candidates per company. Verify every candidate's email via the Work Email subroutine (`t_0tl41guQrDBpYmSWuVN`) and current employment via Summarize Work History before counting them as resolved. Run Clay calls **sequentially, one at a time** inside a single routine fire — do not fan out to concurrent subagents from within the routine; the account's Clay workspace has a confirmed low concurrency ceiling, and a single sequential caller is always safe.

**Company-match verification (added after a confirmed bad match in testing):** `search-contacts` can resolve a supplied domain to the wrong entity — observed live: `hoover.com` (the appliance brand) matched to "Hoover Sales and Service," an unrelated 2–10 person Tennessee shop, and returned contacts for that wrong company. Before accepting any result for a domain, check that Clay's returned company record (its own name/domain/size) is actually consistent with what was supplied for that domain — a company many times smaller than expected, a materially different name, or a domain that doesn't match is a sign of a bad match, not a real company. **If the match looks wrong, mark that company `not_found` rather than accepting its contacts or guessing which part is right.** This is a drop decision, same as any other no-fabrication drop — do not try to salvage a wrong-company match.

Batch size: ~50 companies per fire is a safe default (comfortably under an hour at the full research+contacts pace observed historically, ~6.5K tokens/company). Adjust down if a given list's companies are proving unusually hard to research.

## 7. Per-lead output schema and merge-variable fallback table

Every entry in `state.done` must carry the full merge-variable set the target sequence needs, each field set to a real value or its documented fallback — **never an omitted key**, since whatever tool later sends the emails renders a missing key as literal unresolved `{{tag}}` text.

```json
{
  "company_name": "...",
  "domain": "...",
  "contacts": [
    {"name": "...", "title": "...", "email": "...", "linkedin_url": "...", "confidence_flag": "confirmed current | domain_mismatch_flagged | not_found"}
  ],
  "merge_variables": {
    "top_signal": "... or 'No verified signal found.'",
    "incumbent_vendor": "... or 'No incumbent vendor identified.'",
    "incumbent_metric": "... or ''",
    "industry_usecase": "...",
    "manifest_capability": "...",
    "platform": "...",
    "traffic": "...",
    "exec_hire_name": "... or 'No qualifying executive hire found.'",
    "title_hiring": "... or ''",
    "rloe_connection": "... or ''",
    "competitor_benchmark": "... or ''",
    "case_study_quote": "... or ''"
  },
  "notes": "..."
}
```

Fallback mechanism per field — reproduced from `outbound-agent` §7, this is the authoritative table and must not drift from it:

| Variable | Populated when | Fallback |
|---|---|---|
| `top_signal` | Always — strongest verified evidence | Sentinel `"No verified signal found."` |
| `incumbent_vendor` | A named tool is confirmed | Sentinel `"No incumbent vendor identified."` |
| `incumbent_metric` | A named stat is confirmed | Empty string `""` |
| `industry_usecase` / `manifest_capability` / `platform` / `traffic` | Always, from research | — |
| `exec_hire_name` | A real, sourced recent senior hire exists | Sentinel `"No qualifying executive hire found."` |
| `title_hiring` | A real, found job posting exists | Empty string `""` |
| `rloe_connection` | Independently verified | Empty string `""` |
| `competitor_benchmark` | A real, sourced, citable case study/peer brand exists | Empty string `""` |
| `case_study_quote` | A real, correctly-attributed quote exists | Empty string `""`, never reassigned to a different person |

`suggested_date`, `suggested_time`, `calendly_link`, `founder_linkedin`, `playbook_link` are static/computed-at-send-time fields — this agent does not need to populate them; they stay the sequence's own standing values.

## 8. End of a run

When `state.remaining_domains` is empty, set `state.complete = true` and publish a final summary: total companies processed, hit rate, total verified contacts, total leads with a full merge-variable set ready to push, and a clear flag on any left in a degraded state (e.g., contact found but no company signal confirmed). The final `contacts.csv` holds every company and contact from the whole run. This final artifact is the handoff document for the Monday interactive session — review it there before any lead is pushed anywhere; this agent never pushes.
