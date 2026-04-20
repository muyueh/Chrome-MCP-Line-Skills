# Runtime

Run the snippets in this file through `chrome-devtools` page-script evaluation against the attached extension page.

## Stable Extension Target

- Extension id: `ophjlpahpchlmihnnnihgmmeilfjmjjc`
- Entry URL: `chrome-extension://ophjlpahpchlmihnnnihgmmeilfjmjjc/index.html#/chats`
- One observed install root pattern on this machine:
  `~/Library/Application Support/com.openai.atlas/browser-data/**/Extensions/ophjlpahpchlmihnnnihgmmeilfjmjjc/*/manifest.json`
- One observed version from a live run: `3.7.2`

Treat any install path or version as a debugging clue only. The shared contract is the extension id plus the explicit entry URL, not a specific local browser implementation detail.

The extension must be installed in whichever browser/profile `chrome-devtools` is actually attaching to. If the route uses a dedicated browser/profile, installing LINE only in the user's everyday Chrome profile is not enough.

## Shared-Copy Target Policy

- Do not hardcode a real chat name, real chat id, or direct chat URL in the shared copy of this skill.
- Require the caller to provide an exact chat id or exact chat name for controlled reads and sends.
- If a local operator keeps a convenience target for private use, keep it outside the distributed skill files.

## Shared Page-Context Helper

Run this once after attach. It installs a reusable helper on `window.__lineSkill`.

```js
() => {
  window.__lineSkill = window.__lineSkill || {};
  window.__lineSkill.getStore = () => {
    const root = document.getElementById('root');
    if (!root) return null;
    const key = Object.keys(root).find(k => k.startsWith('__reactContainer$'));
    if (!key) return null;
    function findStore(node, seen = new Set()) {
      if (!node || seen.has(node)) return null;
      seen.add(node);
      if (node.memoizedProps?.store?.getState) return node.memoizedProps.store;
      return findStore(node.child, seen) || findStore(node.sibling, seen) || null;
    }
    return findStore(root[key]);
  };
  window.__lineSkill.getState = () => {
    const store = window.__lineSkill.getStore();
    return store ? store.getState() : null;
  };
  return {
    ok: typeof window.__lineSkill.getStore === 'function',
    hasRoot: !!document.getElementById('root'),
  };
}
```

## Install API-First Helpers

Run this after the shared helper. It adds the primary internal send path and the token-state verifier.

```js
() => {
  window.__lineSkill = window.__lineSkill || {};
  window.__lineSkill.resolveSubmitHandle = () => {
    const el =
      document.querySelector('textarea-ex[placeholder="Enter a message"]') ||
      document.querySelector('textarea-ex') ||
      document.querySelector('textarea') ||
      document.querySelector('[contenteditable="true"]');
    if (!el) return { ok: false, reason: 'composer_unavailable' };
    const fiberKey = Object.keys(el).find(k => k.startsWith('__reactFiber$'));
    let fiber = fiberKey ? el[fiberKey] : null;
    let depth = 0;
    while (fiber && depth < 12) {
      const props = fiber.memoizedProps || {};
      if (typeof props.onSubmit === 'function') {
        return {
          ok: true,
          source: 'composer_chain',
          messageBoxId: props.messageBoxId || null,
          depth,
          onSubmit: props.onSubmit,
        };
      }
      fiber = fiber.return;
      depth += 1;
    }
    return { ok: false, reason: 'internal_submit_unavailable' };
  };
  window.__lineSkill.sendViaInternalSubmit = payload => {
    const text = String(payload);
    const handle = window.__lineSkill.resolveSubmitHandle();
    if (!handle.ok) return handle;
    handle.onSubmit([text], []);
    return {
      ok: true,
      method: 'internal_onSubmit',
      messageBoxId: handle.messageBoxId,
      payload: text,
    };
  };
  window.__lineSkill.verifyTokenState = token => {
    const state = window.__lineSkill?.getState?.();
    if (!state) return { ok: false, reason: 'store_unavailable' };
    const currentId =
      state.messageBox?.currentMessageBoxId ||
      state.messageBox?.selectedMessageBoxId ||
      state.chat?.currentMessageBoxId ||
      location.hash.split('/').filter(Boolean).at(-1) ||
      null;
    const ids = state.message?.lists?.[currentId]?.data || [];
    const localIds = state.message?.localLists?.[currentId] || [];
    const entities = state.message?.entities?.messages || {};
    const serverRows = ids
      .map(id => entities[id])
      .filter(Boolean)
      .filter(m => (m.text || '') === token);
    const localRows = localIds
      .map(id => entities[id])
      .filter(Boolean)
      .filter(m => (m.text || '') === token);
    return {
      ok: true,
      currentId,
      serverCount: serverRows.length,
      localCount: localRows.length,
      serverRows: serverRows.slice(0, 3),
      localRows: localRows.slice(0, 3),
    };
  };
  return {
    ok: true,
    installed: [
      'resolveSubmitHandle',
      'sendViaInternalSubmit',
      'verifyTokenState',
    ],
  };
}
```

## Session Status

Use this before any read or send.

```js
() => {
  const state = window.__lineSkill?.getState?.();
  if (!state) {
    return {
      ok: false,
      reason: 'store_unavailable',
      hash: location.hash,
    };
  }
  const routeId = location.hash.split('/').filter(Boolean).at(-1) || null;
  const currentId =
    state.messageBox?.currentMessageBoxId ||
    state.messageBox?.selectedMessageBoxId ||
    state.chat?.currentMessageBoxId ||
    routeId;
  const profileMid = state.auth?.profile?.mid || null;
  const sandboxVisible = !!document.querySelector('iframe[src*="ltsmSandbox.html"]');
  const composerVisible =
    !!document.querySelector('textarea-ex[placeholder="Enter a message"]') ||
    !!document.querySelector('textarea-ex') ||
    !!document.querySelector('textarea') ||
    !!document.querySelector('[contenteditable="true"]');
  return {
    ok: !!profileMid,
    hash: location.hash,
    routeId,
    currentId,
    profileMid,
    sandboxVisible,
    composerVisible,
    stateKeys: Object.keys(state),
  };
}
```

## Resolve Exact Chat Target

Use this only when the caller provides an exact chat id or exact chat name. Shared copies should not inject a private default target.

```js
(rawTarget) => {
  const state = window.__lineSkill?.getState?.();
  if (!state) return { ok: false, reason: 'store_unavailable' };
  const target = String(rawTarget || '').trim();
  if (!target) return { ok: false, reason: 'target_required' };
  const sources = [
    state.messageBox?.entities?.messageBoxes,
    state.messageBox?.entities,
    state.chat?.entities?.messageBoxes,
    state.chat?.entities,
  ].filter(Boolean);
  const rows = [];
  for (const source of sources) {
    for (const [id, row] of Object.entries(source)) {
      if (!row || typeof row !== 'object') continue;
      const name =
        row.name ||
        row.chatName ||
        row.displayName ||
        row.title ||
        row.label ||
        null;
      if (typeof name !== 'string' || !name.trim()) continue;
      rows.push({ id, name: name.trim() });
    }
  }
  const unique = Array.from(new Map(rows.map(row => [row.id, row])).values());
  const exact = unique.filter(row => row.id === target || row.name === target);
  const partial = unique.filter(row => row.name.includes(target)).slice(0, 10);
  return {
    ok: exact.length === 1,
    target,
    exact,
    partial,
    totalCandidates: unique.length,
  };
}
```

## Read Current Chat Rows

```js
() => {
  const state = window.__lineSkill?.getState?.();
  if (!state) return { ok: false, reason: 'store_unavailable' };
  const currentId =
    state.messageBox?.currentMessageBoxId ||
    state.messageBox?.selectedMessageBoxId ||
    state.chat?.currentMessageBoxId ||
    location.hash.split('/').filter(Boolean).at(-1) ||
    null;
  const ids = (state.message?.lists?.[currentId]?.data || []).slice(0, 8);
  const entities = state.message?.entities?.messages || {};
  return {
    ok: !!currentId,
    currentId,
    rows: ids.map(id => {
      const m = entities[id] || {};
      return {
        id,
        text: m.text || null,
        from: m._from || m.from || null,
        to: m.to || null,
        createdTime: m.createdTime || null,
      };
    }),
  };
}
```

## Install Detailed Request Hook

Use this before a controlled read or send when the task is API discovery, benchmark verification, or request-shape inspection.

```js
() => {
  window.__lineSkill = window.__lineSkill || {};
  if (window.__lineSkill.requestHookInstalled && window.__lineSkill.requestsDetailed) {
    return {
      ok: true,
      alreadyInstalled: true,
      buffered: window.__lineSkill.requestsDetailed.length,
    };
  }
  window.__lineSkill.requestsDetailed = [];
  const origOpen = XMLHttpRequest.prototype.open;
  const origSetHeader = XMLHttpRequest.prototype.setRequestHeader;
  const origSend = XMLHttpRequest.prototype.send;
  XMLHttpRequest.prototype.open = function(method, url, ...rest) {
    this.__lineSkillMeta = { method, url, headers: [] };
    return origOpen.call(this, method, url, ...rest);
  };
  XMLHttpRequest.prototype.setRequestHeader = function(name, value) {
    if (this.__lineSkillMeta && typeof name === 'string') {
      this.__lineSkillMeta.headers.push(name);
    }
    return origSetHeader.call(this, name, value);
  };
  XMLHttpRequest.prototype.send = function(body) {
    this.addEventListener('loadend', () => {
      const meta = this.__lineSkillMeta || {};
      window.__lineSkill.requestsDetailed.push({
        method: meta.method || null,
        url: meta.url || null,
        status: this.status,
        headerNames: Array.from(new Set(meta.headers || [])),
        bodyType: body == null ? 'none' : typeof body,
        bodyPreview: typeof body === 'string' ? body.slice(0, 300) : null,
        responsePreview:
          typeof this.responseText === 'string'
            ? this.responseText.slice(0, 400)
            : null,
      });
    });
    return origSend.call(this, body);
  };
  window.__lineSkill.requestHookInstalled = true;
  return { ok: true, buffered: 0 };
}
```

## DOM Fallback: Inject Composer Text

Only use this if `sendViaInternalSubmit()` is unavailable before any send attempt has happened.

```js
(payload = 'REPLACE_ME') => {
  const text = String(payload);
  const host =
    document.querySelector('textarea-ex[placeholder="Enter a message"]') ||
    document.querySelector('textarea-ex');
  const inputEl = host?.inputEl || host?._inputEl || null;
  if (inputEl) {
    inputEl.focus();
    inputEl.value = text;
    inputEl.dispatchEvent(new Event('input', { bubbles: true, composed: true }));
    return { ok: true, method: 'line_textarea_ex', value: inputEl.value };
  }
  const nativeTextArea = document.querySelector('textarea');
  if (nativeTextArea) {
    nativeTextArea.focus();
    nativeTextArea.value = text;
    nativeTextArea.dispatchEvent(new Event('input', { bubbles: true, composed: true }));
    return { ok: true, method: 'native_textarea', value: nativeTextArea.value };
  }
  const editable = document.querySelector('[contenteditable="true"]');
  if (editable) {
    editable.focus();
    editable.textContent = text;
    editable.dispatchEvent(new Event('input', { bubbles: true, composed: true }));
    return { ok: true, method: 'contenteditable', value: editable.textContent };
  }
  return { ok: false, reason: 'composer_unavailable' };
}
```

Follow this with DevTools `press_key` using `Enter`. Treat `Enter a message` as a last-validated cue, not the only acceptable composer signature.

## Runtime-Bound Request Clues

Observed live send path:

- `POST /api/talk/thrift/Talk/TalkService/sendMessage`
- `POST /api/talk/thrift/Talk/TalkService/sendChatChecked`

Observed live read path:

- `POST /api/talk/thrift/Talk/TalkService/getRecentMessagesV2`
- `POST /api/talk/thrift/Talk/TalkService/getMessageReadRange`

Observed runtime-bound header names:

- `X-Hmac`
- `X-Line-Access`
- `X-Line-Chrome-Version`
- `X-LAL`

Do not use these observations to justify raw replay as the default path. The live negative control returned `HTTP 400` with `REQUEST_MUST_UPGRADE`.

Important verification nuance from the live benchmark:

- a successful internal send may appear first as a local optimistic message
- final success should be asserted only after reload or equivalent server-backed reread
- the success contract is `serverCount = 1` and `localCount = 0`
