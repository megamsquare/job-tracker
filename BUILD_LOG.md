# Build log

- 2026-11-12: Smoke test app deleted after testing publish, update, version history and rollback.

## v1 — 2026-11-12 — add, list and edit applications, one status each
App: Job Tracker, private, https://job-tracker-k3lta7ebhm.cloudcrane.ai/ (path A: app spec via `publish_spec` with `private: true`).
CloudCrane versions: 1 (first publish), 2 (v1.1 fixes, live). Specs: `versions/v1.spec.json`, `versions/v1.1.spec.json`.

**Saved data:** `jt.data`, `data.version: 1`, `{ schemaVersion: 1, nextId: 1, apps: [{ id, company, role, appliedOn, status, notes }] }`. Item schema is open (no `additionalProperties: false`). `id` comes from the saved `nextId` counter (not `now()`: the test clock only moves on `advance`, so two adds in one test would get the same id). Edits find their item by id: a computed `selIndex` looks up the id's current position each time, and a `data.views` entry points `/ui/edit` at `/saved/apps/${/computed/selIndex}`. No migration (first version).

**Pre-publish v2 migration check (draft only, not published):** `drafts/v2-migration-check.spec.json` (`status` kept, `stages: [{stage, date}]` added, migration `{from:1, to:2}`) passed `check_publish`. What that shows about the CEL input:
- Nothing. `check_publish` does not compile or type-check migration CEL: a migration reading an undeclared variable (`bogusvar.apps.map(a, a)`) also passed. The only migration rule it enforces is the step chain ("steps go one version at a time, up to data.version").
- So the input variable name (`saved`, by analogy with rules) and the return shape stay unconfirmed until a real `update_spec` runs with a data test. The data test that runs on update (setup in the old version, check in the new) is the only real check.
- CEL has no map merge, so a migration has to list every field by name; "never drop unknown fields" can't be guaranteed in CEL. The v1 shape didn't need to change for v2.

**Tests (final, app version 2):** passed 3/3, proven 8/8, `against: 1` (T-3's setup ran in version 1, its check in version 2), browser time 51 s of the 60 s allowed.
- T-1 start empty, add, see row + "10 Nov 2026" + count + missing-fields hint. T-2 add two, edit the second's status, the first is unchanged. T-3 (data) add + edit, survives reload and update.
- `try_app` on version 1: edit-the-second-of-two flow passed every step (first entry's notes still empty, status still Applied).
- `look_at_app` version 2: no errors; title, count, button and empty text aligned. Quality 65/100 (Speed 17, Feel 93, Looks 100).

**What broke and how it was fixed:**
1. First publish: tests timed out ("took longer than the 60 s a run allows, so T-4 and T-5 didn't run"), no per-test results, yet version 1 went live anyway. Fixed by cutting 5 tests / 61 steps to 3 tests / 31 steps, each covering several journey steps.
2. The Save button's `checks` messages never show: tapping Save with an empty company does nothing visible (`try_app`: "Add the company first" not on the page). Fixed with a computed hint line ("Add the company and role to save.") under the form.
3. Each nested `List` (used to show/hide by mode) indented its content further, giving a staircase layout (x = 30, 64, 80, 96). Fixed by putting every block one `List` deep under the root, and the status picker in plain `Row`s of buttons instead of a nested list.
4. `check_publish` refused an object literal as a `set` value and a `personal` pointer of `/saved/apps` (it wants `/apps`, relative to the saved data).

**CloudCrane friction:**
- `check_publish` doesn't validate migration CEL at all (see above).
- A first publish whose tests fail (timeout) still goes live; the failure only shows in `get_spec`/`list_versions`.
- The test budget is 60 s for all tests together, and nothing says so before you hit it.
- Button `checks` messages aren't shown to the person.
- The runtime's own button is 31 px tall, below the 44 px its own check asks for, and a spec can't change it.
- Each `List` adds indentation, and showing or hiding things needs a List over a 0-or-1-item computed list, so layouts drift right.
- Speed scores 17/100 because the spec runtime is ~590 KB of script; nothing in a spec can fix that.
- `publish_spec`'s reply says the app is private "because its code or blueprint seems to hold personal details", not because `private: true` was passed, which leaves it unclear which one decided it.
- A ChoicePicker can only bind to a list of strings, so a single status stored as a plain string needs buttons instead.
- The app link opens an install page whose content the built-in browser couldn't read (empty accessibility tree).

## v2 — 2026-11-12 — stage history per application (saved-data change), numbered cards, status chips that follow the value
Live as CloudCrane version 7 (promoted from preview with `rollback_app` at 14:48 after you approved it). Spec: `versions/v2.spec.json`.

### What the migration CEL actually receives and returns
Observed by running the real preview (CloudCrane version 4) in a browser with six v1-shaped entries staged in its storage (`jt.data` = `{"v":1, schemaVersion:1, nextId:7, extraTop:"keep-me", apps:[…6 items, one with extraField…]}`, `cc:data` = `{"app":2,"v":1}`), then reloading:
- **Storage layout:** CloudCrane keeps the saved data in localStorage under `data.key` (`jt.data`) as `{ "v": <data version>, …saved fields }`, plus `cc:data` = `{ "app": <app version>, "v": <data version> }`. No IndexedDB, Cache Storage, OPFS or service worker on the preview origin.
- **Input:** the CEL gets the saved object as `saved`, with every field except the runtime's own `v`, unknown fields included (`v1Backup` came out with `schemaVersion, nextId, extraTop, apps`, and `extraField` still on item 3).
- **Output:** the CEL's result replaces the saved data whole, and the runtime adds `"v": 2`. Nothing is merged back: without the backup, `extraTop` and `extraField` were dropped.
- **No backup:** after the migration, the device held no pre-migration copy anywhere (localStorage, sessionStorage, IndexedDB, caches, OPFS). So the v2 migration stores its own: `'v1Backup': saved`.
- **Result on the six entries:** all six present, each with exactly one stage `{stage: status, date: appliedOn}`, both "Quillstone Labs" entries separate (ids 1 and 5), notes intact, `nextId` 7, `schemaVersion` 2.
- `check_publish` does evaluate `logic.vectors`, so a function carrying the same body as the migration (`migrateV1`) plus a vector tests the expression before publishing. It still doesn't check the migration entry itself.
- **Test runs so far (all previews, none promoted; version 2 = v1.1 still live):**
  - CloudCrane version 3: T-1 failed at step 13. `tap label "Offer"` finds nothing, because ChoicePicker chips are `<button aria-pressed>`, not labelled fields. Fixed by tapping `role: button, name: Offer`; `expect checked` can't read `aria-pressed`, so the chip check reads state `data.ui.editStatusList`. Verified in a browser: after tapping Offer and then Notes, Offer has the `a2ui-chip-selected` class, `aria-pressed="true"` and the primary colour, and no other chip does.
  - Version 4: T-1 step 3 (type Company) "took longer than 5 s". `try_app` on the unchanged live version 2 failed the same way at the same time, so it was the runner, not the app; it recovered minutes later.
  - Version 5 (identical to 4): "took longer than the 60 s a run allows, so T-2 from version 2 and T-3 from version 2 didn't run". The run also includes the previous version's passing tests, which aren't in the new spec, and they share the 60 s budget. This isn't documented anywhere. The new T-2 also hit a "took longer than 5 s" typing timeout.
- **Friction:** `update_spec` with `preview: true` still says "goes live by itself when its tests pass"; get_spec shows `goesLive: false`. Previous versions' tests are re-run silently inside the same 60 s budget. The test runner's typing step intermittently times out on unchanged versions. The preview lives at its own address, so it opens with empty storage and can't see a phone's existing data.
- **Diagnostic (version 6, data test only):** inconclusive. It hit the runner's 5 s typing timeout at setup step 3, inside v1.1, before reaching anything else.
- **Version 7 (the v2 candidate, preview, `versions/v2.spec.json`):** passed. `testRun`: `"against":2`, `"carried":[T-1, T-2, T-3 all passed]` (v1.1's tests, re-run against v2), `"results":[T-1 passed, T-2 passed]`, `browserMs` 25244, proven 9/10. That confirms CloudCrane re-runs the previous version's passing tests as `carried` inside the same 60 s budget. The 10th item is P-3 ("changing one application never changes another"): no v2 test proves it now, though v1.1's carried T-2 still exercises it.
  - T-1 (behaviour, 15 steps): start empty, add, see "#1 ·" and "Product designer · Applied · 12 Nov 2026", open it, tap the Offer chip, tap into Notes, state `data.ui.editStatusList` == ["Offer"], Done, reopen, see "Applied · 12 Nov 2026" and "Offer · 12 Nov 2026".
  - T-2 (data, 10 steps): setup in v1.1 adds Brightwater & Finch as Interviewing; check in v2 sees it, opens it, sees "Interviewing · 12 Nov 2026", `data.saved.nextId` == 2 and `data.saved.schemaVersion` == 2.

### Promotion and checks
- **Saved-data change:** `data.version` 1 → 2. Each item gains `stages: [{stage, date}]`. Migration `{from:1, to:2}`: `{'schemaVersion': 2, 'nextId': saved.nextId, 'v1Backup': saved, 'apps': saved.apps.map(a, {id, company, role, appliedOn, status, notes, 'stages': [{'stage': a.status, 'date': a.appliedOn}]})}`. `status` is still written and always equals the latest stage (Save sets both, Done sets both). Items still have no `additionalProperties:false`.
- **Behaviour:**
  - The status picker is a chip ChoicePicker bound to a one-item list, so the highlight follows the value, not focus (checked in a browser after tapping into Notes).
  - Done adds `{stage, date: today}` only when the status changed; unchanged adds nothing.
  - The edit form shows the stage history, oldest first.
  - Cards read "#n · Company", then "Role · Status · d MMM yyyy". The number is display order, not saved.
  - All v1 labels and button text are unchanged.
- **Tests:** version 7 passed. Results T-1 and T-2 passed; carried v1.1 tests T-1, T-2 and T-3 passed; 25 s; proven 9/10, with P-3 not proven by a v2 test.
- **look_at_app (version 7):** no errors. Quality 65/100 (Speed 17, Feel 93, Looks 100); the "Add application" button is 123×31.
- **Not verified yet:** the migration on your phone's six real entries. It was verified on six staged v1 copies in the preview, and `v1Backup` holds the untouched v1 data if anything is wrong.

### What broke and how it was fixed
1. Version 3: `tap label "Offer"` found nothing, because chips are buttons, not labelled fields. Fixed with `tap role button name Offer`, and the chip check reads state.
2. `pickStatus` returned null because its parameter was named `list`, a CEL type name; caught by a vector in `check_publish`. Renamed to `chosen`.
3. CloudCrane keeps no pre-migration copy (CLAUDE.md said it did). Added `v1Backup: saved` to the migration and corrected CLAUDE.md.
4. Versions 4 to 6: the runner's typing step timed out (> 5 s) even on unchanged v1.1, and v1.1's carried tests pushed version 5 past 60 s. Fixed by cutting to one behaviour test plus the data test (25 steps) once the runner recovered.

### CloudCrane friction
- `update_spec` with `preview: true` still says "goes live by itself when its tests pass"; get_spec shows `goesLive: false`.
- Previous versions' passing tests are re-run as `carried` in the same 60 s budget, which isn't documented.
- The test runner's typing step intermittently times out on unchanged versions.
- The preview runs at its own address with empty storage, so it can't be tried against a phone's existing data.
- No pre-migration copy is kept, despite `rollback_app`'s "offer the copy from before".
- A migration result replaces the saved data whole, so unknown fields are dropped unless the CEL keeps them.
- `expect checked` can't read `aria-pressed` chips.
- A CEL parameter named `list` silently returns null.
- Promoting a preview is done with `rollback_app`, whose reply says "live again" for a version that was never live.
- `look_at_app` only ever shows the empty state, so it can't show the list or the edit form.
- Six CloudCrane versions (2 to 7) for one app version; failed previews stay in the version list.

## v3 (in preview, not promoted) — 2026-11-12 — dates on iPhone, same-day stage rules, history cleanup, sorting, summary
Spec: `versions/v3.spec.json`. Previews: CloudCrane versions 8, 9, 10 (10 is the candidate). Version 7 (v2) is still live.

### The date bug: passed tests, failed on a real phone
- **Symptom (your iPhone):** cards read "Role · Applied ·" and the history read "Applied ·", with no date. The v2 tests checked "Product designer · Applied · 12 Nov 2026" and passed.
- **Cause:** CloudCrane's runtime `formatDate` adds a `Z` to any value without a timezone, so the stored `appliedOn` "2026-11-02" became `new Date("2026-11-02Z")`. Checked directly: JavaScriptCore (Safari, every iPhone browser) returns `Invalid Date` for that string, and V8 (Chromium, which the test runner uses) parses it. So `formatDate` returned "" on iPhone and "2 Nov 2026" in tests. The saved data was fine: real input and test input are the same "YYYY-MM-DD" string. The difference is the browser engine, not the data shape.
- **Was P-3 the card date?** No. P-3 is "Changing one application never changes another". The card date was tested (v2 T-1), and it passed because the runner is Chromium.
- **Fix:** v3 formats dates in CEL (`prettyDate`): `timestamp(d + 'T00:00:00Z')` and then `getDate()`, `getMonth()` and `getFullYear()` (the UTC getters with no argument). It doesn't use the runtime's `formatDate`. Vectors check "2026-11-02" → "2 Nov 2026" inside `check_publish`. A Chromium-only test can't catch a Safari-only parse bug, so the guard is: never hand a date-only string to `formatDate`, and the vectors check our own formatting.
- First attempt (version 8) used `getDate('UTC')`. The runtime builds those as `new Date(h.toLocaleString("en-US", {timeZone}))`, and the rows didn't render under the test runner's frozen clock. A `try_app` run without the clock showed the rows fine. Switched to the no-argument getters.

### Data change (schema 3, migration {from:2, to:3})
- Per item, `cleanStages` merges consecutive identical stages and collapses same-day stages into that day's last state. It never removes the first stage. `status` is recalculated from the cleaned last stage.
- `v2Backup` keeps every item's v2 stages. `v1Backup` and `nextId` are carried over.
- **Checked in a browser** on six staged v2 entries in the preview: Lumen Harbor (Interviewing 28 Oct, Offer 12 Nov, Interviewing 12 Nov, Offer 12 Nov) became Interviewing (28 Oct) → Offer (12 Nov), and the other five were unchanged. The migration CEL can call spec functions (`cleanStages`, through `fold`).
- **Done rule:** the same `cleanStep` function. Same status adds nothing; going back to the previous stage on the same day removes today's; any other same-day change replaces today's; a change on a new day adds a stage; the first stage is never replaced. Checked in a browser: Orchard Vale Applied (8 Nov) → Offer → Applied the same day left only "Applied · 8 Nov 2026".

### Polish
- Sorted by date applied, newest first (`byDateDesc`, using `.sort()` on keys).
- Summary line under the count, for example "2 Applied · 1 Interviewing · 1 Offer · 2 Rejected".
- **Cards:** "#n ·" and the company sit in a tappable row, and "Role · Status · date" is plain text outside the button, so it's left-aligned. The token `button.margin: 0` lines buttons up with the text.
- **Limit:** a company name long enough to wrap is centred inside its button. A native `<button>` centres text, and nothing in the spec can change that.

### Tests
- T-1 (17 steps): add, see "#1 ·", the dated card and "1 Applied"; change to Offer and back the same day; the state shows only the original stage, and the history shows "Applied · 12 Nov 2026".
- T-2 (data, 18 steps): setup in v2 creates Applied → Offer → Applied → Offer on the same day; the check in v3 sees the dated card, a cleaned history `[Applied, Offer]`, and `v2Backup` holding all four.
- **Version 8:** failed. Rows missing under the frozen clock (fixed as above), and tapping "Quillstone Labs" found nothing, because a tap by text must match the whole text and the title had become "#1 · Quillstone Labs" (fixed by giving the name its own text). v2's carried data test asserts `schemaVersion == 2`, which v3 must break, so the data promise moved from P-1 to P-6, as the runner's message suggests.
- **Versions 9 and 10 (identical):** every test, including v2's carried T-1, timed out at its first typing step ("took longer than 5 s"). At the same time a `try_app` run (no clock) on version 9 passed every step of the flow: add, the dated card, tapping the bare name, Offer, the dated history. The add form those tests type into is unchanged since v2, whose tests passed. Treated as the publish runner stalling, as during v2, not as a v3 bug. Not yet passed.

### v3 preview check: verified on iPhone Safari (your check, version 10)
The test runner is Chromium, so it can't cover Safari. You checked the v10 preview in Safari on your iPhone and everything passed:
- Dates show on the card and in the history, e.g. "Product Engineer · Applied · 12 Nov 2026" and "Applied · 12 Nov 2026".
- The summary line updates with the status.
- "#1 · Orchard Vale" sits on one left-aligned line.
- Stage rules: Applied → Offer (Done) gives two dated stages; Interviewing the same day replaces Offer; going back to Applied leaves only the original stage.

Sorting wasn't checked on the phone, so version 11 adds it to T-1: advance the test clock one day, reload, add Lumen Harbor, see "Data analyst · Applied · 13 Nov 2026", and check rows 0 and 1 are Lumen Harbor then Quillstone Labs. The app code is identical to version 10.

### Publish-test attempts for v3 (version 11; versions 9 and 10 had identical code)
- **Attempt 1, sent 15:42:17 UTC, ran 15:42:59:** failed. T-1 step 3, T-2 step 2 and v2's carried T-1 step 3 all "took longer than 5 s" (the first typing step); `results` were empty. A no-clock `try_app` on version 11 at 15:43 also stalled at its first typing step, so the runner itself was stalling. Not promoted.
- **Runner probe, about 16:03 UTC (no attempt spent):** `try_app` on version 11 and on the live, previously passing version 7 both stalled at step 2, the first typing step ("took longer than 5 s"). The runner was down for every version, so attempt 2 was held back.
- **Probe 1, about 16:26 UTC:** `try_app` on the live version 7 passed (typing worked).
- **Attempt 2, sent 16:26:57 UTC as version 12 (same spec as 11), ran 16:27:41:** failed. T-1 step 3 and v2's carried T-1 step 3 "took longer than 5 s" (first typing step). T-2's setup typed and saved in v2 but then couldn't find "Lumen Harbor" to tap at step 5; the same failure appeared in version 8's run. Not promoted.
- **Probe 2, about 16:29 UTC:** `try_app` on version 12 stalled at step 3 (typing). A control run on the live version 7 straight after also stalled at step 2 (typing). So the runner is stalling for every version.
- **Stopped as instructed, about 16:30 UTC.** No timers running.
  - **Live:** version 7 (v2, data version 2). Your phone is unchanged.
  - **v3 candidate:** version 12 is a preview with identical code to version 10, which you verified on iPhone Safari. Spec: `versions/v3.spec.json`.
  - **Tomorrow:** probe with `try_app` on version 7. Once typing works, re-send `versions/v3.spec.json` with `update_spec` and `preview: true`, and promote only on a passing run.
  - **Two things to watch on restart:**
    1. T-2's setup step 5 (tap "Lumen Harbor" in v2 after Save) failed in both runs that got that far (versions 8 and 12). In v2 the card's name is its own text inside the button, so the tap should match. If it fails again with typing healthy, check whether the setup's Save went through, for example by adding an `expect see "Lumen Harbor"` before the tap.
    2. The runner's typing has stalled for long stretches today (about 14:24 to 14:31 and 15:19 to 16:30 UTC, with gaps), across unchanged versions too.
- **CloudCrane friction:** publish-time tests and `try_app` share a flaky typing step ("took longer than 5 s") that also hits unchanged, previously passing versions. Nothing reports runner health, so the only way to tell a runner outage from an app bug is to re-run a known-good version.

### v3 diagnosis (next session)
- **"No examples" warning cleared:** added vectors for `sortKey` (an October and a November key; the November one sorts first), `dropStep` (keeps a list's first item, drops its last) and `joinStep` (`"" + "1 Applied"`, then `"1 Applied" + "1 Offer"` → "1 Applied · 1 Offer"). `check_publish` now reports `"warnings":[]`. The same-day back-and-forth, same-day replace, new-day add and same-status cases are `cleanStep`'s job and were already covered by its vectors. `dropStep` and `joinStep` are a list helper and the summary joiner.
- **T-2 setup** now does `expect see "Lumen Harbor"` before tapping it, to tell a missed Save from a missed tap.
- **Frozen-clock theory, version 8 vs 9 (`diff_versions`):** the only changes were the card title (one "#n · Company" text became a row of "#n ·" and the company), `prettyDate` dropping the `'UTC'` argument from its getters, and P-1 → P-6. None of these run on the add form, where the tests stall.
- **Per-keystroke work on the add form:** `addHint`, `draftStatus`, `draftStages` (no date parsing) and `summaryOf` (counts only, no dates). `listRows` (which uses `prettyDate`) is `[]` outside list mode, and `editHistory` runs only in edit mode. No date code runs on keystrokes there, and nothing loops or retries on Invalid Date. Nothing found to fix.
- **Trying to recreate the frozen clock in a browser didn't work:** the shell refuses iframes ("Frames can't be used in this app"), and a rewritten page doesn't restart the runtime.
- **Probe 1 of 3, 17:21 UTC:** `try_app` on the live version 7 stalled at step 2 (first typing step). A/B not run.
- **Probe 2 of 3, about 17:43 UTC:** `try_app` on version 7 passed (typing worked).
- **A/B, back to back:**
  - Re-sending the v2 spec unchanged, even with only the `generator` label changed, was refused as "unchanged" (CloudCrane compares the built page). A ran with only a test title changed.
  - **A = version 13 (unchanged v2 app), sent 17:45:53, ran 17:46:47: FAILED.** T-1 step 3 and T-2 step 2 "took longer than 5 s" (typing), and the carried v2 tests failed the same way. v2 passed these same tests at 14:42.
  - **B = version 14 (v3 with the new vectors and the "Lumen Harbor" check), sent 17:48:12, ran 17:48:51: FAILED, differently.** Typing worked. T-1 step 6 `see "#1 ·"` was "not on the page" after Save; the carried v2 T-1 failed at the same step. The new check in T-2's setup, `see "Lumen Harbor"` after Save, failed too, and that setup runs in the unchanged v2.
  - Versions 13 and 14 load the same runtime file as the live version 7 (`/_cc/runtime/0.4.0-fb23d7d56b0f.js`, same integrity hash).
- **Cause (as far as the evidence goes):** since about 15:00 UTC, CloudCrane's publish-time runner can't complete the add flow (type, Save, row appears) under the test clock, on any version of this app, including the unchanged v2 that passed at 14:42. Sometimes typing times out; sometimes typing goes in but Save produces no row (in v2 too). `try_app` without the clock passes the same flow on the same versions. The v3 code isn't the cause. Not promoted; v2 (version 7) stays live.
- **Friction:** identical specs are refused as "unchanged", so there's no clean way to re-test the current version; publish runs with a clock behave differently from `try_app`; nothing reports that the runner is degraded; failed runs return `results: []` and `browserMs: 0`, with no screenshot or log to diagnose from.
- **Stopped for the day at your request (after the A/B).** No timers running. v2 (version 7) is live. The v3 candidate is version 14, a preview with `versions/v3.spec.json`. Next step: re-run the A/B; promote v3 only if A (unchanged v2) passes again and B passes.

## v3 — 2026-11-12 — Promoted without a passing publish run (version 14, live 20:03:59 UTC)
This is a deliberate exception to the CLAUDE.md rule "promote only on passing tests", made by you after the A/B.

**Why:**
1. The A/B proved the publish runner fails the unchanged v2 app too (passed at 14:42, failed at 17:46), so the failure isn't in v3's code.
2. v3 was verified by hand in iPhone Safari (the version 10 preview check), and its full flow passes in `try_app`.
3. The live v2 had a known bug on every iPhone: no dates showed. v3 fixes it.
4. Backups are in place: `v1Backup` and `v2Backup` are both kept inside the saved data.

**A/B evidence:**
- A = version 13 (unchanged v2 app): typing timed out at T-1 step 3 and T-2 step 2.
- B = version 14 (v3): typing worked, but after Save "#1 ·" wasn't on the page, and the new `see "Lumen Harbor"` check failed in the setup, which runs in the unchanged v2.
- `try_app` without the clock passed the same flows on versions 7 and 9.
- All builds load the same runtime file, `0.4.0-fb23d7d56b0f.js`.

**Version 14 vs 10 (the one checked on iPhone), from `diff_versions` 10 → 14:**
- Only the new function examples (`sortKey`, `dropStep`, `joinStep`) and test steps (T-1's sorting check, T-2's `see "Lumen Harbor"`).
- The blueprint changes list is empty. No changes to screens, saved data, migrations, functions or computed values.

**After promotion:** `look_at_app` version 14 showed `errors: []`. Quality 67/100 (Speed 24, Feel 93, Looks 100). The empty state renders, and "Add application" now lines up with the text (button-margin token).

**Saved data:** data version 2 → 3 on the first open of the home-screen app. The migration `{from:2, to:3}` cleans the stages and keeps `v2Backup` (each item's old stages) and `v1Backup`.

**Page link:** none yet. `cloudcrane.ai/blueprints/job-tracker-k3lta7ebhm` returns 404 ("There's no public app at this address. It may have been made private or deleted."), because the app is private. A page exists only once you make it public (dashboard, Settings: Visibility).

### CloudCrane versions
1 = v1 · 2 = v1.1 · 3 = preview: tests failed (chip tap) · 4 = preview: tests failed (runner timeout) · 5 = preview: tests failed (60 s budget) · 6 = preview: tests failed (diagnostic) · 7 = v2 · 8 = preview: tests failed (v3 rows under clock, tap text) · 9 = preview: tests failed (runner) · 10 = preview: tests failed (runner; checked on iPhone) · 11 = preview: tests failed (runner) · 12 = preview: tests failed (runner) · 13 = preview: tests failed (A/B control, unchanged v2) · 14 = v3 (live, promoted without a passing run)

### Check: do `v1Backup` and `v2Backup` really exist after the v3 migration? (read-only, nothing published)
**Answer: yes.**
- **Why the question came up:** the Versions tab's "keeps" text for v14 reads "On this phone: schemaVersion, nextId, apps". That text lists only the schema's *required* fields. The blueprint's `shape` for the same key does include `v1Backup: object` and `v2Backup: list` (seen in the `check_publish` output since version 8). So the tab text is misleading, not a sign the backups are dropped.
- **Live v14 spec (`find_in_app` version 14):**
  - The schema declares `"v1Backup":{"type":"object"}` and `"v2Backup":{"type":"array"}`.
  - Migration 1→2: `'v1Backup': saved`.
  - Migration 2→3: `'v1Backup': has(saved.v1Backup) ? saved.v1Backup : {}`, plus `'v2Backup': saved.apps.map(a, {'id': a.id, 'stages': has(a.stages) ? a.stages : []})`.
- **Version 7 (the v2 your phone ran):** its 1→2 migration also writes `'v1Backup': saved`, so your phone's v2 data already held `v1Backup` before v3.
- **Runtime check in a browser** on the version 12 preview (same app code as v14), with six v2-shaped entries plus `v1Backup` staged in its storage:
  - **After the 2→3 migration:** the stored keys were `v, schemaVersion, nextId, v1Backup, v2Backup, apps`. `v1Backup` held 6 entries; `v2Backup` held 6 items, including Lumen Harbor's original four stages.
  - **After an edit (Done):** both were still present and unchanged.
  - **After adding an entry (Save):** both were still present (`nextId` 7 → 8, 7 apps).
  - So "a migration's result replaces the saved data" means the CEL's own output replaces it. Fields the CEL returns are kept, including the two backups. Only fields the CEL leaves out are dropped, and ordinary saves don't touch them.
- **`try_app` not used:** it always starts empty, so it can't stage your v2 data.
- **Friction:** the Versions tab's "keeps" text lists only required fields, which makes declared optional fields such as backups look as if they aren't kept.
