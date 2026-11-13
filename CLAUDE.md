CloudCrane challenge: Job Tracker
Goal: build a private phone app through the CloudCrane MCP and ship at least 3 versions. Deadline: Sat 14 Nov, 11:59 PM IST. Submit: the app's page link, about 300 words (what I built and why, what changed in each version, one thing that broke and how it was fixed, honest feedback on CloudCrane), and optionally a share link to the AI chat.
Model roles

* Opus (main): builds every version, designs data changes and migrations, decides whether to ship or roll back, and does the final review before each publish.
* `researcher` subagent (Sonnet): reads the CloudCrane guides and runs the tests, then reports back so Opus can verify. Use it at three points only: reading the guides before v1, the test pass before each version is called done, and the v2 migration check.

The app
An app spec (path A, spec-native), kept in `spec.json` and published with `publish_spec` / `update_spec`. Structured as:

1. Storage: `data.key` is `jt.data`, persisted at `/saved`. CloudCrane owns the storage; the spec only describes it.
2. Migrations: `data.version` is the data version. Each change of saved shape raises it and adds one CEL entry to `data.migrations` (`{from, to, cel}`), step by step. A migration never drops fields it doesn't recognise, and the item schema never uses `additionalProperties: false`, so older and newer data both stay readable. CloudCrane does not keep a pre-migration copy on the device (checked in a v2 preview), so each migration stores the untouched old object in the saved data itself (v2: `v1Backup: saved`), unknown fields included. The migration CEL gets the saved object as `saved` (without the runtime's own `v` key), and its result replaces the saved data whole.
3. State/logic: CEL rules in `logic` (functions, computed values, vectors) for derived values such as days since the last update.
4. UI: A2UI components in `surface`, list view and add/edit form, mobile-first. Offline comes with the platform.

Saved data starts versioned: `{ schemaVersion: 1, nextId: 1, apps: [{ id, company, role, appliedOn, status, notes }] }`, with `data.version: 1`. `schemaVersion` mirrors `data.version` so exports describe themselves. `id` comes from the saved `nextId` counter, never from list position; edits find their item by `id`. Keep form labels and button text stable across versions, because data tests drive the previous version's screens.
Versions

* v1: add, list and edit applications, each with a single status.
* v2 (data change): `status` becomes `stages: [{ stage, date }]`. Migrate existing entries to one stage that uses their `appliedOn` date. Save a backup of the v1 data before migrating. Keep writing `status` (set to the latest stage) so v1 can still read everything if v2 is rolled back.
* v3: driven by real use. Candidates: a follow-up flag after N days with no change, a filter by stage, and export/import of the saved data.

Rules

* Create the app with `publish_spec` and `private: true`. Never use `publish_app`, because it always starts public. Only make the app public when I say so.
* No real personal details, in the app or in test data. Use made-up companies.
* Before each version is done: run `check_publish`, then `try_app`, then `look_at_app`. The cross-version data check (data saved in the previous version must read back in the new one) comes from the data tests that run on publish; read their results via `get_spec`. `try_app` is only for flows within one version. Both `try_app` and `look_at_app` are limited to 50 runs a day.
* Update the blueprint and its tests in the same change as the code, every time.
* If a version makes things worse, use `rollback_app` and log it.
* Never call `delete_app` without asking me.
* Stop after each version and wait for me to try it on my phone.

BUILD_LOG.md
After each version, append: the version and a one-line summary, any saved-data change and how it was migrated, test results, anything that broke and how it was fixed, and any CloudCrane friction noticed.
CloudCrane feedback found so far

* `publish_app` starts public with no option to make it private (`publish_spec` has one), and you're only told after it's done
* Tools carry no permission labels (read, write or destructive)
* `update_app` doesn't report its test results; they only show up in `get_spec`
* A rollback leaves no record and gives no check that saved data is compatible with the older version
* `look_at_app` had no score for v2 even after a minute
* Nothing warns when the blueprint drifts from the code
* `list_apps` says "installed version" but shows the version being served
* The server's instructions were cut off mid-sentence
* The random suffix added to app addresses isn't explained
* Deleting an app works cleanly: the link returns 410 and Explore returns 404
* The spec guide never shows a real migration or says what a CEL migration receives and returns (both examples have `migrations: []`)
* `private` exists only on `publish_spec`; after that, visibility changes only in the dashboard. No guide says whether a spec app can later be updated with hand-written HTML
* The backup and rollback rules for saved data ("offer the copy from before") appear only in the `rollback_app` tool description, not in any guide
* `kit: "page"` for coded components is in the schema but never described; test `state` paths differ between specs (`data.saved...`) and HTML (`window.appState`) with no explanation