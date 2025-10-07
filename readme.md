A continuación tienes un proyecto funcional **Manifest V3** con todo lo necesario para guardar, buscar, copiar y categorizar prompts. Copia esta estructura en carpetas/archivos tal cual.

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
    .title { font-weight: 600; }
    .muted { color: #666; font-size: 12px; }
    .btn { padding: 8px; border: 1px solid #ccc; border-radius: 8px; cursor: pointer; background: #fff; }
    .btn.primary { background: #111; color: #fff; border-color: #111; }
    .btn.danger { background: #b00020; color: #fff; border-color: #b00020; }
    .gap { height: 8px; }
    details summary { cursor: pointer; }
  </style>
</head>
<body>
  <header>
    <input id="q" type="search" placeholder="Buscar… (título o contenido)" />
    <select id="filterList"></select>
  </header>

  <section class="list" id="results"></section>

  <div class="gap"></div>

  <details>
    <summary>Nuevo / Editar prompt</summary>
    <div class="gap"></div>
    <div class="row">
      <input id="title" placeholder="Título" />
      <select id="list"></select>
    </div>
    <div class="gap"></div>
    <textarea id="content" rows="5" placeholder="Contenido del prompt…"></textarea>
    <div class="gap"></div>
    <div class="row">
      <button class="btn" id="clear">Limpiar</button>
      <button class="btn primary" id="save">Guardar</button>
    </div>
  </details>

  <script type="module" src="popup.js"></script>
</body>
</html>
```

---

## `src/popup.js`

```js
import { getState, addList, searchPrompts, upsertPrompt, deletePrompt } from "./storage.js";

const $ = (sel) => document.querySelector(sel);
const results = $("#results");
const q = $("#q");
const filterList = $("#filterList");
const listSelect = $("#list");
const title = $("#title");
const content = $("#content");
const saveBtn = $("#save");
const clearBtn = $("#clear");

let editingId = null; // si existe, estamos editando

async function loadLists() {
  const state = await getState();
  const lists = state.lists || ["Inbox"];
  filterList.innerHTML = `<option value="ALL">Todas</option>` +
    lists.map((l) => `<option value="${l}">${l}</option>`).join("");
  listSelect.innerHTML = lists.map((l) => `<option value="${l}">${l}</option>`).join("");
}

function renderItem(p) {
  const date = new Date(p.updatedAt).toLocaleString();
  const el = document.createElement("div");
  el.className = "item";
  el.innerHTML = `
    <div class="title">${escapeHtml(p.title)}</div>
    <div class="muted">Lista: ${escapeHtml(p.list)} · Actualizado: ${date}</div>
    <div class="muted">${escapeHtml(p.content).slice(0, 180)}${p.content.length > 180 ? "…" : ""}</div>
    <div class="row" style="margin-top:6px">
      <button class="btn" data-act="copy" data-id="${p.id}">Copiar</button>
      <button class="btn" data-act="edit" data-id="${p.id}">Editar</button>
      <button class="btn danger" data-act="del" data-id="${p.id}">Eliminar</button>
    </div>
  `;
  return el;
}

function escapeHtml(str) {
  return str.replace(/[&<>"']/g, (m) => ({
    "&": "&amp;",
    "<": "&lt;",
    ">": "&gt;",
    '"': "&quot;",
    "'": "&#39;"
  })[m]);
}

async function refresh() {
  const items = await searchPrompts({ query: q.value, list: filterList.value });
  results.innerHTML = "";
  items.forEach((p) => results.appendChild(renderItem(p)));
}

async function fillEdit(p) {
  editingId = p.id;
  title.value = p.title;
  content.value = p.content;
  await ensureListExists(p.list);
  listSelect.value = p.list;
  document.querySelector("details").open = true;
}

async function ensureListExists(name) {
  const state = await getState();
  if (!state.lists.includes(name)) {
    await addList(name);
    await loadLists();
  }
}

async function onResultClick(e) {
  const btn = e.target.closest("button[data-act]");
  if (!btn) return;
  const id = btn.dataset.id;
  const act = btn.dataset.act;
  const state = await getState();
  const p = state.prompts.find((x) => x.id === id);
  if (!p) return;

  if (act === "copy") {
    await navigator.clipboard.writeText(p.content);
    btn.textContent = "Copiado";
    setTimeout(() => (btn.textContent = "Copiar"), 1200);
  }
  if (act === "edit") {
    fillEdit(p);
  }
  if (act === "del") {
    await deletePrompt(id);
    await refresh();
  }
}

async function onSave() {
  const prompt = await upsertPrompt({ id: editingId, title: title.value, content: content.value, list: listSelect.value });
  editingId = prompt.id;
  await refresh();
}

function onClear() {
  editingId = null;
  title.value = "";
  content.value = "";
}

// Eventos
results.addEventListener("click", onResultClick);
q.addEventListener("input", refresh);
filterList.addEventListener("change", refresh);
saveBtn.addEventListener("click", onSave);
clearBtn.addEventListener("click", onClear);

// Init
(async function init() {
  await loadLists();
  await refresh();
})();
```

---

## `src/options.html`

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Prompt Notes — Opciones</title>
  <style>
    body { font-family: system-ui, Arial, sans-serif; margin: 20px; max-width: 800px; }
    .row { display: flex; gap: 8px; margin-bottom: 12px; }
    input, button { padding: 8px; }
    .btn { border: 1px solid #ccc; border-radius: 8px; cursor: pointer; background: #fff; }
    .btn.primary { background: #111; color: #fff; border-color: #111; }
    table { width: 100%; border-collapse: collapse; }
    td, th { padding: 8px; border-bottom: 1px solid #eee; text-align: left; }
  </style>
</head>
<body>
  <h1>Listas / Categorías</h1>

  <div class="row">
    <input id="newList" placeholder="Nombre de la nueva lista" />
    <button class="btn primary" id="add">Añadir lista</button>
  </div>

  <table>
    <thead>
      <tr><th>Lista</th><th>Renombrar a…</th><th>Acciones</th></tr>
    </thead>
    <tbody id="tbody"></tbody>
  </table>

  <script type="module" src="options.js"></script>
</body>
</html>
```

---

## `src/options.js`

```js
import { getState, addList, removeList, renameList } from "./storage.js";

const $ = (s) => document.querySelector(s);
const tbody = $("#tbody");
const newList = $("#newList");
const addBtn = $("#add");

function rowTemplate(name) {
  const tr = document.createElement("tr");
  tr.innerHTML = `
    <td>${name}</td>
    <td><input data-role="rename" data-name="${name}" placeholder="Nuevo nombre" /></td>
    <td>
      <button data-role="do-rename" data-name="${name}">Renombrar</button>
      <button data-role="delete" data-name="${name}">Eliminar</button>
    </td>
  `;
  return tr;
}

async function refresh() {
  const state = await getState();
  tbody.innerHTML = "";
  state.lists.forEach((l) => tbody.appendChild(rowTemplate(l)));
}

async function onClick(e) {
  const btn = e.target.closest("button[data-role]");
  if (!btn) return;
  const role = btn.dataset.role;
  const name = btn.dataset.name;

  if (role === "delete") {
    if (name === "Inbox") return alert("Inbox no se puede eliminar");
    await removeList(name);
    await refresh();
  }
  if (role === "do-rename") {
    const input = tbody.querySelector(`input[data-role="rename"][data-name="${CSS.escape(name)}"]`);
    const to = input?.value?.trim();
    if (!to) return alert("Escribe el nuevo nombre");
    await renameList(name, to);
    await refresh();
  }
}

async function onAdd() {
  const name = newList.value.trim();
  if (!name) return;
  await addList(name);
  newList.value = "";
  await refresh();
}

tbody.addEventListener("click", onClick);
addBtn.addEventListener("click", onAdd);

refresh();
```

---

## `src/background.js`

```js
// Inicializa estado por primera vez y crea menú contextual para guardar selección
importScripts("./storage.js");

chrome.runtime.onInstalled.addListener(async () => {
  // Garantizar estado inicial
  chrome.storage.local.get(["state"], (res) => {
    if (!res.state) {
      chrome.storage.local.set({ state: { lists: ["Inbox"], prompts: [] } });
    }
  });

  // Crear menú contextual si hay permiso
  try {
    chrome.contextMenus?.create({
      id: "prompt-notes-save-selection",
      title: "Guardar selección como prompt (Inbox)",
      contexts: ["selection"]
    });
  } catch (e) {
    // Nada: en MV3 si ya existe puede lanzar error.
  }
});

chrome.contextMenus?.onClicked.addListener(async (info, tab) => {
  if (info.menuItemId !== "prompt-notes-save-selection") return;
  const content = (info.selectionText || "").trim();
  if (!content) return;
  const title = content.split("\n")[0].slice(0, 60) || "Selección";

  // Cargar módulo dinámicamente porque importScripts no exporta símbolos a este scope
  const modUrl = chrome.runtime.getURL("src/storage.js");
  const storage = await import(modUrl);
  await storage.upsertPrompt({ title, content, list: "Inbox" });
});
```

---

## `assets/icon16.png`, `assets/icon48.png`, `assets/icon128.png`

Iconos cualesquiera (puedes usar temporales).

---

### Notas rápidas

* Carga la carpeta con **Load unpacked** en `chrome://extensions`.
* Si no quieres menú contextual, quita `contextMenus` del `manifest` y borra su código en `background.js`.
* Este código usa módulos ES en popup/options y un `service_worker` MV3.
