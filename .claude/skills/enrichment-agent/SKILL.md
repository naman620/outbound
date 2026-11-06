---
name: enrichment-agent
description: Research and contact-enrichment half of the outbound-agent pipeline, run in a normal interactive session on a company list the user uploads
---

# Enrichment Agent — Operating Spec

This is the **research + enrichment** half of `outbound-agent` (see that skill for the canonical end-to-end process). The user uploads a company list as a CSV; this agent researches each company, finds and verifies contacts in Clay, and hands back a downloadable CSV ready to load into an outbound tool. Pushing leads to Smartlead or any other sending tool is never part of this agent's job — the user does that themselves after reviewing the file.

## 1. Scope: what this agent does and does not do

Does:
- Company-level research (web search/fetch) for the enrichment variables listed in §7.
- Contact discovery + email verification via Clay (`search-contacts`, `run_subroutine` Work Email).
- Assembly of the full per-lead merge-variable set, with every field set to either a real sourced value or the exact documented fallback — never a missing key.
- Delivering the results as downloadable CSVs (§8).

Never does:
- Call any Smartlead tool, or push a single lead anywhere.
- Run Clay's "Summarize Work History" or any Clay enrichment other than Work Email — the user declined it.
- Guess or invent a value for any field. Every hard no-fabrication rule in `outbound-agent` §2 applies unchanged here — a candidate without a verified email is dropped, a company signal without a real source is left as its documented fallback, nothing is fabricated to look fuller than it is.

## 2. Starting a run

1. Save the uploaded CSV in its own folder in the scratchpad and read it with `python3 -I`. It needs a `domain` column; other columns (`merchant_name`, `installed_apps_names`, `estimated_monthly_visits`, `platform`, `company_location`) are supplied data to use per §5.
2. Remove duplicate domains, and compare against every list already enriched in this session (strip a leading `www.` before comparing). If the upload overlaps an earlier list, tell the user before spending anything — re-enriching the same companies wastes Clay credits.
3. Check whether `installed_apps_names` is filled. If it is empty, say so: `incumbent_vendor` will then rely on what the research can find on each brand's own site.
4. Split the list into **batches of 100 companies** unless the user asks for another size. Run one batch at a time and pause after each one so the user can check their Claude and Clay usage before the next starts (§4).

## 3. How the work is divided

Each batch is processed by background subagents, with this session coordinating and checking their results.

- **Research:** groups of 20 companies per subagent. Each writes one JSON file and keeps existing entries, so a re-run only fills what is missing.
- **Contacts:** chunks of 50 companies per Clay subagent. Each writes one JSON file in the §7 contacts shape, keeps existing domains, and skips them on a re-run.
- Write the research brief, the Clay brief and the short per-agent prompts to files in the scratchpad, and give each subagent only its file paths and group number.
- Never mark a row as researched unless it actually was. If a subagent fills unresearched rows from input data alone, delete those rows and re-run them.

## 4. Limits — stop cleanly, never push past them

- **Web search cap:** all subagents launched in one turn share a limit of 200 WebSearch calls. Allow at most 2 searches per company and launch at most about 5 research groups (100 companies) per turn; launch the next groups on a later turn.
- **Claude usage:** no tool reports the user's usage percentage, so the agent cannot watch for a threshold itself. Pause after every batch and let the user decide whether to continue. If subagents stop with a usage-limit error, keep what was saved, tell the user when the limit resets, and resume the unfinished groups after that (a `send_later` check-in can do the restart).
- **Clay credits:** Clay's connection does not show remaining credits. When Work Email returns `ERROR_OUT_OF_CREDITS` or "Credits are exhausted", stop all Clay work, mark each affected domain's notes with `CREDITS_BLOCKED` (listing any candidates already found and their task/entity IDs), and ask the user to top up in Clay under Settings → Usage. When credits are back, re-run only the blocked domains — remove their keys first, or put them in their own chunk, since a re-run skips domains already in the file.
- **Clay concurrency:** the workspace has a low concurrency ceiling. Each Clay subagent makes its calls one at a time; two or three Clay subagents can run side by side. On "Too many concurrent requests", wait a few seconds and retry the same call.

## 5. Company research (per `outbound-agent` §5 — unchanged)

Company-level research does **not** use Clay. For each company, in order of preference:
1. Use any tech-stack/traffic/installed-apps data already supplied in the input list — treat it as sufficient, no need to re-verify. When `installed_apps_names` is empty, check the brand's own homepage code for a chat or helpdesk script.
2. Otherwise do genuine web research (`WebSearch`/`WebFetch`) — the company's own site, job postings, press, etc.
3. A case-study quote, growth stat, or named competitor benchmark must be sourced and citable. If none exists, the corresponding field gets its documented empty/sentinel fallback — never an invented one.
4. Only put a fact in a field if a page that was actually opened states it. Facts seen only in search-result snippets go in `notes`, not in fields.
5. `exec_hire_name` is only a VP/Director/C-level hire in ecommerce, marketing, CX or digital who joined within the last 12 months. General CEO/President/COO appointments, sales roles and older hires go in `notes` only.
6. If `installed_apps_names` lists "Manifest AI Chatbot", start the company's notes with `MANIFEST INSTALLED:` — it is likely an existing customer.
7. If the domain is not a real storefront (a test or staging site, a parked domain, a non-retail programme) set `company_match_ok` to false; that company gets no contacts.

## 6. Contact + email discovery (per `outbound-agent` §4, with a narrower title filter)

Only three functions buy an AI shopping/support assistant, so contacts are limited to them: **digital transformation, customer experience (CX) and eCommerce**. Marketing, sales and other roles are out of scope, even when senior.

Run the standard Clay filter per batch of up to 10 domains per `search-contacts` call:
```
select from people where
  experiences.any(
    is_current = true
    and seniority in ("Founder", "Owner", "VP", "Head", "Director", "C-suite")
    and job_title is_similar_to ("digital transformation", "customer experience", "CX", "ecommerce", "e-commerce", "digital commerce")
  )
limit 3 by clay_company_id
```
`is_similar_to` matches loosely and can return adjacent roles, so screen every candidate's actual title before enriching: keep a person only if the title names digital transformation, customer experience / CX, or eCommerce / digital commerce (e.g. "VP Ecommerce", "Head of Customer Experience", "Director of Digital Transformation", "Chief Digital Officer"). Drop titles that are only marketing, sales, customer service operations or IT, and say who was dropped and why in the company's notes.

Broadened-seniority fallback (add "Manager") only after a zero-result first pass; no bare company-name fallback (confirmed broken). Cap at 3 enriched candidates per company. Verify every candidate's email via the Work Email subroutine (`t_0tl41guQrDBpYmSWuVN`) with `fieldMapping: {"name": "Full Name", "url": "Social Profile URL", "latest_experience_company": "Company Name", "domain": "Company Domain"}`. A contact's Clay `domain` is sometimes a previous employer's; for those, run `run_subroutine_direct` with the target domain instead. If a search result has `hasMore: true`, load the next page; its emails run on the new taskId it returns.

**Regional and sub-domain storefronts:** Clay cannot look up storefront subdomains (`us.brand.com`, `shop.brand.com`). Search the parent domain instead, add a `location_country` filter for the store's market, and save the result under the domain exactly as it appears in the input.

**Location:** for a US brand, prefer contacts based in the US or Canada. Keep someone based abroad only when the company has no North America contact, and say so in notes.

**Verdict per contact:**
- Work Email empty / `{}` / null → drop.
- Email on the target domain, or on a clearly identified parent/sister brand domain (explain in notes) → `confirmed current`.
- Email on an unrelated or variant domain, or whose local part clearly belongs to someone else (e.g. `anna@` for "Hannah", `andrew@` for "Nicole Andrew") → `domain_mismatch_flagged`, explained in notes.
- University emails, personal webmail, or two contacts returning the same email → keep at most the one whose name matches; drop the rest.

**Company-match verification (added after a confirmed bad match in testing):** `search-contacts` can resolve a supplied domain to the wrong entity — observed live: `hoover.com` (the appliance brand) matched to "Hoover Sales and Service," an unrelated 2–10 person Tennessee shop, and returned contacts for that wrong company. Before accepting any result for a domain, check that Clay's returned company record (its own name/domain/size) is actually consistent with what was supplied for that domain — a company many times smaller than expected, a materially different name, or a domain that doesn't match is a sign of a bad match, not a real company. **If the match looks wrong, mark that company `wrong_company` rather than accepting its contacts or guessing which part is right.** This is a drop decision, same as any other no-fabrication drop — do not try to salvage a wrong-company match.

## 7. Per-lead output schema and merge-variable fallback table

Every company must carry the full merge-variable set the target sequence needs, each field set to a real value or its documented fallback — **never an omitted key**, since whatever tool later sends the emails renders a missing key as literal unresolved `{{tag}}` text.

```json
{
  "company_name": "...",
  "domain": "...",
  "contacts": [
    {"name": "...", "title": "...", "email": "...", "linkedin_url": "...", "confidence_flag": "confirmed current | domain_mismatch_flagged"}
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

## 8. Output files

After each batch, build two CSVs with a real CSV writer (Python's `csv` module, UTF-8, header row, fields quoted where needed) and send them with `SendUserFile`:

- **Contacts file** — one row per verified contact, ready to load into Smartlead: `email, first_name, last_name, company_name, domain, contact_name, title, linkedin_url, confidence_flag`, then every §7 merge variable in the order listed there, then `notes`. Split `first_name` / `last_name` from the contact name. No duplicate emails, within the file or against earlier files.
- **All-companies file** — every company in the batch: one row per contact, plus one row with empty contact columns and `confidence_flag = not_found` for each company without a verified contact (including `wrong_company` and `company_match_ok: false`).

Keep both files in the scratchpad, never in the repo — they contain personal emails. If a user pastes CSV text into Google Sheets and it lands in one cell, offer the same data as an `.xlsx` or tab-separated file instead.

When reporting a batch, give: companies processed, companies with a verified contact, total contacts, how many are flagged and why, wrong-company domains, companies still blocked (credits or usage), and anything to check before outreach (likely existing Manifest customers, regulated categories, weak-fit storefronts). The user reviews the files and does any pushing themselves; this agent never pushes.
