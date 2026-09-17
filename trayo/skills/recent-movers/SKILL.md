---
name: recent-movers
description: Find people who recently changed jobs — who left a company, who joined one, or who moved between two. Use when the user asks about "recent hires at", "who left", "job changes", "past champions who moved".
---

# Recent movers

Collection results may use `delivery: "file"`. In that case, `preview` and `metadata` are compact and may be shortened; download `file.downloadUrl` and process the complete JSON in code before selecting or importing rows. Keep the original `hasMore`/`nextCursor` pagination, and do not repeat the search to get its file. Use `output: "file"` when a download is wanted.

1. `trayo_search_job_changes` with `destination: { website }` for who joined, `source: { website }` for who left, or both for moves from one to the other. At least one of the two is required. Each row: the `person`, both roles under `source` and `destination`, `detectedAt` and `startedAt`. At most 25 rows come back, most recently detected first, one row per person per destination.
2. This search is the reliable read. Watching job changes over time needs a `job_change` signal (`trayo_create_signal`, then discovery — skill `scan-for-signal`), and that type goes silent on any account Trayo holds no hiring data for. An account you import becomes searchable by it on its own, usually within minutes; until then the run reports the signal in `blockedSignals` with `reason: "account_not_ready"` instead of returning nothing meaningful, and `company_not_found` there means re-running will not help.
3. Output: the movers with old and new role and the date; offer a CSV, or keep them with `trayo_add_people` — up to 200 rows of `{ fullName, title, linkedinUrl }` taken from each `person`, plus `company: { name: destination.name, domain: destination.website }` to file them under the company they joined. `destination.accountId` is only populated when step 1 named that company by `accountId` in the first place — name it by `website` and it comes back null even when the company is an account of yours, so do not read it as a lookup. Then `trayo_add_to_list` with the `created[].id`s, taking `existingId` from every `skipped` row marked `reasonCode: "duplicate"`.

`detectedAt` is when Trayo first saw the new role, not the start date. `startedAt` is the start date, to the month (`2026-03`, or `2026` when only the year is known, or null when we hold none) — use it for "who joined in the last N weeks", and expect nulls.

`hasMore: true` means the 25-row cap was reached and more matches exist. The tool does not paginate, so there is no next page to fetch: narrow the search (send both sides, or a smaller company) and say so rather than implying the 25 rows are everything.
