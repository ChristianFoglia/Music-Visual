# Design Spec: Nature Visual — Audio-Reactive 3D Sfera + Particelle

**Data:** 2026-06-12
**Branch:** nature-visual
**Target:** `index.html` (sostituisce la versione 2D fullscreen)
**Scopo:** Passare da un visualizer 2D caotico a una scena 3D pulita con sfera liquida al centro e particelle orbitanti, mantenendo tutte le feature audio avanzate.

---

## 1. Visione Visiva Finale

Uno **spazio buio profondo** (sfondo nero totale) con al centro una **sfera luminosa e pulsante** — come un pianeta vivente o un nucleo di energia liquida. La superficie della sfera è distorta dal noise e colorata da gradienti che scorrono. Attorno, **3000 particelle** formano anelli e spirali come polvere cosmica. Quando arriva un beat, la sfera si gonfia e le particelle accelerano o esplodono.

Effetti chiave:
- Scie lunghe e persistenti (ping-pong feedback sulla sfera)
- Mutazione istantanea del pattern al ritmo dei beat
- Glow neon su sfondo nero
- Respirazione fluida, non caotica

---

## 2. Componenti della Scena

### 2.1 Sfera Centrale (Mesh + ShaderMaterial)
- **Geometria:** `SphereGeometry(raggio=2.5, 256 segmenti, 256 segmenti)` — superficie super densa per displacement fluido
- **Vertex Shader:** Displacement dei vertici lungo la normale tramite `snoise(vec3)` 3D. Il noise è animato nel tempo e reagisce ai bassi.
- **Fragment Shader:** Colore basato su posizione normale + palette tematica. Glow radiale. Opacità e saturazione modulate dall'audio.
- **Reattività:**
  - Bassi → displacement maggiore (la sfera "respira")
  - Medi → trigger mutazione: cambio istantaneo del `uSeed` del noise
  - Alti → intensità colore e glow aumentano

### 2.2 Particelle Orbitanti (Points + BufferGeometry)
- **Geometria:** BufferGeometry con 3000 vertici posizionati su spirale/elica attorno alla sfera
- **Shader:** Vertex shader per posizione orbitale animata. Fragment shader per glow circolare.
- **Reattività:**
  - Bassi → espansione radiale (esplodono fuori)
  - Medi → cambio velocità orbitale
  - Alti → dimensione e luminosità aumentano

### 2.3 Sistema di Feedback (Ping-Pong Post-Processing)
- **Architettura:** 3 render target: `sceneRT` (frame corrente 3D), `feedbackA`, `feedbackB`
- **Passo 1:** Renderizza la scena 3D (sfera + particelle) su `sceneRT`
- **Passo 2:** Renderizza un quad fullscreen su `feedbackA` con shader di post-processing che legge `sceneRT` come input corrente e `feedbackB` come frame precedente, applica fade (0.92), zoom leggero (0.998) e blur minimo
- **Passo 3:** Displaya `feedbackA` sullo schermo
- **Passo 4:** Scambia `feedbackA` ↔ `feedbackB`
- **Effetto:** Le scie persistenti sono create nel pass di post-processing, non sulla texture della sfera. La scena 3D resta pulita e definita, mentre il feedback crea la coda visiva liquida.

### 2.4 Camera e Renderer
- `PerspectiveCamera(fov=60, near=0.1, far=100)` posizionata a z=6
- `WebGLRenderer` con antialias, dimensione fullscreen, pixelRatio max 2
- Sfondo nero totale (`renderer.setClearColor(0x000000)`)

---

## 3. Shader GLSL — Dettaglio

### Vertex Shader Sfera (`vsSphere`)
Input: `position`, `normal`, `uv`
Uniform: `uTime`, `uBass`, `uSeed`
Algoritmo:
1. Calcola `noise = snoise(normal * scale + uTime * speed + uSeed)`
2. Displacement: `position += normal * noise * displacementAmount * (1.0 + uBass)`
3. Trasforma in clip space

### Fragment Shader Sfera (`fsSphere`)
Input: `vNormal`, `vPosition`, `vUv`
Uniform: `uTime`, `uBass`, `uTreble`, `uTheme`, `uSeed`
Algoritmo:
1. Calcola colore dalla palette tematica basato su `vNormal` e `uTime`
2. Aggiungi glow radiale basato sulla distanza dal centro visivo
3. Modula saturazione con `uBass`
4. Aggiungi flash su beat forti
5. Il feedback e le scie persistenti sono gestiti dal pass di post-processing, non dalla sfera

### Vertex Shader Particelle (`vsParticles`)
Input: `position` (attributo), `aPhase` (attributo per offset animazione)
Uniform: `uTime`, `uBass`, `uMid`
Algoritmo:
1. Posizione base su spirale: calcolata da `aPhase`
2. Orbita attorno a Y basata su `uTime`
3. Espansione radiale con `uBass`
4. Dimensione punti modulata da `uMid`

### Fragment Shader Particelle (`fsParticles`)
Uniform: `uTheme`, `uTreble`
Algoritmo:
1. Glow circolare attorno al centro del punto (distance-based)
2. Colore dalla palette tematica
3. Opacità e luminosità modulate da `uTreble`

---

## 4. Palette Colori (5 Temi — identiche a reaction-diffusion.html)

Slider Tema: 0 → Rosa, 1 → Giallo, 2 → Blu, 3 → B/N, 4 → Arcobaleno

### Tema 0: Rosa (idx < 0.5)
- hotPink `#ff0080`, electricBlue `#3380ff`, acidGreen `#99ff33`, deepPurple `#6600cc`
- Mix basato su `wave`, `wave2`, `wave3` animate nel tempo

### Tema 1: Giallo (idx < 1.5)
- lemon `#ffff00`, hotMagenta `#ff0099`, toxicGreen `#00ff4d`, bloodOrange `#ff3300`
- Mix basato su `wave`, `wave2`, `wave3` animate nel tempo

### Tema 2: Blu (idx < 2.5)
- navy `#001a99`, neonCyan `#00ffff`, sunsetOrange `#ff4d1a`, laserPink `#ff00cc`
- Mix basato su `wave`, `wave2`, `wave3` animate nel tempo

### Tema 3: B/N (idx < 3.5)
- Scala di grigi animata + neonCyan `#00ffff` modulato da `uTreble`
- Mix tra bianco/nero e neon basato su `tint * uTreble * 0.4`

### Tema 4: Arcobaleno (else)
- RGB sinusoidali sfasati di 120° (`2.094`, `4.189` rad)
- Intensità modulata da `0.7 + uTreble * 0.6`

---

## 5. Audio & Trigger

- **FFT:** 2048 bin (mantenuto)
- **Bassi (0-150Hz):** Displacement sfera, espansione particelle
- **Medi (150-2500Hz):** Trigger mutazione (cambio `uSeed` istantaneo quando > 0.55, cooldown 0.2s)
- **Alti (2500-8000Hz):** Glow, luminosità particelle, saturazione colore
- **Equalizer:** 8 barre UI (mantenuto)

---

## 6. Struttura File (index.html)

```
index.html
├── <head>: CSS (stili UI, overlay, equalizer, bottoni)
├── <body>
│   ├── <div id="canvas-container"> — Contiene il renderer Three.js
│   ├── <div id="ui-overlay"> — Pulsanti, slider, equalizer
│   └── <script>
│       ├── Shader GLSL inline (vsSphere, fsSphere, vsParticles, fsParticles, vsFullscreen, fsFeedback)
│       ├── Variabili globali (renderer, camera, scene, mesh, material, buffer geometry, render targets)
│       ├── init() — Crea scene, mesh, particelle, render targets
│       ├── setupRenderTargets() — Ping-pong A/B
│       ├── loadAudio() — Web Audio API, FFT 2048
│       ├── getAudioLevels() — Estrae bass/mid/treble
│       ├── setupUI() — Event listener bottoni e slider
│       ├── animate() — Loop principale: audio → update uniform → render ping-pong → display
│       └── resize handler
└── Three.js r128 via CDN
```

---

## 7. Cosa viene Rimosso dalla Versione Attuale

- ❌ Shader fullscreen 2D (`fsSim` con kaleidoscope, mirror, aberrazione, blur radiale, posterizzazione)
- ❌ Render target usati per fullscreen 2D
- ❌ `OrthographicCamera` (sostituita con PerspectiveCamera)
- ❌ Palette Cyberpunk/Magma/Ocean → **Palette mantenute: Rosa/Giallo/Blu/BN/Arcobaleno** (identiche a reaction-diffusion.html)
- ❌ Pattern concentrico "occhio/volto"
- ❌ Reaction-diffusion approssimata

---

## 8. Criteri di Successo

1. La sfera ha una superficie **liquida e fluida**, non caotica
2. Le particelle **orbitano stabilmente** e reagiscono ai beat senza diventare "zuppa"
3. Il feedback ping-pong crea **scie persistenti** visibili ma non invadenti
4. Il trigger mutazione **cambia istantaneamente** il pattern senza glitch visivi
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
