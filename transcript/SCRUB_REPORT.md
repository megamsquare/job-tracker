# Scrub report

Each item below was replaced with `[removed]` in the transcripts. Values are never shown here.

**Scanned for:** email addresses; API keys and tokens (OpenAI/Anthropic `sk-`, GitHub, AWS, Slack, JWTs); bearer/auth headers; cookies; credentials (password, secret, key or token assignments); IP addresses; personal details unrelated to the project. Local file paths were kept, as requested.

**Left out before scanning** (never written to the transcripts): thinking blocks, system messages and reminders (including the account context, which holds your email address), task notifications, and the full tool outputs (only short summaries are kept).

| # | File | Line | Type | Count | Where |
|---|---|---|---|---|---|
| 1 | `transcript/session-2-build.md` | 779 | personal detail: local timezone name (reveals location) | 1 | tool result |
| 2 | `transcript/session-2-build.md` | 848 | personal detail: local timezone name (reveals location) | 1 | tool result |

**Worth a look:** the transcripts still contain your macOS username inside local paths (e.g. `/Users/<username>/…`, `/private/tmp/claude-501/-Users-<username>-…`), the app id and its install link, and your first name in the speaker headings. These were kept on purpose; mask them too if you prefer.
