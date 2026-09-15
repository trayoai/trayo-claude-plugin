---
name: research-account
description: Research one account — what the company does, why now, who the stakeholders are, and what has happened at it. Use when the user names a company and asks for a brief, a profile, "tell me about", "who should I talk to at", or pre-meeting prep.
---

# Research an account

1. `trayo_research_company` with `{ company: { website } }`, or `{ company: { accountId } }` when it is in the workspace — `accountId` goes on its own, never beside `website`, `linkedinUrl` or `name`. It answers the overview in `research`, the same content as `reportMarkdown`, and the company as resolved.
2. `trayo_search_stakeholders` with the same `company` and, if the user described the roles they sell to, that prose as `definition`: 5 people with `whyThisPerson` by default, so pass `limit: 10` when the user wants up to ten. Read `candidates` next to `people` — a positive `candidates` with an empty `people` means none of them fit the definition, which is an answer, not a failure.
3. If the company is an account in the workspace, `trayo_list_events` with `{ accountId }` for what has happened at it; otherwise say events need the account imported and a discovery run (skill `scan-for-signal`).
4. Hand back one brief: what they do, why now, the stakeholders with their reasons, recent events with their `eventDate` and the `signalKeys` each one matched — every claim from a tool result, none invented.

Check `company.name` and `company.website` on both answers before trusting either: a common name or a guessed handle resolves to a different company, and the people would be theirs. `company_not_found` is `do_not_retry` — the same reference keeps missing, so send the website or company profile URL instead.

Output: the brief in chat, or written to a file the user names. To keep the people, `trayo_add_people` with the step-2 rows as they came back — `headline`, `location` and `whyThisPerson` have no column on a person and are dropped, so copy the reason into `note` yourself if it is worth keeping — and add `accountId` (from `company.accountId`, when the company is an account of yours) or `company: { name, domain }` (`company.website` is the domain) to each row, or they land unlinked. Then `trayo_add_to_list` with the `created[].id`s.
