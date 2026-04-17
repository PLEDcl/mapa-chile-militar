# mapa-chile-militar

Mapa interactivo de Chile con HUD militar / sci-fi, orientación horizontal, para pantalla ultra-wide 3584×768.

## Quick start

```bash
cd mapa-chile-militar-export
npx --yes http-server -p 4200 -c-1 .
```

Luego abre http://localhost:4200

## Usando Claude Code en este PC

**Antes de cualquier edición, lee `CONTEXTO.md` completo.** Contiene:
- Reglas de idioma (español neutro, NUNCA voseo)
- Decisiones de diseño y por qué están tomadas así
- Bugs históricos ya corregidos (no los reintroduzcas)
- Zonas sensibles del código
- Historial de iteraciones

## Contenido de la carpeta

```
mapa-chile-militar-export/
├── index.html              ← todo el proyecto (HTML + CSS + JS embebidos)
├── README.md               ← este archivo
├── CONTEXTO.md             ← contexto técnico completo para Claude
└── .claude/
    └── launch.json         ← config de preview server (puerto 4200)
```

## Stack

- MapLibre GL JS 4.7.1 (CDN)
- Google Fonts: Orbitron (CDN)
- Tiles: Esri / OpenTopoMap / CARTO (sin API key)

Sin dependencias locales. Sin `node_modules`. Sin build.
