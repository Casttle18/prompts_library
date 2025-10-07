

---

## `manifest.json`

```json
{
  "manifest_version": 3,
  "name": "Prompt Notes",
  "version": "1.0.0",
  "description": "Guarda y organiza prompts como notas en categorías, con búsqueda y copia rápida.",
  "action": {
    "default_title": "Prompt Notes",
    "default_popup": "src/popup.html"
  },
  "options_page": "src/options.html",
  "background": {
    "service_worker": "src/background.js"
  },
  "permissions": ["storage", "contextMenus", "tabs"],
  "icons": {
    "16": "assets/icon16.png",
    "48": "assets/icon48.png",
    "128": "assets/icon128.png"
  }
}
```

---

## `src/storage.js`

```js
// Pequeña capa sobre chrome.storage para mantener un estado consistente
const DEFAULT_STATE = {
  lists: ["Inbox"],
  prompts: [] // {id, title, content, list, updatedAt}
};

function now() {
  return Date.now();
}

export async function getState() {
  return new Promise((resolve) => {
    chrome.storage.local.get(["state"], (res) => {
      const state = res.state || DEFAULT_STATE;
      // Asegurar que existen claves mínimas
      if (!state.lists || !Array.isArray(state.lists)) state.lists = ["Inbox"];
      if (!state.prompts || !Array.isArray(state.prompts)) state.prompts = [];
      resolve(state);
    });
  });
}

export async function setState(state) {
  return new Promise((resolve) => {
    chrome.storage.local.set({ state }, () => resolve());
  });
}

export async function addList(name) {
  const state = await getState();
  if (!state.lists.includes(name)) {
    state.lists.push(name);
    await setState(state);
  }
  return state.lists;
}

export async function removeList(name) {
  const state = await getState();
  if (name === "Inbox") return state; // Inbox no se elimina
  state.lists = state.lists.filter((l) => l !== name);
  state.prompts = state.prompts.map((p) => (p.list === name ? { ...p, list: "Inbox" } : p));
  await setState(state);
  return state;
}

export async function upsertPrompt({ id, title, content, list }) {
  const state = await getState();
  const idx = state.prompts.findIndex((p) => p.id === id);
  const prompt = {
    id: id || crypto.randomUUID(),
    title: title?.trim() || "Sin título",
    content: content || "",
    list: list || state.lists[0] || "Inbox",
    updatedAt: now()
  };
  if (idx >= 0) {
    state.prompts[idx] = { ...state.prompts[idx], ...prompt, id: state.prompts[idx].id, updatedAt: now() };
  } else {
    state.prompts.unshift(prompt); // primero de la lista
  }
  await setState(state);
  return prompt;
}

export async function deletePrompt(id) {
  const state = await getState();
  state.prompts = state.prompts.filter((p) => p.id !== id);
  await setState(state);
  return true;
}

export async function searchPrompts({ query = "", list = "ALL" } = {}) {
  const state = await getState();
  const q = query.toLowerCase();
  return state.prompts.filter((p) => {
    const inList = list === "ALL" || p.list === list;
    if (!inList) return false;
    if (!q) return true;
    return (
      p.title.toLowerCase().includes(q) ||
      p.content.toLowerCase().includes(q)
    );
  });
}

export async function renameList(oldName, newName) {
  const state = await getState();
  if (oldName === "Inbox") return state; // no renombrar Inbox
  if (!state.lists.includes(oldName)) return state;
  if (!newName || state.lists.includes(newName)) return state;
  state.lists = state.lists.map((l) => (l === oldName ? newName : l));
  state.prompts = state.prompts.map((p) => (p.list === oldName ? { ...p, list: newName } : p));
  await setState(state);
  return state;
}
```

---

## `src/popup.html`

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Prompt Notes</title>
  <style>
    body { font-family: system-ui, Arial, sans-serif; margin: 12px; width: 360px; }
    header { display: flex; gap: 8px; align-items: center; margin-bottom: 8px; }
    input, textarea, select, button { width: 100%; box-sizing: border-box; }
    .row { display: flex; gap: 8px; }
    .row > * { flex: 1; }
    .list { max-height: 260px; overflow: auto; border: 1px solid #ddd; border-radius: 8px; padding: 8px; }
    .item { padding: 6px; border-bottom: 1px solid #eee; }
    .item:last-child { border-bottom: none; }
    .title { font-weig
```
