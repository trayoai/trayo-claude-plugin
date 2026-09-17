---
name: discover-signals
description: Track a list of companies in Trayo and read what happens at them — import accounts, define a signal, run a discovery, wait for it, read the events. Use when the user has company names or websites and wants news, hiring or job-change events for them, or asks to "run discovery", "find signals", "monitor these accounts" or "what happened at these companies".
---

# Track companies and read what happens at them

Collection results may use `delivery: "file"`. In that case, `preview` and `metadata` are compact and may be shortened; download `file.downloadUrl` and process the complete JSON in code before selecting or importing rows. Keep the original `hasMore`/`nextCursor` pagination, and do not repeat the search to get its file. Use `output: "file"` when a download is wanted.

The core Trayo loop, with the `trayo_*` tools.

## Before anything

1. Call `trayo_whoami` once. It tells you the workspace you act in and whether metered work may start (`canInitiate`). Plan against what it says.

## The loop

1. **Import the companies** with `trayo_import_accounts`: up to 200 rows of `{ name, url }` (the url needs its scheme, `https://…`), `linkedinHandle` optional. Read `created[].id` — those are the `accountIds` you need. Rows in `skipped` were duplicates or carried fields the route does not accept; either is fine, not a failure, and the rest of the batch still lands. A skip with `reasonCode: "duplicate"` carries `existingId`, the account that was already there — use it as an `accountId` beside the created ones instead of dropping the company. A row missing `name`, or with a scheme-less `url`, is not skipped: it fails the whole call and creates nothing, so fix that row and re-send. Re-importing a list is safe.
2. **Define what counts** with `trayo_create_signal`: a lowercase `signalKey` (3–60 chars, snake_case, new), `type` `news` (works on any account the moment it is imported; `jobs` and `job_change` read hiring data, so an account only becomes searchable by them shortly after import — a run names any that are not ready in `run.blockedSignals`), and `detects`: plain prose saying what to look for. Use the key in the next step as soon as this returns: the run prepares a signal it names before it searches it.
3. **Run the discovery** with `trayo_run_discovery` `{ accountIds, signalKeys, lookbackDays: 30, waitSeconds: 45 }`. It waits up to 45 s.
   - `settled: true` → done. `run.eventsFound` and `run.eventsNew` are this execution's receipt; `events` holds the first page standing for the run's account × signal × lookback scope, including matches an earlier run wrote. If it is `null`, read the same scope with `trayo_list_events` as `next` says.
   - `settled: false` → call `trayo_get_discovery` `{ runId: run.id, waitSeconds: 45 }` and repeat until `settled`. Never poll faster than the wait the tool offers, and never conclude anything from counts before `settledAt` is set.
4. **Read the events** with `trayo_list_events` `{ discoveryRunId }` for the rest of that run's scope, or without the filter for the whole workspace. Follow `nextCursor` while `hasMore` is true.

## Rules the API holds you to

- Every tool error carries `code` and `retry`. `fix_input`: change the arguments. `retry_later`: wait, then repeat the same call. `do_not_retry`: stop; the same call will keep failing.
- `eventsNew: 0` on a second run over the same accounts is success: results deduplicate against the workspace, while the scoped `events` page still returns matching events already there.
- `run.error` on a completed run means it did not do everything it was asked; say so to the user instead of hiding it.
- Ask before running discovery on more than 100 accounts, and never re-run the same accounts in a loop hoping for more.

## What to hand back

The companies imported (and skipped), the signal used, the run id, `eventsNew`, and the event titles with the account names you imported — events carry `accountId`, not names, so map each one back to the row you sent. Offer `trayo_list_events` for more.

## Scale an approved event preview

For “now give me 1,000,” call `trayo_list_events` with the approved account, signal, date, state and settled `discoveryRunId` filters, `limit: 1000`, `output: "file"`, `expand: "people,signals"`, and no cursor. The total counts events including preview matches, with existing stakeholders nested. Download `file.downloadUrl` in code; check actual `rowCount`, `metadata.collection.stopReason`, and `hasMore`. Continue with `nextCursor` and the remaining count if needed. This reads existing events and attached people; it does not run discovery or find missing stakeholders.
