# MISPAGINAS — Custodia del Proyecto

> Última actualización: 2026-10-12 · Versión del documento: v1

## Mapa del documento

Lectura por defecto: `7. MEMORIA` y este mapa. Las secciones `1`–`5` se abren **solo** cuando la tarea las mencione (`4. CAPACIDADES` y `5. LECCIONES` son de consulta).

- listado, tarjetas, indice de sitios, repos, api.github.com → `4. CAPACIDADES`
- fallback, copia local, si la API falla → `4. CAPACIDADES`
- filtro, buscar, reducir tarjetas en vivo → `4. CAPACIDADES`
- fetch, cors, sin backend, build_type legacy → `2. ARQUITECTURA`
- que es el proyecto, stack, rutas, repo, URL de Pages → `1. IDENTIDAD`
- por que algo se hizo asi, gotchas, errores del pasado → `5. LECCIONES`
- como validar, desplegar, ver logs → `6. SKILLS`
- estado actual, decisiones, tarea activa → `7. MEMORIA`
- historial de cambios → `8. VERSIONES` (solo visibles; `8b` nunca)
- entradas antiguas → `5b` / `8b`

## 1. IDENTIDAD

- Qué es el proyecto: índice web de los sitios publicados en GitHub Pages del usuario. Lee los repositorios públicos con Pages activo y los muestra como tarjetas con enlace directo. No es un sitio de contenido propio, sino un panel de navegación hacia los demás.
- Stack: HTML/CSS/JavaScript en un único archivo (`index.html`, 275 líneas, 7.829 bytes). Sin build, sin framework, sin gestor de dependencias, sin CDN.
- Rutas principales: carpeta del proyecto `Mis Paginas`; dentro, `index.html` (todo el proyecto) y `MISPAGINAS-Custodia.md` (este documento).
- Repo git y estado: `https://github.com/SandovalJon/mis-paginas.git`, rama `master`, un solo commit. `git status` limpio tras el arranque.
- URLs: producción `https://sandovaljon.github.io/mis-paginas/` (HTTP 200, 7.829 bytes servidos, idéntico al archivo local). Repositorio: `https://github.com/SandovalJon/mis-paginas`.
- Versión del código: `64b6d98` ("Indice de sitios publicados en GitHub Pages").
- Fecha de la última versión del código: 2026-10-12.

## 2. ARQUITECTURA

- Componentes: un documento HTML autocontenido. La lista se obtiene en el cliente con `fetch` contra la API pública de GitHub; no hay backend ni build.
- Cómo se conectan: el navegador pide `https://api.github.com/users/SandovalJon/repos?per_page=100&sort=updated` y filtra los repositorios con `has_pages === true`. La API pública responde con `Access-Control-Allow-Origin: *`, por lo que no hace falta token ni servidor propio. Verificado: HTTP 200, CORS `*`, campo `has_pages` presente.
- Entry points: `index.html` servido estáticamente por GitHub Pages.
- Flujos clave:
  - Carga: `load()` pide la API → filtra por `has_pages` → ordena por nombre → `render()`. Si la petición falla, usa la constante `FALLBACK` y lo indica en pantalla con la clase `err`.
  - Enlace de cada sitio: `siteUrl()` construye `https://<usuario>.github.io/<repo>/`. Es correcto mientras el repo no tenga dominio propio: ninguno de los repos actuales tiene `cname` configurado, comprobado en la API de Pages.
  - Filtro: evento `input` sobre `#filter`, filtra por nombre y descripción sin volver a pedir datos.
- Despliegue (comandos exactos): `git add index.html` · `git commit -m "<mensaje>"` · `git push` · `gh api repos/SandovalJon/mis-paginas/pages --jq '.status'` hasta `built` · comprobar con `Invoke-WebRequest -Uri "https://sandovaljon.github.io/mis-paginas/" -UseBasicParsing` y comparar `$r.RawContentLength` con el tamaño de `index.html`.
- Operación (comandos exactos): listar los sitios con `gh api "user/repos?per_page=100&affiliation=owner" --paginate --jq '.[] | select(.has_pages==true) | .name'`. Estado del sitio con `gh api repos/SandovalJon/mis-paginas/pages --jq '{url: .html_url, status: .status}'`.

## 3. RECURSOS

- APIs que usa: API pública de GitHub REST v3 en modo anónimo, endpoint `GET /users/{usuario}/repos`. Sin autenticación, por lo que el límite es de 60 peticiones por hora y por IP. La copia local `FALLBACK` cubre ese caso.
- Dónde vive la configuración: el usuario está fijado en la constante `USERNAME` dentro del `<script>` de `index.html`. No hay archivo de configuración ni variables de entorno. No hay secretos: la llamada es anónimo y no se envía ningún token.
- Esquemas de datos: no hay persistencia. Forma del objeto que consume la lista: `{ name, description, has_pages, updated_at }`, tomado directamente de la respuesta de la API. La copia local `FALLBACK` mantiene solo `name`, `description` y `updated_at`.
- Memoria / almacenamiento: ninguno. No se usa `localStorage` ni cookies; el filtro es estado volátil de la página.

## 4. CAPACIDADES

- Listado en vivo de los sitios con Pages — filtra la API por `has_pages` y muestra 3 tarjetas ordenadas por nombre. Verificado por: ejecución en jsdom con `fetch` real, salida `3 sitios · datos en vivo`.
- Respaldo con copia local — si la API falla, se muestran los mismos sitios desde la constante `FALLBACK` y se avisa en pantalla. Verificado por: ejecución en jsdom con `fetch` rechazado, salida `No se pudo contactar con la API (red bloqueada). Se muestra la copia local.` y 3 tarjetas.
- Filtro por nombre y descripción — reduce las tarjetas en vivo sin peticiones de red. Verificado por: filtro `ruleta` deja 1 tarjeta; filtro `noexiste` deja 0.
- Indicador de antigüedad — etiqueta `hoy`, `hace N d`, y marca en color de aviso cuando el repo lleva más de 180 días sin actualizarse. Verificado por: salida `send-anywhere-p2p | hoy` frente a `hace 7 d` en los otros dos.
- Enlace al repositorio y actualización manual — botón "Ver repos" y botón "↻ Actualizar". Verificado por: `href` resuelto a `https://github.com/SandovalJon` en el marcado; el efecto del botón de actualizar no está verificado en producción.

## 5. LECCIONES

1. En la API GraphQL de GitHub el tipo `Repository` no tiene el campo `hasPages` — Causa raíz: la consulta GraphQL falló con `Field 'hasPages' doesn't exist on type 'Repository'`; el nombre correcto solo existe en REST. · Fix: usar REST (`user/repos`), donde el campo sí se llama `has_pages`. · Verificación: petición REST con HTTP 200 devolviendo los 3 repositorios esperados.
2. `gh api graphql -f query=...` no acepta literales en `login(...)` — Causa raíz: PowerShell y el parser de GraphQL chocan al pasar `user(login: "X")` dentro de `-f`, produciendo `argumentLiteralsIncompatible`. · Fix: pasar los argumentos como variables GraphQL (`-F owner=...`) o usar REST. · Verificación: consulta sustituida por REST, con resultado correcto.
3. El `--jq` de `gh` se rompe con el operador de coalescencia y una cadena que empieza por guion — Causa raíz: `(.homepage // "-")` provoca `unexpected token ")"` porque el guion se interpreta como opción. · Fix: evitar literales que empiecen por `-` dentro de `--jq`, o usar el equivalente en PowerShell. · Verificación: la consulta se rehízo sin ese literal y devolvió las tres filas.
4. `gh` escribe el progreso del push en stderr, no en stdout — Causa raíz: `gh repo create --push` imprime la URL del remoto y el `* [new branch]` por stderr, lo que PowerShell reporta como error aunque el push funcione. · Fix: no confiar en la ausencia de error en stderr; comprobar con `gh repo view`. · Verificación: `gh repo view` devolvió el repo creado y la rama correctamente.
5. jsdom no implementa `fetch`, así que la página no puede probarse sin inyectarlo — Causa raíz: sin `fetch` la carga falla y solo se ejercita el camino de respaldo, dando un falso positivo. · Fix: inyectar el `fetch` de Node en `beforeParse` para probar el camino real. · Verificación: la prueba con `fetch` real mostró `datos en vivo`, y la de `fetch` rechazado mostró la copia local: son los dos caminos probados por separado.
6. La API pública de GitHub tiene un límite de 60 peticiones por hora y por IP — Causa raíz: sin token, la cuota es compartida por IP; un campus o una NAT con muchos usuarios puede agotarla. · Fix: copia local embebida como red de seguridad y aviso explícito en pantalla cuando se usa. · Verificación: camino de respaldo ejecutado con `fetch` rechazado.
7. La URL de un sitio Pages solo es predecible si el repo no tiene dominio propio — Causa raíz: `https://<user>.github.io/<repo>/` se deduce del nombre, pero un `cname` configurado rompe esa regla. · Fix: la regla sirve mientras ningún repo tenga dominio propio, condición verificada contra la API de Pages. · Verificación: los 3 repos devuelven `cname: ninguno`.

## 5b. LECCIONES (archivo)

Sin entradas archivadas todavía.

## 6. SKILLS

- Skill: publicar en GitHub Pages con la CLI de GitHub — Estado: verificada
  - Propósito: subir el cambio y confirmar que el sitio sirve exactamente el código local.
  - Precondiciones: `gh` autenticado con scope `repo`; Pages habilitado en el repo.
  - Pasos: `git add index.html` · `git commit -m "<mensaje>"` · `git push` · `gh api repos/SandovalJon/<repo>/pages --jq '.status'` en bucle hasta `built` · `Invoke-WebRequest -Uri "<url>" -UseBasicParsing` y comparar `$r.RawContentLength` con el tamaño local.
  - Verificación: HTTP 200 y tamaño servido idéntico al local (7.829 bytes).
  - Agente: [opencode-1]

- Skill: enumerar los repos con Pages de un usuario — Estado: verificada
  - Propósito: obtener la lista sin abrir la web de GitHub.
  - Pasos: `gh api "user/repos?per_page=100&affiliation=owner" --paginate --jq '.[] | select(.has_pages==true) | .name'` y, para la URL autoritativa de cada uno, `gh api repos/SandovalJon/<repo>/pages --jq '{url: .html_url, cname: .cname}'`.
  - Verificación: devuelve `Metadata-WAV-Studio-by-SandovalJon`, `Ruleta-by-SandovalJon` y `send-anywhere-p2p`, los tres con `cname` vacío.
  - Agente: [opencode-1]

- Skill: probar una página que consume una API con jsdom — Estado: verificada
  - Propósito: ejercitar el camino real y el de respaldo por separado.
  - Precondiciones: `npm install --no-save jsdom`.
  - Pasos: cargar el HTML con `runScripts:'dangerously'` · en `beforeParse` asignar `window.fetch` a una implementación real para el camino feliz · repetir con `window.fetch = () => Promise.reject(new Error(...))` para el respaldo · capturar `error` y `unhandledrejection` para detectar fallos silenciosos · comprobar el texto de `#status` y el número de `.card`.
  - Verificación: ambos escenarios devolvieron 3 tarjetas con mensajes distintos.
  - Agente: [opencode-1]

## 7. MEMORIA

- [opencode-1] Tarea activa: ninguna. Arranque Ruta A ejecutado el 2026-10-12 con la creación de este documento, tras publicar la v1.
- [opencode-1] Decisión: proyecto independiente con repo `mis-paginas` y no una página dentro de `send-anywhere-p2p` — motivo: el índice no pertenece a la app P2P y su `index.html` es el sitio publicado; mezclar los dos Would acoplar ambos despliegues — estado: tomada (confirmada por el usuario).
- [opencode-1] Decisión: listar en vivo desde la API pública con copia local de respaldo — motivo: se actualiza solo al publicar sitios nuevos sin mantenimiento, a cambio de depender de la cuota anónima de 60 peticiones por hora e IP — estado: tomada.
- [opencode-1] Decisión: `NOMBRE` es `MISPAGINAS` — motivo: se aplica la convención del prompt al nombre de la carpeta y al repo, unidos en mayúsculas y sin espacios; el repo es `mis-paginas` y la carpeta es `Mis Paginas` — estado: tomada.
- [opencode-1] Cambio pedido por el usuario el 2026-10-12: la carpeta pasó de `MISPAGINAS` a `Mis Paginas`, con mayúscula solo en la primera letra de cada palabra, para igualarse al estilo de las carpetas hermanas (`Ruleta by SandovalJon`, `Bases de datos`). El nombre del archivo de custodia no cambia porque `MISPAGINAS` sigue siendo la unión en mayúsculas de la carpeta. El renombrado es solo local: no afecta al repo ni al sitio publicado.
- [opencode-1] Decisión: no incluir repos privados ni sin Pages — motivo: el objetivo son los sitios visitables sin sesión — estado: tomada.
- Preferencias del usuario: español; respuestas concisas; quiere que el agente ejecute y publique sin pedir confirmación paso a paso. Acepta la estructura de custodio y delega decisiones estructurales cuando se le presentan con opciones.
- Comprometidos: repo `mis-paginas` con Pages activo en `master`. Este proyecto es independiente de `SENDAWHEREP2P` y tiene su propio archivo de custodia.
- Ideas (no comprometidos): permitir elegir el usuario o escanear otros perfiles; marcar los sitios que devuelvan error al abrirse; exportar la lista a un archivo de marcadores.

## 8. VERSIONES

- v1 | 2026-10-12 | Publicación del índice de páginas | código: 64b6d98
  - Agente: [opencode-1]
  - Qué cambió: se creó `index.html` con listado en vivo de los repositorios con GitHub Pages, filtro por nombre y descripción, indicador de antigüedad y copia local de respaldo. Se creó el repo `mis-paginas`, se publicaron los archivos y se activó GitHub Pages. No hay código previo en este proyecto.
  - Secciones actualizadas: todas (creación). `6. SKILLS` registra la publicación en Pages, la enumeración de repos con Pages y la prueba de una página que consume una API con jsdom.
  - Verificación: con `fetch` real la página muestra `3 sitios · datos en vivo` y las 3 tarjetas esperadas; con `fetch` rechazado muestra la copia local y lo indica; el filtro reduce a 1 con `ruleta` y a 0 con `noexiste`. El sitio responde HTTP 200 con 7.829 bytes, idénticos a `index.html` local.
  - Pendientes: comprobar en producción el botón "↻ Actualizar"; validar el comportamiento cuando se añada un repo con dominio propio, que rompería la construcción de la URL.

## 8b. VERSIONES (archivo)

Sin entradas archivadas todavía.