# Design Spec: Nature Visual — Organismo Alieno (Approccio 1 + Iniezioni dal 2)

**Data:** 2026-06-12
**Branch:** nature-visual
**Target:** `index.html` (sostituisce la versione attuale)
**Scopo:** Creare un visualizer 3D audio-reattivo con estetica "organismo alieno/malato" — caos fluido, forme organiche senza simmetria forzata, particelle con vita propria.

---

## 1. Visione Visiva Finale

Una **sfera piccola e densa** (raggio 0.7) al centro di uno spazio buio profondo (#000000). La superficie della sfera è viva: gonfiori morbidi si alternano a spigoli e escrescenze, come un organismo malato che respira a fatica. Attorno, **3000 particelle** formano tentacoli, code e detriti che si muovono con **vita propria** — ognuna ha velocità, direzione e soglia di accensione individuali. Quando arriva un kick, l'organismo si **contorce violentemente**, le particelle **esplodono** verso l'esterno in un flash bianco sporco, e il colore della palette cambia istantaneamente (mutazione). L'effetto complessivo è un **caos organico fluido**: mai simmetrico, mai prevedibile, sempre in mutazione.

---

## 2. Componenti della Scena

### 2.1 Sfera Centrale (Mesh + ShaderMaterial)
- **Geometria:** `SphereGeometry(0.7, 256, 256)` — piccola ma super densa per superficie dettagliata
- **Vertex Shader:** Simplex Noise 3D a 2 layer:
  - **Layer 1 (bassa frequenza):** `snoise(pos * 2.0 + time * 0.3)` — onde morbide, gonfiori lenti
  - **Layer 2 (alta frequenza):** `snoise(pos * 7.0 + time * 0.6) * 0.3` — detriti, spigoli, escrescenze
  - Displacement: `normal * (lowFreq + highFreq) * 0.4 * (1.0 + uBass * 2.0)` — i bassi fanno contorcere l'organismo
  - **Normale finta:** calcolata dal gradiente del noise per shading realistico
- **Fragment Shader:**
  - Colore dalla palette acida basato su `vPosition` + `uTime` — **NO simmetria forzata**
  - Fresnel glow sui bordi (colore complementare alla palette)
  - Shading calcolato sulla normale finta (luce da sinistra-alto)
  - Flash bianco sporco su beat forti (`uBass > 0.65`)

### 2.2 Particelle Orbitanti (Points + BufferGeometry)
- **Geometria:** BufferGeometry con 3000 vertici
- **Attributi custom:**
  - `aPhase` (float): offset animazione 0-1
  - `aSpeed` (float): velocità individuale 0.2-2.0
  - `aThreshold` (float): soglia per accensione/spegnimento 0.1-0.9
- **Vertex Shader:**
  - Movimento **non orbitale** — traiettoria sinusoidale indipendente per ogni particella
  - Espansione radiale su beat forti: `radius *= (1.0 + smoothstep(0.5, 0.9, uBass) * 3.0)`
  - Flash bianco sporco quando esplodono
- **Fragment Shader:**
  - Size piccolissimo (0.2px base) ma glow intenso via `smoothstep`
  - Colore dalla palette acida

### 2.3 Audio & Trigger
- **FFT:** 2048 bin
- **Bassi (0-150Hz):** Contorsione sfera (displacement), espansione particelle
- **Medi (150-2500Hz):** Trigger mutazione — cambio istantaneo del `uSeed` del noise + shift colore palette
- **Alti (2500-8000Hz):** Frequenza lampeggio particelle, intensità glow
- **Trigger Mutazione:** Quando `uMid > 0.55`, cambia `uSeed += random(5.0, 13.0)` con cooldown 0.2s

### 2.4 Post-processing (Opzionale)
- Feedback blur leggero per scie sottili: fade 0.96, zoom 0.999, blur minimo
- Implementato come pass separato su render target

### 2.5 Camera e Renderer
- `PerspectiveCamera(fov=60, near=0.1, far=100)` a z=4
- `WebGLRenderer` antialias, sfondo nero #000000
- Rotazione lenta automatica scena: `scene.rotation.y += 0.005`

---

## 3. Shader GLSL — Dettaglio

### Vertex Shader Sfera (`vsSphere`)
1. Calcola `lowFreq = snoise(position * 2.0 + vec3(uTime * 0.3, uTime * 0.25, uTime * 0.35) + uSeed)`
2. Calcola `highFreq = snoise(position * 7.0 + vec3(uTime * 0.6, uTime * 0.5, uTime * 0.7) + uSeed * 2.0) * 0.3`
3. Displacement: `float disp = (lowFreq + highFreq) * 0.4 * (1.0 + uBass * 2.0);`
4. `vec3 newPos = position + normal * disp;`
5. Calcola normale finta tramite gradiente implicito del noise (6 campioni offset)
6. Passa `vPos`, `vNorm`, `vViewDir` al fragment

### Fragment Shader Sfera (`fsSphere`)
1. Colore base: `palette(vPos.x + vPos.y + vPos.z + uTime * 0.15, floor(uTheme))`
2. Shading: `dot(vNorm, lightDir)` con luce da (0.5, 0.8, 0.3)
3. Fresnel: `pow(1.0 - dot(vNorm, vViewDir), 3.0)`
4. Flash bianco: `if(uBass > 0.65) col += vec3(smoothstep(0.65, 0.9, uBass) * 0.6);`

### Vertex Shader Particelle (`vsParticles`)
1. `vec3 basePos = position;` (posizione iniziale randomica)
2. Movimento: `basePos += vec3(sin(uTime * aSpeed + aPhase * 10.0) * 0.3, cos(uTime * aSpeed * 0.7 + aPhase * 15.0) * 0.2, sin(uTime * aSpeed * 0.5 + aPhase * 8.0) * 0.25);`
3. Espansione radiale: `float explosion = smoothstep(0.5, 0.9, uBass) * 3.0;`
4. `vec3 finalPos = basePos * (1.0 + explosion);`
5. Flash: `float flash = smoothstep(0.6, 0.85, uBass);`
6. Size: `gl_PointSize = (0.2 + flash * 2.0) * (300.0 / gl_Position.z);`

### Fragment Shader Particelle (`fsParticles`)
1. Glow intenso: `float alpha = smoothstep(0.5, 0.05, dist) * (0.5 + uTreble);`
2. Core bianco: `float core = smoothstep(0.15, 0.0, dist);`
3. Colore dalla palette

---

## 4. Palette Colori (5 Temi Acidi)

Slider Tema: 0 → Rosa Tossica, 1 → Giallo Bile, 2 → Blu Elettrico, 3 → B/N Sporco, 4 → Arcobaleno Malato

| # | Nome | Colori principali |
|---|------|-------------------|
| 0 | Rosa Tossica | `#ff0080` (hotPink), `#ccff00` (acid), `#6600cc` (deepPurple), `#ff4400` (blood) |
| 1 | Giallo Bile | `#ffff00` (lemon), `#ff0066` (hotMagenta), `#00ff33` (toxicGreen), `#ff2200` (bloodOrange) |
| 2 | Blu Elettrico | `#00ffff` (neonCyan), `#ff00ff` (magenta), `#ffff00` (acid), `#001133` (deep) |
| 3 | B/N Sporco | grigio animato + neon cyan sporadico modulato da `uTreble` |
| 4 | Arcobaleno Malato | RGB sfasati con glitch bianco randomico |

---

## 5. Audio & Trigger

- **FFT:** 2048 bin
- **Bassi (0-150Hz):** Displacement sfera (amplificatore 2.0x), espansione particelle
- **Medi (150-2500Hz):** Trigger mutazione (cambio `uSeed` istantaneo quando > 0.55, cooldown 0.2s)
- **Alti (2500-8000Hz):** Lampeggio particelle, intensità glow, saturazione colore
- **Equalizer:** 8 barre UI (mantenuto)

---

## 6. Struttura File (index.html)

```
index.html
├── <head>: CSS (stili UI, overlay, equalizer, bottoni)
├── <body>
│   ├── <div id="canvas-container"> — Renderer Three.js
│   ├── <div id="ui-overlay"> — Pulsanti, slider, equalizer
│   └── <script>
│       ├── Shader GLSL inline (vsSphere, fsSphere, vsParticles, fsParticles)
│       ├── Variabili globali (renderer, camera, scene, mesh, material, buffer geometry)
│       ├── init() — Crea scene, mesh, particelle
│       ├── loadAudio() — Web Audio API, FFT 2048
│       ├── getAudioLevels() — Estrae bass/mid/treble
│       ├── setupUI() — Event listener bottoni e slider
│       ├── animate() — Loop principale: audio → update uniform → render
│       └── resize handler
└── Three.js r128 via CDN
```

---

## 7. Cosa viene Rimosso dalla Versione Attuale

- ❌ `SphereGeometry(1, 128, 128)` → `SphereGeometry(0.7, 256, 256)` (più piccola e densa)
- ❌ Wireframe toggle (eliminato — non serve all'estetica organismo)
- ❌ Particelle orbitanti circolari → Particelle con traiettorie sinusoidali indipendenti
- ❌ Palette originale (Rosa, Giallo, Blu, B/N, Arcobaleno standard) → Palette acide/malate
- ❌ `camera.position.z = 3` → `camera.position.z = 4` (sfera più piccola, serve più distanza)
- ❌ Scene rotation y += 0.15 (troppo veloce) → Scene rotation y += 0.005 (lenta e inquietante)

---

## 8. Criteri di Successo

1. La sfera ha una superficie **organica e malata** — mai liscia, mai simmetrica
2. Le particelle si muovono con **vita propria** — non sincronizzate, non orbitali
3. Il trigger mutazione **cambia istantaneamente** il pattern senza glitch
4. L'effetto complessivo è **caotico ma leggibile** — si capisce che è un organismo
5. L'UI (play/pause, slider, equalizer) **funziona come prima**
6. **Un solo file**, nessun bundler, Three.js via CDN

---

## 9. Vincoli Mantenuti

- Unico file `index.html`
- Nessun framework esterno
- Three.js r128 via CDN
- Cache disabilitata (`no-cache`)
- JavaScript ES6+ puro
- Commenti in italiano
- Shader GLSL inline come stringhe JS
