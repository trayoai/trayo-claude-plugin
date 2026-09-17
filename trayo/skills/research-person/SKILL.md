---
name: research-person
description: Research one person — their role and trajectory, what they are likely up against, and how it ties to what you sell. Use when the user names someone or pastes a professional profile and asks "who is this", "prep me for this call", "what should I know about them", or wants a brief before writing to them.
---

# Research a person

Collection results may use `delivery: "file"`. In that case, `preview` and `metadata` are compact and may be shortened; download `file.downloadUrl` and process the complete JSON in code before selecting or importing rows. Keep the original `hasMore`/`nextCursor` pagination, and do not repeat the search to get its file. Use `output: "file"` when a download is wanted.

1. Name them. `trayo_research_person { person: { linkedinUrl } }` takes a professional profile URL or just its handle, and they do NOT have to be in the workspace. `{ person: { personId } }` names someone who already is. Exactly one of the two, never both — sending both is `validation_failed`. A company page is not a person profile and is refused.
2. If the user has no URL, find the person first and copy the `linkedinUrl` from the row: `trayo_search_stakeholders { company: { website }, definition }` for the right people at one company (each row carries `whyThisPerson` and, when Trayo holds one, `linkedinUrl`), or `trayo_find_people` for a title across many companies. Check `person.fullName` and `person.resolvedBy` on the answer before you trust the brief — it tells you whether Trayo looked up the supplied profile or used the workspace person's existing match.
3. `grounded: true` adds a web search. It fills `personalInsights` and `companyIntel`, and it is the only way `location` and `education` are ever filled; without it those are empty and null by design, not missing data. It costs 5–15 s instead of 2–3 s and occasionally fails outright (`research_failed`, `retry_later` — retry once, then go without it). Neither mode is metered.
4. Read the answer. `research` holds the structured fields — `careerProgression` (oldest first), `professionalInsights`, `challenges` — and `reportMarkdown` is the same content rendered for display, so use one or the other, not both. A grounded fact is only stated when Trayo finds a supporting source, but the sources themselves are not published: treat every grounded claim as a lead to check, and never present it to the user as a citation.
5. Nothing is written. This tool does not create, enrich or update a person. To keep them, `trayo_add_people` with `{ fullName, title, linkedinUrl }` and then `trayo_add_to_list`; for their email address, skill `enrich-contacts`. For the company behind them, `trayo_research_company` (skill `research-account`).

Output: the brief in chat, or written to a file the user names — their role and trajectory, what they are likely up against, and the two or three things worth opening with. Every line from the answer, none invented.
