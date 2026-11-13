# Job Tracker

A small, private phone app for keeping track of job applications: company, role, date applied, status and notes. Each application has a dated stage history (Applied → Interviewing → Offer …), and a summary under the count shows how many applications are at each status. It runs offline from the home screen and keeps everything on the phone.

App page: https://cloudcrane.ai/blueprints/job-tracker-k3lta7ebhm (this page only exists while the app is public; it returns 404 while the app is private).

Built for the CloudCrane challenge, entirely through the CloudCrane MCP server.

## How it was built

- **Claude Opus** (main session) did all the building: it wrote the app spec, designed the data changes and migrations, decided what to ship or hold back, and reviewed each publish.
- **A read-only Claude Sonnet "researcher" subagent** read the CloudCrane guides, ran the test passes (`get_spec`, `try_app`, `look_at_app`, `diff_versions`) and reported back with evidence. It had no publish, update, rollback or delete tools, and Opus checked its reports against the source before relying on them.
- **The app is a CloudCrane app spec**, not hand-written HTML: screens, saved data, CEL rules, migrations, journeys, promises and tests, all described as JSON (`versions/`).

## Version map

| App version | CloudCrane version | What changed |
|---|---|---|
| v1 | 1 | Add, list and edit applications, one status each |
| v1.1 | 2 | Layout fixes, a hint when company or role is missing, quicker tests |
| v2 | 7 | Saved-data change: a dated stage history per application; numbered cards; status chips that follow the value |
| v3 | 14 | Dates fixed on iPhone; same-day stage rules; history clean-up; sorting; status summary |

Versions 3–6 and 8–13 were previews whose publish-time tests failed. None of them went live.

## Notes

**Migration 1 → 2 (v2).** `status` became a list of dated stages. Each existing application got one stage from its status and date applied. `status` is still written, always equal to the latest stage. CloudCrane keeps no copy of the old data, so the migration stores the untouched v1 data inside the saved data as `v1Backup`. A browser run with six staged entries confirmed what a migration's CEL receives (the saved object, as `saved`) and returns (it replaces the saved data whole).

**Migration 2 → 3 (v3).** It cleans up junk stages left by switching back and forth: identical consecutive stages are merged, and stages from the same day collapse into that day's last state. It never removes the first stage, and it recalculates `status`. Every item's v2 stages are kept in `v2Backup`, and `v1Backup` is carried over. Example: Interviewing (28 Oct) → Offer → Interviewing → Offer (12 Nov) becomes Interviewing (28 Oct) → Offer (12 Nov). Both backups were checked to survive the migration, later edits and new entries.

**The Safari date bug.** v2 showed no dates on iPhone ("Applied ·" with nothing after it), but every test passed. CloudCrane's runtime `formatDate` adds a `Z` to date-only values, which turns "2026-11-02" into `new Date("2026-11-02Z")`. Chromium, which the test runner uses, accepts that; Safari's JavaScriptCore returns Invalid Date. So the tests passed, and the phone failed. v3 formats dates itself in CEL from a full timestamp, with examples that `check_publish` evaluates. It was verified by hand in iPhone Safari, because the runner can't cover Safari.

**Promoting v3 without a passing publish run.** From mid-afternoon on 12 Nov, CloudCrane's publish-time test runner failed the add-and-save flow under its frozen test clock. An A/B run showed it also failing the unchanged, previously passing v2 app (passed at 14:42, failed at 17:46), while `try_app` without the clock passed the same flows. Since the failure wasn't in v3's code, v3 had been verified on an iPhone, v2 had a visible bug on every iPhone, and both data backups were in place, v3 (version 14) was promoted deliberately without a passing run. The decision and its evidence are recorded in `BUILD_LOG.md`.

## What's in this repo

- `BUILD_LOG.md`: the log for every version: the summary, the saved-data change and migration, test results, what broke and how it was fixed, and CloudCrane friction.
- `CLAUDE.md`: the working rules Claude followed (model roles, data model, testing and release rules) and the running list of CloudCrane feedback.
- `spec.json`: the current app spec (same as `versions/v3.spec.json`).
- `versions/`: the app spec of each shipped version (`v1`, `v1.1`, `v2`, `v3`).
- `drafts/`: specs used for checks and experiments, never shipped (the v2 migration check, the diagnostic run, the lean test set, and the A/B control).
- `transcript/`: the two AI chat sessions in readable form (`session-1-setup.md`, `session-2-build.md`), plus `SCRUB_REPORT.md` listing what was masked.
- `agents/researcher.md`: the read-only researcher subagent's definition (a copy of `~/.claude/agents/researcher.md`).
