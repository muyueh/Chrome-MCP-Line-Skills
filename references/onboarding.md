# Onboarding

## What This Skill Assumes

- `chrome-devtools` is already available in the user's Codex environment. This skill does not install the MCP server itself.
- The official LINE Chrome extension is installed in the exact browser/profile that `chrome-devtools` will attach to.
- The user can complete LINE login inside that extension surface.

If `chrome-devtools` is missing, stop and fix the Codex environment first. That setup is outside this skill.

## Minimum Setup For A New User

1. Confirm the Codex environment exposes `chrome-devtools`.
2. Install the official LINE Chrome extension in the browser/profile used by that route.
3. Open `chrome-extension://ophjlpahpchlmihnnnihgmmeilfjmjjc/index.html#/chats`.
4. Log in to LINE inside the extension and wait until chats load.
5. Run the shared helper and `session_status` from [runtime.md](runtime.md).
6. For any controlled read or send, provide an exact chat id or exact chat name and stop on ambiguity.

The smallest healthy ready signal is:

- attach succeeds at the explicit extension URL
- `session_status.ok === true`
- `session_status.profileMid` is present

If those are not true, treat the environment as not ready.

## How To Run The JS Snippets

- Run the snippets in [runtime.md](runtime.md) through `chrome-devtools` page-script evaluation against the attached extension page.
- Do not treat the shared instructions as "open Chrome DevTools and paste everything manually into the browser console."
- Run them in order: shared helper first, then `session_status`, then target resolution, then send/verify helpers only if needed.

## Required vs Recommended

Required:

- official LINE Chrome extension installed
- LINE logged in inside the extension app
- `chrome-devtools` can attach to the explicit extension URL

Recommended:

- use a dedicated browser/profile for this extension so attach and session state are stable
- use a test chat before allowing real sends
- keep any personal target chat notes outside the shared skill files

A dedicated browser/profile is recommended for stability and isolation, but it is not a hard prerequisite if `chrome-devtools` can already attach to the correct extension session.

## Failure Meanings

- Explicit extension URL will not attach:
  the LINE extension is missing from that browser/profile, or `chrome-devtools` is attached to a different browser/profile than the one where LINE was installed.
- `session_status` returns `store_unavailable`:
  the extension app is not fully loaded yet. Reload once, then re-check.
- `session_status` returns no `profileMid`:
  LINE login is incomplete or the session has expired.
- Exact target resolution is ambiguous:
  do not send. Require the user to provide an exact id or exact full name.
- Composer is unavailable:
  the target chat was not opened successfully, or the current extension surface is not the expected chats app.

## Sharing Rules

- Minimum shareable package:
  - `SKILL.md`
  - `agents/openai.yaml`
  - `references/runtime.md`
  - `references/onboarding.md`
- Share the skill folder, not private local notes.
- Do not ship a real chat id, real chat name, or direct chat URL inside the shared copy.
- Do not present machine-specific install roots or profile names as universal requirements.
- Do not ship run artifacts that may contain request previews, screenshots, runtime traces, local profile paths, or message metadata.
- If you are copying this skill out of a larger benchmark repo, do not include benchmark fixtures, reports, runs, or repo-specific docs.
