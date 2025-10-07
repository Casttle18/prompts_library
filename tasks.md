# Listado de tareas

## Estructura general
- [ ] Crear la estructura de carpetas `src/` y `assets/` indicada en el README.
- [ ] Añadir los iconos requeridos en `assets/icon16.png`, `assets/icon48.png` y `assets/icon128.png` (pueden ser temporales).
- [ ] Configurar la extensión en modo "Load unpacked" desde `chrome://extensions` una vez generados los archivos.

## Archivos principales
- [ ] Implementar `manifest.json` con la configuración de Manifest V3 proporcionada.
- [ ] Crear `src/storage.js` con las funciones para gestionar el estado en `chrome.storage` (`getState`, `setState`, `addList`, `removeList`, `upsertPrompt`, `deletePrompt`, `searchPrompts`, `renameList`).
- [ ] Crear `src/popup.html`, `src/popup.js` y `src/popup.css` con la interfaz de administración de prompts siguiendo las estructuras descritas.
- [ ] Implementar `src/background.js` como *service worker* que inicializa el estado y gestiona el menú contextual para guardar selecciones.
- [ ] Crear `src/options.html` y `src/options.js` para administrar listas de categorías, con los manejadores de eventos y plantillas de filas indicados.

## Funcionalidades del popup
- [ ] Diseñar el HTML del popup con cabecera, buscador, selector de listas y listado de prompts.
- [ ] Implementar en `src/popup.js` la lógica de renderizado de prompts, búsqueda, filtrado por listas y acciones de copiar/editar/eliminar.
- [ ] Añadir estilos base en `src/popup.css` para la interfaz del popup, incluyendo responsividad y estados vacíos.

## Funcionalidades de las opciones
- [ ] Implementar en `src/options.js` las funciones `refresh`, `onClick`, `onAdd` y el comportamiento descrito para renombrar y eliminar listas (evitando eliminar "Inbox").
- [ ] Definir la tabla y formularios en `src/options.html` con los identificadores usados por `options.js`.

## Flujo de trabajo adicional
- [ ] Garantizar que los prompts se guardan con `updatedAt` y que las operaciones devuelven el estado actualizado.
- [ ] Manejar la creación de IDs con `crypto.randomUUID()` cuando no se proporcione.
- [ ] Asegurar que el menú contextual llama a `upsertPrompt` con el contenido seleccionado.

