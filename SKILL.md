---
name: chrome-line-extension-automation
description: Operate the official LINE Chrome extension app `ophjlpahpchlmihnnnihgmmeilfjmjjc` through Chrome MCP at `chrome-extension://ophjlpahpchlmihnnnihgmmeilfjmjjc/index.html#/chats`. Use when the task is to attach to the official extension app, open an exact LINE chat, read from page context or read-side APIs, send through internal page-context API invocation with DOM fallback, inspect runtime requests, or verify safe failure. Do not use for desktop LINE, toolbar popup work, Browser MCP current-tab tasks, alternative extensions, or raw API replay as the primary route.
---

# Chrome LINE Extension Automation

Use `chrome-devtools` as the primary tool. Treat the official LINE extension app as the product surface. Do not switch to Browser MCP, Playwright, or raw HTTP replay unless the user explicitly asks for a different route.

This shared skill does not install the LINE extension, install `chrome-devtools`, or perform LINE login on the user's behalf. It assumes a Codex environment where `chrome-devtools` is already available and a browser/profile where the official LINE extension can be opened at the explicit extension URL.

## Prerequisites

- A Codex environment where `chrome-devtools` is available.
- The official LINE Chrome extension installed in the same browser/profile that `chrome-devtools` will attach to.
- LINE already logged in and synced inside that extension surface before any controlled read or send.
- An exact chat id or exact chat name for controlled chat work. Shared copies of this skill should not depend on a private default target.
- Read [references/onboarding.md](references/onboarding.md) for first-run setup, install/login expectations, and shareability rules.

## Fit

- The target is the official LINE Chrome extension app at the explicit extension URL.
- The task needs page-context reads, internal API-backed sends, runtime request inspection, or safe-failure checks.
- The caller can provide an exact chat id or exact chat name.

## Non-Fit

- Desktop LINE app tasks.
- Toolbar popup tasks.
- Browser MCP current-tab or current-profile tasks.
- Alternative or unofficial LINE extensions.
- Raw request replay as the primary control path.

## Workflow

1. Attach to the explicit extension target.

- Navigate directly to `chrome-extension://ophjlpahpchlmihnnnihgmmeilfjmjjc/index.html#/chats`.
- Do not depend on `list_pages` discovering the extension tab. Use the explicit URL as the primary attach path.
- Treat attach as successful only when the snapshot root is `LINE` at the extension URL.

2. Run the session and identity gate before any read or send.

- Install the shared helper from [references/runtime.md](references/runtime.md), then run the `session_status` snippet.
- Require a healthy result before continuing: store available, `auth.profile` present, route available, and the extension surface not stuck on a blank or login-only view.
- Treat any browser-specific install path, version, and `ltsmSandbox.html` iframe cue in [references/runtime.md](references/runtime.md) as debugging aids, not mandatory preconditions if the explicit extension route is already healthy.
- If attach or session health fails, stop and report the missing prerequisite from [references/onboarding.md](references/onboarding.md) instead of improvising a different control path.
- If the session is unhealthy, stop and report the environment problem instead of trying to send.

3. Resolve the exact target chat and verify the composer.

- If the caller provides an exact chat id or exact chat name, resolve it first through page context. Stop on ambiguity.
- Shared copies of this skill must not ship a real private chat id, chat name, or chat URL as a default target.
- Prefer an exact row match in the UI once the expected target is known. Use `Go chatroom` and `Enter a message` as last-validated UI cues, not universal invariants.
- Treat navigation as successful only when the route id, store current chat id, and expected target id all match, and the composer is visible.

4. Choose the correct branch before the controlled action.

- For API inspection, benchmark, or send verification tasks, install the detailed request hook from [references/runtime.md](references/runtime.md) before any controlled read or send.
- For a quick read-only task, skip request hooking unless it is needed for verification.
- Never let the first controlled send happen before the instrumentation you need is already live.

5. Read through page context, not DOM alone.

- Prefer the page-context helpers from [references/runtime.md](references/runtime.md) to read session status, current chat id, and recent message rows.
- Wait for the React root and Redux store to exist before reading. If the helper returns no store, reload once and re-check. Do not blind-retry beyond that.
- Use the UI snapshot as a secondary confirmation, not the only source of truth.
- For benchmark-grade or server-backed reads, reload with the request hook already installed and confirm the target chat issues `getRecentMessagesV2` or `getMessageReadRange`.

6. Send through internal page-context API invocation first.

- Never use raw HTTP replay as the primary send path.
- Extract the composer chain's internal submit handle from [references/runtime.md](references/runtime.md) and use `onSubmit([payload], [])` as the primary send path.
- Treat this as the extension's own runtime path: it should generate `sendMessage` and the runtime-bound headers from inside the page context.
- For benchmarks or probes, always use unique tokens in the payload.
- Only send after the expected target chat is verified by route and store state.
- Treat the first local optimistic message as provisional, not final success.

7. Fall back to DOM only when the internal handle is unavailable.

- Use the DOM composer helpers from [references/runtime.md](references/runtime.md) only if the internal submit handle cannot be resolved before any send has happened.
- Do not mix an uncertain API-path send with a follow-up DOM send in the same attempt. That is how duplicates happen.
- Keep the DOM path as an explicit fallback, not the default route.

8. Verify independently after every send.

- Verify the exact payload appears in the target thread quickly, either in the UI or as a local optimistic store row.
- Then force server-backed verification: reload or perform an equivalent server-backed reread, rerun the token-state check, and require `serverCount = 1` and `localCount = 0`.
- Treat transport-only success, route drift, `wrong_chat`, `duplicate_send`, or `serverCount != 1` as failure.

9. Inspect the internal API only through the live runtime.

- Record method, URL, response status, and header names from the live runtime hook.
- When needed, record short request and response previews for `sendMessage`, `getRecentMessagesV2`, and `getMessageReadRange`.
- Treat `X-Hmac`, `X-Line-Access`, `X-Line-Chrome-Version`, and `X-LAL` as runtime-bound. Do not print or persist full secret values.
- Use raw replay only as a negative control to demonstrate safe failure.

10. Fail safely.

- If the explicit extension route does not attach, if the session is not healthy, if the target chat cannot be verified exactly, if the internal handle is unavailable and DOM fallback is also unavailable, or if verification cannot reach server-backed state, stop.
- If raw replay fails with `REQUEST_MUST_UPGRADE` or another 4xx, do not retry it.
- Prefer explicit failure over improvising a different automation surface.

## References

- Read [references/onboarding.md](references/onboarding.md) for prerequisites, first-run setup, failure meanings, and what to share.
- Read [references/runtime.md](references/runtime.md) for the extension id, entry URL, shared page-context helpers, and request-hook snippet grounded in the live Track 3 run.
