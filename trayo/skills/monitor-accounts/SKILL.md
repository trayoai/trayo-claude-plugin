---
name: monitor-accounts
description: Watch a set of accounts — given as a list of companies, or as a definition of the kind of company — for a set of signals, backfill their recent history, and report what is new on every later check. Use when the user asks to "monitor", "keep an eye on", "alert me when", "watch these accounts for…", or "what's new at our accounts since…".
---

# Monitor accounts for signals

Two shapes of the same job. The user either **names the accounts** ("watch my book"), or **describes them** ("AI companies in the US"). Only step 2 differs.

Trayo has no push: what you build is a backfill plus a repeating read. Say so plainly, and never describe it as an alert that arrives on its own.

## Set up (once)

1. **`trayo_whoami`.** Read `monitoring`. `enabled: false` means this workspace is not scanned on a schedule at all, so the ONLY events that will ever appear are from runs you start — say so plainly before you promise ongoing coverage, and plan to run discovery yourself. It says whether, not how often: pick whatever checking rhythm suits the user, and never tell them a schedule Trayo has not promised.

2. **`trayo_get_workspace`. If `stakeholderCriteria` is null, set it now with `trayo_set_workspace` before anything else.** The people attached to each event — the ones carrying a `reasoning` for why that person matters to that event — are only attached when a definition exists when the run happens, and it is never applied backwards. Set it late and every event you already collected keeps zero people. Write it as prose: the roles that own or buy what the user sells, the roles that influence the decision, the roles to skip. If the user has not said what they sell, ask; it is one question and it steers everything downstream.

3. **The accounts.**
   - *Named accounts:* `trayo_import_accounts` with `{ name, url }` rows, the `url` with its scheme, up to 200 per call. Collect `created[].id`, and `existingId` from every `skipped` row with `reasonCode: "duplicate"` — those companies are already in the workspace and must still be watched.
   - *A definition:* `trayo_find_companies` with `filters` for anything exact (industry, headcount, country, funding) and follow `nextCursor` until you have the set the user asked for; a `query` sentence returns one ranked page and takes no cursor. Then import those rows — pass each row's `url`, not its bare `website`. Read `droppedUnaddressable` in the answer before you report a count.
   - Either way, **put the set on a list**: `trayo_add_to_list { name, members: [{ accountId, via }] }`. The list is how the set survives to the next check — a later session reads it back with `trayo_list_lists` then `trayo_get_list_members` instead of rebuilding it, and a rebuilt set silently drifts from the one that was backfilled.

4. **The signals, one per business event to watch.** `trayo_create_signal` with a snake_case `signalKey`, a `type` (`news`, `jobs`, `job_change`) and `detects` in plain prose. Several signals are one run, not one run each: every event names the signals it matched. Keep every key — the repeating check filters on them.

5. **Backfill.** `trayo_run_discovery { accountIds, signalKeys, lookbackDays: 90, waitSeconds: 45 }`, then `trayo_get_discovery { runId, waitSeconds: 45 }` while `settled` is false. A run is final only when `settledAt` is set.
   - **A run takes at most 500 accounts.** More than that is several runs over slices of the set, each with the same `signalKeys` and `lookbackDays`; wait for each to settle before starting the next so the work does not pile up.
   - `lookbackDays` sets how far back news counts **and** how hard the search digs, so 90 days over hundreds of accounts is not a fast call. Tell the user it is running.
   - Read `blockedSignals` before you believe a small number. `account_not_ready` means hiring data for those companies has not resolved yet, so `jobs` and `job_change` skipped them; `news` is unaffected. `company_not_found` means re-running will not help.
   - `eventsNew: 0` on a workspace that has run before is usually not silence: an event already in the workspace is not written twice. Read the events rather than trusting the counter.

6. **Report the backfill** with `trayo_list_events { signalKeys, expand: "all" }`, following `nextCursor`. Per event: `accountName` (the account is named on every event — you never need to map an id back yourself), `title`, `eventDate`, the matched `signalKeys`, and `whyItMatters` — the paragraph discovery wrote relating this event to this workspace. Quote it; never invent one. It is null on older events and on some signal types, and then `summary` is what you have. Name the signal from `signalKeys`, not `signalType`: the type is only the kind of source. `expand` adds each match's `confidence` and `evidence`, and the `people` array with a per-person `reasoning`.

7. **Stakeholders.** If the user wants people per account rather than per event, `trayo_search_stakeholders` on the accounts that fired, and `trayo_get_contacts` for contact details on the people that matter. Contact lookups draw on the workspace allowance `trayo_whoami` reports.

## Every later check

8. **`trayo_list_events { discoveredSince: <the time you last looked>, signalKeys: [...] }`**, following `nextCursor`.
   - `discoveredSince` is when Trayo **found** the event; `since` is when the news **happened**. Use `discoveredSince` here, always. An article published three weeks ago that reached the workspace this morning is new to the user and `since` would drop it.
   - Pass a full ISO-8601 timestamp you compute yourself — `yesterday` and `7d` are rejected — and store the timestamp you used, not the events you saw.
   - `signalKeys` takes several keys at once and matches any of them, so one read covers every signal being watched.
   - `state` defaults to live, so events Trayo has withdrawn are already excluded.

9. **Output.** A digest: account, what fired, when, why it matters, and who to talk to. For a recurring job, offer to write the user a script they run on their own schedule — `GET /v1/events?discoveredSince=…&signalKeys=…` with their key, posting to team chat or email. The clock is theirs; nothing in Trayo holds it.

An empty check is a real answer — but before reporting a quiet week, re-read `monitoring` from `trayo_whoami`. A workspace the standing scan stopped covering reads exactly like a workspace where nothing happened.
