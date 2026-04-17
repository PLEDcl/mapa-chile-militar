# Mapa de Chile — HUD militar horizontal ultra-wide

> Documento de contexto para Claude Code. Léelo completo antes de editar `index.html`.

---

## 0. Reglas de comunicación (NUNCA ignorar)

El usuario trabaja en Chile. **Responde SIEMPRE en español neutro/chileno, NUNCA en voseo argentino.**

- Usa **tú**, nunca "vos"
- Conjugaciones: **tienes / quieres / puedes / sabes / haces / eres / estás / dices** (no "tenés / querés / podés / sabés / hacés / sos")
- No uses modismos argentinos: "dale", "bárbaro", "laburo", "pibe", "che", "bancar", "al toque"
- Antes de enviar cada respuesta, escanea mentalmente: ¿alguna palabra termina en "-ás" (tenés/querés/podés)? → corrígela

Esta regla está también en el CLAUDE.md global del usuario y aplica a TODOS los proyectos.

---

## 1. Qué es el proyecto

Aplicación web **single-file** (`index.html`) que renderiza un mapa interactivo de Chile con estética **militar moderna / sci-fi** (tipo Star Wars / TACMAP).

- **Resolución objetivo:** pantalla ultra-wide **3584 × 768** (aspecto ~4.66:1). La UI debe aprovechar el largo horizontal.
- **Orientación:** Chile se muestra **horizontal** con `bearing=90°` — Arica a la izquierda, Magallanes a la derecha. Esto replica el estilo de los mapas IGM horizontales.
- **Nivel de detalle:** similar a cartografía IGM 1:1.000.000.
- **Público:** uso personal del usuario, sin requisitos de producción.

---

## 2. Stack técnico

- **Sin build, sin framework.** Todo es un único `index.html` con CSS y JS embebidos.
- **MapLibre GL JS v4.7.1** cargado por CDN (`https://unpkg.com/maplibre-gl@4.7.1/...`)
- **Fuente Orbitron** por Google Fonts
- **Tiles raster** desde Esri / OpenTopoMap / CARTO (no requieren API key)
- **Fallback local** de contorno de Chile (~237 puntos) embebido como `CHILE_OUTLINE` — se usa si falla la carga remota del GeoJSON Natural Earth.

**Para correr:** hay un `launch.json` en `.claude/` con configuración `mapa-chile-militar` que arranca `http-server` en puerto 4200. Desde esta carpeta:

```
npx --yes http-server -p 4200 -c-1 .
```

Claude Code puede usar la herramienta **`preview_start`** con el nombre `mapa-chile-militar`.

---

## 3. Features que están implementadas y funcionando

### Capas base (botones en barra CAPA)
- **SAT** — Esri World Imagery (satelital)
- **TOPO** — OpenTopoMap (topográfico)
- **TERR** — Esri World Terrain + labels
- **DARK** — CARTO dark
- **LIGHT** — CARTO light

### HUD militar
- Barra superior con coordenadas/bearing/pitch/zoom en tiempo real
- Barra inferior horizontal con las 16 regiones de Chile (click → abre popup)
- **Brújula** abajo-derecha que rota con el `bearing` (labels N/S/E/W y aguja)
- **Reglas en píxeles**: horizontal abajo, vertical derecha, muestran `window.innerWidth/Height`
- **Grid de radar** superpuesto + scanline + crosshair
- **Compass / pitch / bearing** sliders
- Botones: RESET, **FIT CL** (encuadra Chile Arica→extremo sur), ZOOM +/-
- Herramientas: MEDIR (distancia haversine en km), COORDS (copia al portapapeles), 3D (terreno raster-dem), LBL (toggle etiquetas)
- SYS: toggle DARK/LIGHT + FULL (fullscreen API)

### Panels flotantes
- Click en región (mapa o barra inferior) → panel Star Wars con capital, superficie, población, clima, geografía, economía, landmarks, ciudades
- Click en ciudad marker → panel con info de ciudad (capital/capital regional)
- ESC cierra el panel

### Datos
- Array `REGIONES` con las 16 regiones de Chile (num, id, nombre, capital, coord, superficie, población, clima, geografía, economía, landmarks, ciudades)

---

## 4. Decisiones clave y por qué (no las deshagas sin entenderlas)

### 4.1 Orientación horizontal — `bearing = 90°`
La primera versión mostraba Chile vertical. El usuario pidió que se viera horizontal para aprovechar el ultra-wide. Se usa `HORIZONTAL_BEARING = 90` en varios lugares (RESET view, FIT CL, init del mapa).

### 4.2 `fitChileHorizontal()` usa `map.fitBounds`
Antes calculaba zoom manualmente con m/px — daba mal encuadre (no se veía Arica o extremo sur). Ahora usa `map.fitBounds([[-76,-56.2],[-66.5,-17.45]], { bearing: 90, padding: ... })` que respeta la rotación automáticamente.

### 4.3 Click en región → proximidad, NO match por nombre de feature
**IMPORTANTE.** El GeoJSON remoto de Natural Earth a veces no carga bien y el fallback es **un solo polígono "Chile"**. El handler antiguo hacía:

```js
const name = (props.name || '').toLowerCase();
REGIONES.find(r => r.nombre.toLowerCase().includes(name))
```

Si `name === ''`, entonces `.includes('')` es SIEMPRE `true` → `find` devolvía REGIONES[0] = Arica. **Por eso todos los clicks abrían Arica.**

La solución actual (`pickRegionByProximity`) elige la región con `coord` más cercana al `e.lngLat` del click. Si alguna vez el GeoJSON remoto carga bien con 16 polígonos nombrados, `pickRegionByName` toma precedencia.

También se cambió de `map.on('click', 'regiones-fill', ...)` a `map.on('click', ...)` (global) porque el polígono fallback tiene huecos en la Patagonia y los clicks en Aysén/Magallanes no disparaban el handler delegado.

### 4.4 Brújula rota el `.compass-ring` COMPLETO, no solo la rosa
Antes solo la aguja SVG rotaba con `-bearing`, las letras N/S/E/W estaban fijas. Con `bearing=90°` la aguja apuntaba a la izquierda (correcto) pero la "N" seguía arriba (incorrecto). Ahora rota todo el anillo — las letras acompañan a la aguja como en una brújula de aviación real.

### 4.5 Capa `regiones-line` eliminada
El usuario pidió eliminar "la línea celeste que da el contorno a Chile". Se quitó la capa de `type: 'line'` con `line-color: '#00f0ff'` y `line-dasharray: [2, 1]`. Se conservó `regiones-fill` con `fill-opacity: 0` — invisible pero clickeable para disparar el panel de región.

### 4.6 Sin `maxBounds` ni `minZoom`
Se probó restringir la navegación a `CHILE_BOUNDS` pero clampeaba el zoom y no dejaba ver Chile entero con pitch. Se eliminó; la cámara es libre.

### 4.7 No hay máscara "solo Chile", ni vista ISO 3D, ni vista CARTO hipsométrica
Se añadieron en iteraciones intermedias y luego se eliminaron porque el usuario pidió "volvamos a la versión 1" (HUD militar horizontal puro). Si las necesitas reimplementar, el historial git tiene las versiones anteriores — ahí estaban las capas `iso-extrude`, `iso-shadow`, `iso-outline`, `mask-fill`, y el STYLE `carto` con OpenTopoMap + shaded relief.

### 4.8 Filtro sci-fi en modo dark
`[data-mode="dark"] #map .maplibregl-canvas { filter: contrast(1.08) saturate(0.85) brightness(0.92) hue-rotate(-8deg); }` — da el tinte cyan-frío característico del HUD militar. No lo quites a menos que el usuario lo pida.

### 4.9 `CHILE_OUTLINE` embebido (~237 puntos)
Es **fallback** del GeoJSON. No se usa para dibujar contorno (ya se eliminó la capa line). Solo sirve para que `buildMarkers` + `pickRegionByProximity` tengan una feature clickeable si falla la carga remota. No lo elimines.

---

## 5. Historial de iteraciones (resumen)

1. **v1 (base)** — HUD sci-fi militar con Chile vertical, SAT/TOPO/TERR/DARK/LIGHT, controles Google-Earth, reglas de píxeles, panels Star Wars.
2. **v2** — Rotación a `bearing=90°` (Chile horizontal).
3. **v3** — Menús reorganizados horizontales + añadidos: botón **SOLO CL** (máscara país), botón **CARTO** (estilo hipsométrico IGM).
4. **v4** — Añadido botón **ISO 3D** (Chile extruido arquitectónico sobre fondo blanco).
5. **v5** — Más detalle al outline de Chile; eliminado SOLO CL; mapa siempre mostraba solo Chile (máscara always-on + maxBounds).
6. **v6 (estado actual)** — "Volvamos a la versión 1" manteniendo horizontal. Eliminados CARTO, ISO 3D, máscara always-on, maxBounds.
7. **Fixes posteriores:**
   - Eliminada línea celeste del contorno
   - `FIT CL` reescrito con `fitBounds` (muestra Arica → extremo sur)
   - Click en región corregido (ya no abre siempre Arica)
   - Brújula rota completa con el bearing

---

## 6. Cómo abordar cambios típicos

### Si el usuario pide cambiar algo visual
1. Lee la zona relevante de `index.html` (el archivo es ~1800 líneas, úsalo con `offset`/`limit` en Read).
2. Identifica si el cambio va en CSS (líneas ~10-600 aprox) o JS (desde ~1170 hasta el final).
3. Después de cada edición, **recarga el preview con cache-busting**: `location.href = location.pathname + '?v=' + Date.now()`.
4. Verifica con `preview_screenshot` o `preview_eval`.

### Si el usuario reporta un bug
1. Usa `preview_console_logs` primero para ver errores.
2. Usa `preview_eval` para inspeccionar estado (`map.getBearing()`, `map.getZoom()`, `document.querySelector(...)`, etc.).
3. Antes de "arreglar" algo, reproduce el bug simulando el evento (ver ejemplo del click handler donde se simularon 12 clicks por `dispatchEvent`).

### Medidas de la pantalla objetivo
- `window.innerWidth` = 3584, `window.innerHeight` = 768 en la pantalla final del usuario
- El preview MCP usa 1792×384 (escala 0.5) para testing — los rulers muestran esas dimensiones dinámicamente, no hardcodeadas

### Puertos / comandos
- Preview server: `http-server -p 4200 -c-1 .` (cache-off, relativo a la carpeta del proyecto)
- Si `http-server` no está instalado: `npx --yes` lo baja automáticamente (ver `launch.json`)

---

## 7. Zonas sensibles del código (ojo al editar)

| Zona | Líneas aprox | Qué tiene |
|---|---|---|
| Variables CSS + filtros | 1-100 | `--hud-accent`, filtros sci-fi, rulers |
| Compass CSS | ~540-580 | `.compass-ring` (rota con bearing), labels N/S/E/W |
| HUD HTML | ~700-900 | Barra superior, panels, brújula, botones |
| `REGIONES` array | ~950-1150 | 16 objetos región con todo el contenido |
| `STYLES` (map styles) | ~1160-1230 | SAT/TOPO/TERR/DARK/LIGHT |
| `CHILE_OUTLINE` | ~1280-1335 | Fallback coords |
| `loadRegionGeoJSON` | ~1336-1366 | Carga remota + fallback |
| `addRegionLayer` | ~1368-1382 | Solo `regiones-fill` (invisible, clickeable) |
| `showRegionInfo` / `showCityInfo` | ~1398-1450 | Poblado de panels |
| Layer switcher | ~1500-1525 | Click en botones CAPA |
| `fitChileHorizontal` | ~1530-1545 | `fitBounds` con bbox de Chile |
| Click handler global | ~1775-1810 | `pickRegionByName` + `pickRegionByProximity` |
| `updateHUD` | ~1480-1495 | HUD + rotación brújula |

---

## 8. Checklist antes de dar por cerrada una tarea

- [ ] El código recargado en el preview refleja el cambio (no hay caché stale)
- [ ] No hay errores en `preview_console_logs`
- [ ] El cambio es coherente con el resto del HUD militar (no introduce elementos fuera de estilo)
- [ ] Si tocaste el layer switcher, probaste al menos SAT + DARK
- [ ] Si tocaste el click handler, simulaste 3+ clicks en distintas latitudes

---

## 9. Cosas que el usuario ha pedido NO hacer

- No usar voseo argentino (regla #0, inmutable)
- No inflar el archivo con features que no pidió
- No revertir a la brújula con labels fijas
- No reponer el contorno celeste punteado de las regiones
- No asumir que cada capa se ve igual en dark y light — probar ambas si aplica

---

## 10. Perfil del usuario

- Trabaja en PLED (empresa chilena), rol técnico enfocado en HubSpot
- Hardware potente (RTX 5080) — puedes pedirle que corra lo que sea localmente
- Sabe programar pero no es dev senior — prefiere soluciones simples y probadas sobre enfoques creativos
- Pide que el código funcione a la primera: antes de entregar, verifica edge cases mentalmente, revisa documentación si la solución no te parece sólida
