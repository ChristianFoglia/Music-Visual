# Audio-Visual Three.js Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Creare un file HTML autonomo (`audio-visual.html`) che combina Three.js, shader GLSL personalizzato e Web Audio API per un visual audio-reattivo in stile TouchDesigner.

**Architecture:** File monolite HTML con CSS inline, JavaScript inline, Three.js da CDN. Shader GLSL come stringhe JS. Web Audio API per analisi FFT dell'audio caricato dall'utente. UI overlay con controlli per forma, colore e caricamento audio.

**Tech Stack:** HTML5, CSS3, Vanilla JavaScript, Three.js (CDN r160+), Web Audio API, GLSL.

---

### File Structure

- **Create:** `audio-visual.html` — unico file contenente tutto (HTML, CSS, JS, shader GLSL inline).

---

### Task 1: Scaffolding HTML e CSS

**Files:**
- Create: `audio-visual.html`

- [ ] **Step 1: Scrivi la struttura HTML base con CSS inline**

Il file deve avere:
- `<!DOCTYPE html>`, `<html>`, `<head>`, `<body>`
- Meta viewport per mobile
- Import Three.js da CDN (`https://unpkg.com/three@0.160.0/build/three.module.js` via importmap, o `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js` via script classico)
- CSS: body margin 0, overflow hidden, background #050505
- CSS: overlay UI in alto a sinistra, sfondo rgba(0,0,0,0.6), padding, bordi arrotondati, colore testo bianco, font sans-serif
- CSS: pulsanti e slider stilizzati con colori neon (bordo azzurro/viola, sfondo scuro)
- CSS: mini equalizer container flex con barre
- Elementi DOM: `#canvas-container`, `#ui-overlay` con: file input nascosto, bottone "Carica Canzone", bottone "Cambia Canzone", slider "Forma" (0-1), slider "Tema Colore" (0-3), display nome file, container `#equalizer` per 8 barre

```html
<!DOCTYPE html>
<html lang="it">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Audio Visual Three.js</title>
  <style>
    body { margin: 0; overflow: hidden; background: #050505; color: #fff; font-family: 'Segoe UI', sans-serif; }
    #canvas-container { width: 100vw; height: 100vh; position: fixed; top: 0; left: 0; z-index: 1; }
    #ui-overlay { position: fixed; top: 20px; left: 20px; z-index: 10; background: rgba(0,0,0,0.6); padding: 16px 20px; border-radius: 12px; border: 1px solid rgba(0,255,255,0.3); backdrop-filter: blur(4px); max-width: 300px; }
    #ui-overlay h1 { margin: 0 0 12px 0; font-size: 16px; color: #0ff; text-transform: uppercase; letter-spacing: 2px; }
    .btn { background: rgba(0,255,255,0.1); border: 1px solid #0ff; color: #0ff; padding: 8px 14px; border-radius: 6px; cursor: pointer; font-size: 12px; text-transform: uppercase; letter-spacing: 1px; transition: all 0.2s; margin-right: 8px; margin-bottom: 8px; }
    .btn:hover { background: rgba(0,255,255,0.25); box-shadow: 0 0 12px rgba(0,255,255,0.4); }
    .slider-row { margin: 8px 0; display: flex; align-items: center; justify-content: space-between; }
    .slider-row label { font-size: 11px; text-transform: uppercase; letter-spacing: 1px; color: #aaa; width: 100px; }
    .slider-row input[type="range"] { flex: 1; margin-left: 8px; accent-color: #0ff; }
    .slider-row span { font-size: 11px; color: #0ff; width: 40px; text-align: right; }
    #file-name { font-size: 11px; color: #888; margin-top: 6px; font-style: italic; }
    #equalizer { display: flex; gap: 3px; align-items: flex-end; height: 30px; margin-top: 10px; }
    .eq-bar { width: 8px; background: linear-gradient(to top, #0ff, #f0f); border-radius: 2px; min-height: 2px; transition: height 0.05s linear; }
  </style>
</head>
<body>
  <div id="canvas-container"></div>
  <div id="ui-overlay">
    <h1>Audio Visual</h1>
    <input type="file" id="audio-input" accept="audio/*" style="display:none;">
    <button class="btn" id="btn-load">Carica Canzone</button>
    <button class="btn" id="btn-change" style="display:none;">Cambia Canzone</button>
    <div class="slider-row">
      <label>Forma</label>
      <input type="range" id="slider-shape" min="0" max="1" step="0.01" value="0">
      <span id="shape-label">Sfera</span>
    </div>
    <div class="slider-row">
      <label>Tema</label>
      <input type="range" id="slider-theme" min="0" max="3" step="1" value="0">
      <span id="theme-label">Cyber</span>
    </div>
    <div id="file-name">Nessuna canzone caricata</div>
    <div id="equalizer"></div>
  </div>
  <!-- Script verra inserito nei task successivi -->
</body>
</html>
```

- [ ] **Step 2: Verifica struttura**

Apri `audio-visual.html` nel browser (doppio clic). Dovresti vedere:
- Sfondo nero
- Overlay UI in alto a sinistra con titolo "Audio Visual", pulsanti e slider
- Nessun errore nella console (F12 > Console)

---

### Task 2: Setup Three.js Scene e Geometrie

**Files:**
- Modify: `audio-visual.html` (aggiungi script prima di `</body>`)

- [ ] **Step 1: Inizializza Three.js scene, camera, renderer**

Aggiungi dentro un tag `<script type="module">`:
- Importa `* as THREE` da `https://unpkg.com/three@0.160.0/build/three.module.js`
- Crea `scene = new THREE.Scene()`
- Crea `camera = new THREE.PerspectiveCamera(60, window.innerWidth/window.innerHeight, 0.1, 100)`
  - Posiziona camera a `z = 3`
- Crea `renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false })`
  - `renderer.setSize(window.innerWidth, window.innerHeight)`
  - `renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))`
  - Aggiungi renderer.domElement a `#canvas-container`
- Gestisci `window.addEventListener('resize', ...)` per aggiornare camera e renderer
- Aggiungi `OrbitControls` da `https://unpkg.com/three@0.160.0/examples/jsm/controls/OrbitControls.js` (opzionale ma utile per designer)

- [ ] **Step 2: Crea geometrie Sfera e Piano**

Crea due mesh con `ShaderMaterial` (shader verranno definiti nel Task 3):
- `sphereGeo = new THREE.SphereGeometry(1, 128, 128)`
- `planeGeo = new THREE.PlaneGeometry(3, 3, 200, 200)`
- Crea un `shaderMaterial` condiviso (o due materiali identici) per entrambe le mesh
- Aggiungi entrambe le mesh alla scene
- Inizialmente: sfera visibile (scale 1), piano nascosto (scale 0.001 o visibility controllata dallo shader)

---

### Task 3: Shader GLSL (Vertex + Fragment)

**Files:**
- Modify: `audio-visual.html` (definisci le stringhe shader nel tag `<script>`)

- [ ] **Step 1: Scrivi Vertex Shader con 3D Simplex Noise inline**

Il vertex shader deve contenere:
- Funzione `snoise(vec3 v)` — implementazione completa di Simplex Noise 3D (classico codice GLSL di Ashima Arts, ~120 righe). Copia la versione standard che restituisce `float` in range [-1, 1].
- Uniforms: `u_time`, `u_bass`, `u_treble`, `u_shapeMix` (0=sfera, 1=piano)
- Attributi: `position`, `normal` (per sfera), oppure calcolo normale implicito.
- Per la sfera: `displacement = normal * (snoise(position * 2.0 + u_time * 0.5) * 0.3 + u_bass * 1.5)`
- Per il piano: `displacement = vec3(0.0, snoise(vec3(position.x * 2.0, position.y * 2.0, u_time * 0.5)) * 0.3 + u_bass * 1.0, 0.0)`
- Mischia le due deformazioni usando `u_shapeMix`: `mixedPos = mix(spherePos, planePos, u_shapeMix)`
- Passa al fragment: `varying vec3 vNormal`, `varying vec3 vPosition`, `varying float vDisplacement`

Nota: il `ShaderMaterial` di Three.js supporta già le matrici di proiezione/modelView, quindi il vertex shader finale deve includere `project_vertex` standard o usare la struttura Three.js raw shader.

Semplifica: usa `onBeforeCompile` oppure definisci lo shader completo con:
```glsl
uniform float u_time;
uniform float u_bass;
uniform float u_treble;
uniform float u_shapeMix;
varying vec3 vNormal;
varying vec3 vPosition;
varying float vDisplacement;

// --- snoise function here ---

void main() {
  vec3 sphereDisp = normal * (snoise(position * 2.0 + u_time * 0.5) * 0.3 + u_bass * 1.5);
  vec3 planePos = position;
  vec3 planeDisp = vec3(0.0, snoise(vec3(position.x * 2.0, position.y * 2.0, u_time * 0.5)) * 0.3 + u_bass * 1.0, 0.0);
  
  vec3 finalSphere = position + sphereDisp;
  vec3 finalPlane = planePos + planeDisp;
  
  vec3 finalPos = mix(finalSphere, finalPlane, u_shapeMix);
  vDisplacement = length(mix(sphereDisp, planeDisp, u_shapeMix));
  vNormal = normal;
  vPosition = finalPos;
  
  gl_Position = projectionMatrix * modelViewMatrix * vec4(finalPos, 1.0);
}
```

- [ ] **Step 2: Scrivi Fragment Shader con palette colori e Fresnel**

Fragment shader deve contenere:
- Uniforms: `u_colorTheme`, `u_time`, `u_treble`
- Varyings: `vNormal`, `vPosition`, `vDisplacement`
- Funzione `palette(float t, float theme)` che ritorna `vec3`:
  - `theme == 0.0`: Cyberpunk — mix di viola (0.5, 0.0, 1.0), azzurro (0.0, 1.0, 1.0), fucsia (1.0, 0.0, 0.5). Usa `sin` e `cos` di `t + u_time * 0.2 * (1.0 + u_treble)` per shift dinamico.
  - `theme == 1.0`: Magma — arancio (1.0, 0.4, 0.0), rosso (1.0, 0.0, 0.0), oro (1.0, 0.8, 0.0). Shift lento.
  - `theme == 2.0`: Ocean — verde acqua (0.0, 1.0, 0.8), blu notte (0.0, 0.2, 0.6), turchese (0.0, 0.8, 1.0). Shift ondulato.
  - `theme == 3.0`: Psychedelic — `vec3(sin(t*6.28)*0.5+0.5, sin(t*6.28+2.09)*0.5+0.5, sin(t*6.28+4.18)*0.5+0.5)`. Shift molto veloce con `u_time * 0.5 * (1.0 + u_treble * 2.0)`.
- Interpolazione tra theme N e N+1 se `u_colorTheme` ha decimali (smooth).
- Fresnel: `float fresnel = pow(1.0 - dot(normalize(vNormal), vec3(0.0,0.0,1.0)), 3.0)` — aggiunge glow sui bordi.
- Colore finale: `baseColor * (0.8 + vDisplacement * 2.0) + vec3(fresnel * 0.4)`.
- Alpha: 1.0 (nessuna trasparenza necessaria).

- [ ] **Step 3: Assegna ShaderMaterial alle mesh**

Crea `new THREE.ShaderMaterial({ uniforms: {...}, vertexShader: ..., fragmentShader: ..., side: THREE.DoubleSide })`
Uniforms iniziali:
- `u_time: { value: 0.0 }`
- `u_bass: { value: 0.0 }`
- `u_treble: { value: 0.0 }`
- `u_colorTheme: { value: 0.0 }`
- `u_shapeMix: { value: 0.0 }`

Assegna a entrambe le mesh. Nella funzione di render, aggiorna i uniforms.

---

### Task 4: Web Audio API Setup

**Files:**
- Modify: `audio-visual.html`

- [ ] **Step 1: Implementa caricamento e riproduzione audio**

Nel modulo JS:
- Seleziona `audioInput = document.getElementById('audio-input')`, `btnLoad`, `btnChange`, `fileNameDisplay`
- `btnLoad.addEventListener('click', () => audioInput.click())`
- `audioInput.addEventListener('change', (e) => { ... })`
- All'interno del change handler:
  - `file = e.target.files[0]`
  - Se non c'è file, return
  - `fileNameDisplay.textContent = file.name`
  - `btnLoad.style.display = 'none'`
  - `btnChange.style.display = 'inline-block'`
  - Crea `audioElement = document.createElement('audio')`
  - `audioElement.src = URL.createObjectURL(file)`
  - `audioElement.crossOrigin = 'anonymous'` (essenziale per Web Audio)
  - `audioElement.loop = true`
  - Crea `audioContext = new (window.AudioContext || window.webkitAudioContext)()`
  - `source = audioContext.createMediaElementSource(audioElement)`
  - `analyser = audioContext.createAnalyser()`
  - `analyser.fftSize = 512` (o 1024 per più dettaglio)
  - `source.connect(analyser)`
  - `analyser.connect(audioContext.destination)`
  - `audioElement.play()`
  - Salva `analyser`, `audioElement`, `audioContext` in variabili globali del modulo
- `btnChange.addEventListener('click', () => { audioElement.pause(); audioInput.value = ''; audioInput.click(); })`
- Gestisci il resume del context se sospeso: `document.addEventListener('click', () => { if (audioContext && audioContext.state === 'suspended') audioContext.resume(); }, {once:true})`

- [ ] **Step 2: Estrai bassi e alti dai dati FFT**

Funzione `getAudioData()` chiamata ogni frame:
- `dataArray = new Uint8Array(analyser.frequencyBinCount)`
- `analyser.getByteFrequencyData(dataArray)`
- `sampleRate = audioContext.sampleRate` (o assumi 44100)
- `binCount = analyser.frequencyBinCount`
- `nyquist = sampleRate / 2`
- `freqPerBin = nyquist / binCount`
- **Bassi (0-100 Hz):** `bassEndBin = Math.floor(100 / freqPerBin)` → calcola media di `dataArray[0...bassEndBin]`, normalizza a `/255`
- **Alti (2000-8000 Hz):** `trebleStartBin = Math.floor(2000 / freqPerBin)`, `trebleEndBin = Math.floor(8000 / freqPerBin)` → media di quel range, normalizza a `/255`
- Ritorna `{ bass, treble }`

---

### Task 5: Loop di Animazione e Uniform Updates

**Files:**
- Modify: `audio-visual.html`

- [ ] **Step 1: Implementa requestAnimationFrame loop**

Crea funzione `animate(time)`:
- `requestAnimationFrame(animate)`
- `elapsed = time * 0.001` (secondi)
- Aggiorna `shaderMaterial.uniforms.u_time.value = elapsed`
- Se `analyser` esiste:
  - `{ bass, treble } = getAudioData()`
  - **Smoothing (lerp):** `currentBass = shaderMaterial.uniforms.u_bass.value`, `targetBass = bass`. Nuovo valore: `currentBass + (targetBass - currentBass) * 0.15`. Stesso per treble con fattore 0.2.
  - Assegna a `u_bass.value` e `u_treble.value`
- Aggiorna mini equalizer:
  - Dividi `dataArray` in 8 bande, calcola media per ogni banda, imposta `height` delle `.eq-bar` in percentuale
- Aggiorna scale mesh per transizione forma:
  - `shapeMix = parseFloat(document.getElementById('slider-shape').value)`
  - `sphereMesh.scale.setScalar(1.0 - shapeMix)`
  - `planeMesh.scale.setScalar(shapeMix < 0.01 ? 0.001 : shapeMix)` (evita scale 0)
  - `shaderMaterial.uniforms.u_shapeMix.value = shapeMix`
- Aggiorna `u_colorTheme.value = parseFloat(document.getElementById('slider-theme').value)`
- `renderer.render(scene, camera)`

- [ ] **Step 2: Collega eventi slider agli shader**

- `sliderShape.addEventListener('input', (e) => { update label "Sfera"/"Piano" o percentuale; })`
- `sliderTheme.addEventListener('input', (e) => { update label in base al valore (0=Cyber,1=Magma,2=Ocean,3=Psyche); })`

---

### Task 6: Mini Equalizer UI

**Files:**
- Modify: `audio-visual.html`

- [ ] **Step 1: Popola le barre dell'equalizer**

Nel CSS e JS:
- All'avvio, crea 8 div `.eq-bar` dentro `#equalizer` (puoi crearli dinamicamente nel JS all'inizio)
- Ogni frame, aggiorna `style.height` in base ai dati FFT delle 8 bande.

---

### Task 7: Test, Polish e Commit

**Files:**
- Modify: `audio-visual.html` (polish)

- [ ] **Step 1: Verifica funzionamento end-to-end**

1. Apri `audio-visual.html` nel browser (Chrome/Edge/Firefox).
2. Clicca "Carica Canzone" → seleziona un file MP3.
3. L'audio dovrebbe partire.
4. Dovresti vedere:
   - La sfera che pulsa e si deforma con i bassi.
   - I colori cambiare in base al tema selezionato.
   - Il mini equalizer che balla.
5. Muovi lo slider "Forma" → la sfera dovrebbe rimpicciolirsi e il piano apparire.
6. Muovi lo slider "Tema" → i colori dovrebbero cambiare tra Cyberpunk, Magma, Ocean, Psychedelic.
7. Clicca "Cambia Canzone" → dovrebbe fermarsi e permettere nuova selezione.

- [ ] **Step 2: Correggi eventuali bug visivi**

Problemi comuni da controllare:
- Se il piano non si vede: verifica che sia ruotato `-Math.PI/2` sull'asse X (per essere orizzontale) o che la camera lo guardi dal davanti.
- Se lo shader non compila: controlla la console per errori GLSL (Three.js li mostra in rosso).
- Se l'audio non viene analizzato: verifica `crossOrigin = 'anonymous'` e che `audioContext` sia `running`.

- [ ] **Step 3: Aggiungi commenti esplicativi**

Aggiungi commenti in italiano (o inglese semplice) per ogni sezione del codice, specialmente:
- Come funziona il noise
- Come vengono estratti bassi e alti
- Come funziona la palette colore
- Perché il file è monolite

- [ ] **Step 4: Commit del file finale**

```bash
git add audio-visual.html
git commit -m "feat: add audio-reactive 3D visual with Three.js and custom GLSL shader"
```

---

### Spec Coverage Check

| Requisito Spec | Task che lo implementa |
|---|---|
| File HTML autonomo | Task 1, 2, 3, 4, 5, 6 tutti in `audio-visual.html` |
| Three.js da CDN | Task 2 |
| Sfera 128x128 e Piano 200x200 | Task 2 |
| Shader GLSL con 3D Noise | Task 3 |
| Deformazione liquido/magmatica | Task 3 (vertex) |
| Web Audio API con file input | Task 4 |
| Uniforms `u_bass`, `u_treble` | Task 4, 5 |
| Colori vibranti neon/cyberpunk | Task 3 (fragment) |
| 4 palette con slider | Task 3, 5 |
| Sfondo scuro | Task 1 (CSS) |
| Switch Sfera/Piano | Task 1 (UI), Task 2 (mesh), Task 5 (animazione) |
| Mini equalizer | Task 6 |
| Commenti | Task 7 |

---

### Placeholder Scan

Nessun "TBD", "TODO" o riferimento a funzioni non definite. Tutto il codice GLSL e JS è presente nei task.

---

### Type Consistency Check

- `u_colorTheme`: float 0.0-3.0 (anche se slider è step 1, shader gestisce interpolazione)
- `u_shapeMix`: float 0.0-1.0
- `u_bass`, `u_treble`: float 0.0-1.0
- Tutti gli uniforms sono `value` objects come richiesto da Three.js ShaderMaterial.
