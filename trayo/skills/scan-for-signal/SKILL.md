---
name: scan-for-signal
description: Scan a list of accounts for ONE signal — a defined business event such as a new security leader, an office opening, funding — and return only the accounts that fired it. Use when the user asks "which of these companies…", "who hired a new CISO", "scan these accounts for…".
---

# Scan accounts for one signal

Collection results may use `delivery: "file"`. In that case, `preview` and `metadata` are compact and may be shortened; download `file.downloadUrl` and process the complete JSON in code before selecting or importing rows. Keep the original `hasMore`/`nextCursor` pagination, and do not repeat the search to get its file. Use `output: "file"` when a download is wanted.

1. `trayo_whoami` once.
2. Make sure the accounts are in the workspace: `trayo_import_accounts` for any that are not (`{ name, url }`, the `url` with its scheme, up to 200 per call); collect the `accountIds` from `created[].id`, plus `existingId` from every `skipped` row with `reasonCode: "duplicate"` — those companies were already in the workspace and must still be scanned.
3. Define the signal with `trayo_create_signal`: a snake_case `signalKey`, `type: news`, `detects` in plain prose. Go straight to the next step with that key: the run prepares a signal it names before it searches, so a key created a second ago is as good as one created a week ago.
4. Run `trayo_run_discovery` `{ accountIds, signalKeys: [signalKey], lookbackDays, waitSeconds: 45 }`; while `settled` is false, `trayo_get_discovery` `{ runId, waitSeconds: 45 }`. A run is final only when `settledAt` is set. **A run takes at most 500 accounts** — a larger scan is several runs over slices of the set with the same `signalKeys` and `lookbackDays`, each settled before the next starts. `lookbackDays` also sets how hard the search digs, so a long window over many accounts is not a fast call.
5. Read every event with `trayo_list_events` `{ discoveryRunId }`, following `nextCursor`. The accounts that fired are the distinct `accountId`s on the events whose `signalKeys` contains your key; each event names its account in `accountName`, so you never map an id back yourself. Reading `signalKeys` rather than trusting the run is what lets this scale: a run over several signals is attributable event by event, so scanning for more than one signal is one run, not one run each.
6. Output: the fired accounts with the event `title` and `eventDate`, and the accounts that did not fire. Offer `trayo_add_to_list` `{ name, members: [{ accountId, via: 'signal', eventId }] }` for the fired ones — and if the user will want to scan again later, that list is how the set survives, read back with `trayo_list_lists` then `trayo_get_list_members`. Over 200 members takes several calls: send `name` on the first one only, then the `list.id` it answers with as `listId` on every later one, or two by-name calls race and split the set across two lists with the same name.

Follow `retry` on every error. `eventsNew: 0` is a real result, not a failure — and on a workspace that has run before it usually means the events were already there rather than that nothing fired, so read the events instead of the counter. `news` runs on any account from the moment it is imported; `jobs` and `job_change` read hiring data and go silent on accounts that are not ready yet, which the run names in `blockedSignals` — `account_not_ready` means run it again shortly for those ids, `company_not_found` means re-running will not help.

## Scale an approved event preview

For “now give me 1,000,” call `trayo_list_events` with the approved account, signal, date, state and settled `discoveryRunId` filters, `limit: 1000`, `output: "file"`, `expand: "people,signals"`, and no cursor. The total counts events including preview matches, with existing stakeholders nested. Download `file.downloadUrl` in code; check actual `rowCount`, `metadata.collection.stopReason`, and `hasMore`. Continue with `nextCursor` and the remaining count if needed. This reads existing events and attached people; it does not run discovery or find missing stakeholders.
