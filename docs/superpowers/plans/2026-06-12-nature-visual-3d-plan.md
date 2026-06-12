# Nature Visual 3D — Sfera + Particelle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Sostituire il visualizer 2D fullscreen con una scena 3D audio-reattiva: sfera liquida al centro (256×256) + 3000 particelle orbitanti + post-processing feedback ping-pong, mantenendo le palette e le feature audio esistenti.

**Architecture:** Scena 3D con PerspectiveCamera → sfera con ShaderMaterial (vertex displacement via noise 3D + fragment palette) + Points con BufferGeometry (3000 vertici su spirale) → render su `sceneRT` → post-processing ping-pong (mix `sceneRT` + `feedbackPrev` con fade/zoom/blur) → display finale. Tutto in un unico `index.html`.

**Tech Stack:** HTML5, CSS3, JavaScript ES6+, Three.js r128 via CDN, Web Audio API, GLSL inline.

---

## File Structure

| File | Azione | Descrizione |
|------|--------|-------------|
| `index.html` | **Sovrascrivi** | File unico: HTML + CSS + JS + GLSL. Sostituisce completamente la versione 2D. |

---

## Task 1: Setup Base HTML/CSS

**File:**
- Sovrascrivi: `index.html`

**Contesto:** La struttura HTML e il CSS rimangono quasi identici all'attuale. Serve solo il container `#canvas-container`, l'overlay UI con pulsanti, slider, equalizer, e il tag script per Three.js r128.

- [ ] **Step 1.1: Scrivere la struttura HTML + CSS**

Copia la struttura base dall'attuale `index.html` (linee 1–46), mantenendo: meta viewport, meta no-cache, stili body/canvas-container/ui-overlay/btn/slider-row/equalizer/error-box. Slider tema: `min="0" max="4" step="0.01"`. Tre.js via CDN.

Verifica: il file deve iniziare con `<!DOCTYPE html>` e includere il div `#canvas-container`, `#ui-overlay` con tutti i controlli, e lo script Three.js.

- [ ] **Step 1.2: Verifica HTML con browser (opzionale)**

Apri il file in un browser. Dovresti vedere solo l'UI overlay su sfondo nero (nessun canvas visibile fino a Task 5).

---

## Task 2: Shader GLSL Inline

**File:**
- Sovrascrivi (aggiungi dentro `<script>`): `index.html`

**Contesto:** Tutti gli shader sono stringhe JS inline. Servono 6 shader: vsSphere, fsSphere, vsParticles, fsParticles, vsFullscreen, fsPost (feedback), fsDisplay.

- [ ] **Step 2.1: Implementare `vsSphere` (Vertex Shader Sfera)**

Input: `position`, `normal`, `uv`.
Output: `vNormal`, `vUv`, `vPosition`.
Uniform: `uTime`, `uBass`, `uSeed`.
Algoritmo:
1. Include `snoise(vec3)` 3D (porta il simplex noise 2D dall'attuale a 3D: aggiungi la terza coordinata).
2. Calcola `noise = snoise(normal * 3.0 + uTime * 0.3 + uSeed)`.
3. Displacement: `float disp = noise * 0.25 * (1.0 + uBass * 0.8);` — la sfera "respira" con i bassi.
4. `vec3 newPos = position + normal * disp;`
5. Passa `vNormal = normal;`, `vUv = uv;`, `vPosition = newPos;`.
6. `gl_Position = projectionMatrix * modelViewMatrix * vec4(newPos, 1.0);`.

- [ ] **Step 2.2: Implementare `fsSphere` (Fragment Shader Sfera)**

Input: `vNormal`, `vUv`, `vPosition`.
Uniform: `uTime`, `uBass`, `uTreble`, `uTheme`, `uSeed`.
Algoritmo:
1. Include la funzione `palette(float t, float idx)` **identica** a quella in `reaction-diffusion.html` (linee 124–171).
2. Calcola `vec3 viewDir = normalize(cameraPosition - vPosition);` (richiede `cameraPosition` uniform implicita di Three.js).
3. Fresnel/glow: `float fresnel = pow(1.0 - max(dot(normalize(vNormal), viewDir), 0.0), 3.0);`
4. Colore base: `vec3 col = palette(vNormal.x + vNormal.y + uTime * 0.2, floor(uTheme));`
5. Aggiungi glow radiale: `col += vec3(1.0) * fresnel * (0.5 + uTreble);`
6. Saturazione con bassi: `col = mix(vec3(0.02), col, 0.5 + uBass * 0.5);`
7. Flash su beat forti: `if(uBass > 0.6) col += vec3(smoothstep(0.6, 0.9, uBass) * 0.3);`
8. `gl_FragColor = vec4(col, 1.0);`.

- [ ] **Step 2.3: Implementare `vsParticles` (Vertex Shader Particelle)**

Input: `position`, `aPhase` (attributo custom `float`).
Output: `vColor`, `vSize`.
Uniform: `uTime`, `uBass`, `uMid`, `uTheme`.
Algoritmo:
1. La posizione originale (`position`) è calcolata su una spirale in JS, ma qui la animiamo.
2. `float angle = aPhase * 6.28 + uTime * (0.3 + uMid * 0.5);` — orbita attorno a Y.
3. `float radius = length(position.xz) * (1.0 + uBass * 0.6);` — espansione radiale con i bassi.
4. `float y = position.y + sin(uTime * 0.5 + aPhase * 10.0) * 0.1;` — leggero ondeggiamento verticale.
5. `vec3 newPos = vec3(cos(angle) * radius, y, sin(angle) * radius);`
6. Passa `vColor = palette(aPhase + uTime * 0.1, floor(uTheme));`.
7. Passa `vSize = 1.0 + uMid * 3.0;` (usato nel fragment per dimensione glow).
8. `gl_Position = projectionMatrix * modelViewMatrix * vec4(newPos, 1.0);`
9. `gl_PointSize = vSize * (300.0 / gl_Position.z);` — size ridotto con distanza.

- [ ] **Step 2.4: Implementare `fsParticles` (Fragment Shader Particelle)**

Input: `vColor`, `vSize`.
Uniform: `uTreble`.
Algoritmo:
1. `vec2 coord = gl_PointCoord - vec2(0.5);`
2. `float dist = length(coord);`
3. `if(dist > 0.5) discard;` — forma circolare.
4. Glow: `float alpha = smoothstep(0.5, 0.1, dist) * (0.4 + uTreble * 0.6);`
5. `gl_FragColor = vec4(vColor * (1.0 + uTreble), alpha);`.

- [ ] **Step 2.5: Implementare `vsFullscreen` (Vertex Shader Quad Fullscreen)**

Già esistente nell'attuale `index.html` (linee 76–82). Copia identico:
```glsl
varying vec2 vUv;
void main(){
  vUv = uv;
  gl_Position = vec4(position.xy, 0.0, 1.0);
}
```

- [ ] **Step 2.6: Implementare `fsPost` (Fragment Shader Post-Processing / Feedback)**

Input: `vUv`.
Uniform: `tScene` (frame corrente 3D renderizzato), `tPrev` (frame precedente con feedback), `uTime`, `uBass`.
Algoritmo:
1. Zoom leggero: `float zoom = 0.998; vec2 centered = (vUv - 0.5) * zoom + 0.5;`
2. Blur minimo (4 campioni): `vec3 prev = texture2D(tPrev, centered).rgb * 0.5;` + 4 offset di ±0.001 a peso 0.125 ciascuno.
3. Fade: `float fade = 0.92; prev *= fade;`
4. Frame corrente: `vec3 scene = texture2D(tScene, vUv).rgb;`
5. Flash su beat: `if(uBass > 0.6) scene += vec3(smoothstep(0.6, 0.9, uBass) * 0.2);`
6. Blend: `vec3 final = mix(prev, scene, 0.25 + uBass * 0.15);`
7. `gl_FragColor = vec4(final, 1.0);`.

- [ ] **Step 2.7: Implementare `fsDisplay` (Fragment Shader Display Pass-Through)**

Già esistente nell'attuale `index.html` (linee 251–259). Copia identico:
```glsl
precision mediump float;
uniform sampler2D tDiffuse;
varying vec2 vUv;
void main(){
  gl_FragColor = vec4(texture2D(tDiffuse, vUv).rgb, 1.0);
}
```

---

## Task 3: Setup Scena 3D, Mesh e Particelle

**File:**
- Sovrascrivi (JS dentro `<script>`): `index.html`

**Contesto:** Crea la scena 3D con PerspectiveCamera, la sfera con ShaderMaterial, e le particelle con BufferGeometry + Points. Crea anche le scene di supporto per post-processing (quad fullscreen).

- [ ] **Step 3.1: Variabili globali e init() della scena 3D**

Dichiara:
```js
let renderer, perspectiveCamera, scene;
let sphereMesh, sphereMat;
let particles, particlesMat;
let sceneRT, feedbackA, feedbackB;
let postScene, postMesh, postMat;
let displayScene, displayMesh, displayMat;
```

In `init()`:
1. `renderer = new THREE.WebGLRenderer({ antialias: true });`
2. `renderer.setSize(window.innerWidth, window.innerHeight);`
3. `renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));`
4. `renderer.setClearColor(0x000000);`
5. Appendi al `#canvas-container`.
6. `perspectiveCamera = new THREE.PerspectiveCamera(60, w/h, 0.1, 100);`
7. `perspectiveCamera.position.z = 6;`
8. `scene = new THREE.Scene();` (nessuna luce, tutto emissive).

- [ ] **Step 3.2: Creare la sfera**

```js
const sphereGeo = new THREE.SphereGeometry(2.5, 256, 256);
sphereMat = new THREE.ShaderMaterial({
  uniforms: {
    uTime: { value: 0 },
    uBass: { value: 0 },
    uTreble: { value: 0 },
    uTheme: { value: 0 },
    uSeed: { value: 42 }
  },
  vertexShader: vsSphere,
  fragmentShader: fsSphere
});
sphereMesh = new THREE.Mesh(sphereGeo, sphereMat);
scene.add(sphereMesh);
```

- [ ] **Step 3.3: Creare le particelle (3000)**

```js
const PARTICLE_COUNT = 3000;
const positions = new Float32Array(PARTICLE_COUNT * 3);
const phases = new Float32Array(PARTICLE_COUNT);

for(let i = 0; i < PARTICLE_COUNT; i++){
  const angle = Math.random() * Math.PI * 2;
  const radius = 3.5 + Math.random() * 2.5; // tra 3.5 e 6
  const y = (Math.random() - 0.5) * 2;
  positions[i * 3] = Math.cos(angle) * radius;
  positions[i * 3 + 1] = y;
  positions[i * 3 + 2] = Math.sin(angle) * radius;
  phases[i] = Math.random();
}

const pGeo = new THREE.BufferGeometry();
pGeo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
pGeo.setAttribute('aPhase', new THREE.BufferAttribute(phases, 1));

particlesMat = new THREE.ShaderMaterial({
  uniforms: {
    uTime: { value: 0 },
    uBass: { value: 0 },
    uMid: { value: 0 },
    uTheme: { value: 0 }
  },
  vertexShader: vsParticles,
  fragmentShader: fsParticles,
  transparent: true,
  blending: THREE.AdditiveBlending,
  depthWrite: false
});

particles = new THREE.Points(pGeo, particlesMat);
scene.add(particles);
```

- [ ] **Step 3.4: Creare render targets e scene di post-processing**

```js
function setupRenderTargets(){
  const w = window.innerWidth;
  const h = window.innerHeight;
  const opts = {
    minFilter: THREE.LinearFilter,
    magFilter: THREE.LinearFilter,
    format: THREE.RGBAFormat,
    type: THREE.UnsignedByteType
  };
  sceneRT = new THREE.WebGLRenderTarget(w, h, opts);
  feedbackA = new THREE.WebGLRenderTarget(w, h, opts);
  feedbackB = new THREE.WebGLRenderTarget(w, h, opts);
}
```

Scene post-processing (ortho quad):
```js
const orthoCam = new THREE.OrthographicCamera(-1, 1, 1, -1, 0, 1);
const planeGeo = new THREE.PlaneGeometry(2, 2);

postScene = new THREE.Scene();
postMat = new THREE.ShaderMaterial({
  uniforms: {
    tScene: { value: null },
    tPrev: { value: null },
    uTime: { value: 0 },
    uBass: { value: 0 }
  },
  vertexShader: vsFullscreen,
  fragmentShader: fsPost,
  depthWrite: false
});
postMesh = new THREE.Mesh(planeGeo, postMat);
postScene.add(postMesh);

displayScene = new THREE.Scene();
displayMat = new THREE.ShaderMaterial({
  uniforms: { tDiffuse: { value: null } },
  vertexShader: vsFullscreen,
  fragmentShader: fsDisplay,
  depthWrite: false
});
displayMesh = new THREE.Mesh(planeGeo, displayMat);
displayScene.add(displayMesh);
```

---

## Task 4: Audio, UI e Loop di Animazione

**File:**
- Sovrascrivi (JS dentro `<script>`): `index.html`

**Contesto:** Mantieni tutta la logica audio esistente (loadAudio, getAudioLevels, equalizer, UI events). Riscrivi solo `animate()` per il nuovo pipeline di rendering 3D + post-processing.

- [ ] **Step 4.1: Copiare funzioni audio e UI dall'attuale**

Copia identiche le funzioni: `loadAudio`, `getAudioLevels`, `makeEq`, `updateEq`, `setupUI`, e tutti gli event listener. Mantieni anche `themeNames` e `eqBars`.

Nota: `themeNames` rimane `['Rosa','Giallo','Blu','B/N','Arcobaleno']`.

- [ ] **Step 4.2: Implementare il nuovo `animate(time)`**

```js
function animate(time){
  requestAnimationFrame(animate);
  const elapsed = time * 0.001;
  frameCount++;

  // Audio analysis
  let bass = 0, mid = 0, treble = 0;
  let freqData = null;
  if(analyser){
    const lv = getAudioLevels();
    bass = lv.bass; mid = lv.mid; treble = lv.treble;
    freqData = new Uint8Array(analyser.frequencyBinCount);
    analyser.getByteFrequencyData(freqData);
  }

  // Smoothing
  const curB = sphereMat.uniforms.uBass.value;
  const curT = sphereMat.uniforms.uTreble.value;
  sphereMat.uniforms.uBass.value += (bass - curB) * 0.12;
  sphereMat.uniforms.uTreble.value += (treble - curT) * 0.18;
  particlesMat.uniforms.uBass.value = sphereMat.uniforms.uBass.value;
  particlesMat.uniforms.uMid.value = mid;
  particlesMat.uniforms.uTreble.value = sphereMat.uniforms.uTreble.value;

  // Time
  sphereMat.uniforms.uTime.value = elapsed;
  particlesMat.uniforms.uTime.value = elapsed;

  // Theme
  const themeVal = parseFloat(document.getElementById('slider-theme').value);
  sphereMat.uniforms.uTheme.value = themeVal;
  particlesMat.uniforms.uTheme.value = themeVal;

  // Trigger Mutazione
  if(mid > 0.55 && (elapsed - lastMutationTime) > MUTATION_COOLDOWN){
    const seedAdd = 5.0 + Math.random() * 8.0;
    sphereMat.uniforms.uSeed.value += seedAdd;
    lastMutationTime = elapsed;
  }

  // === PASS 1: Render scena 3D su sceneRT ===
  renderer.setRenderTarget(sceneRT);
  renderer.render(scene, perspectiveCamera);

  // === PASS 2: Post-processing ping-pong su feedbackA ===
  postMat.uniforms.tScene.value = sceneRT.texture;
  postMat.uniforms.tPrev.value = feedbackB.texture;
  postMat.uniforms.uTime.value = elapsed;
  postMat.uniforms.uBass.value = bass;
  renderer.setRenderTarget(feedbackA);
  renderer.render(postScene, orthoCam);

  // === PASS 3: Display feedbackA sullo schermo ===
  renderer.setRenderTarget(null);
  displayMat.uniforms.tDiffuse.value = feedbackA.texture;
  renderer.render(displayScene, orthoCam);

  // === SWAP feedbackA <-> feedbackB ===
  const temp = feedbackA; feedbackA = feedbackB; feedbackB = temp;

  // Equalizer
  if(freqData) updateEq(freqData);
}
```

- [ ] **Step 4.3: Implementare resize handler**

```js
function onResize(){
  const w = window.innerWidth;
  const h = window.innerHeight;
  renderer.setSize(w, h);
  perspectiveCamera.aspect = w / h;
  perspectiveCamera.updateProjectionMatrix();
  sceneRT.setSize(w, h);
  feedbackA.setSize(w, h);
  feedbackB.setSize(w, h);
}
```

---

## Task 5: Avvio e Test

**File:**
- Sovrascrivi: `index.html`

- [ ] **Step 5.1: Chiamate di avvio in fondo allo script**

```js
init();
setupRenderTargets();
setupUI();
animate(0);
```

- [ ] **Step 5.2: Verifica sintassi**

Apri il file in un browser. Non devono esserci errori in console. La scena deve mostrare:
- Sfera nera al centro (primi frame) che prende colore quando il noise si attiva
- Particelle bianche/piccole che orbitano
- UI overlay visibile con pulsanti e slider funzionanti

- [ ] **Step 5.3: Test audio**

Carica una canzone. Verifica che:
- Play/Pause funzioni
- Le barre dell'equalizer si muovano
- La sfera "respiri" (si espanda/contraiga) con i bassi
- Le particelle accelerino/espandano con i beat
- Il trigger mutazione cambi il pattern quando i medi sono forti
- Lo slider tema cambi i colori in tempo reale

---

## Spec Coverage Check

| Requisito Spec | Task che lo implementa |
|---------------|----------------------|
| Sfera 3D liquida (256×256, noise 3D displacement) | Task 2.1, 2.2, 3.2 |
| Particelle orbitanti 3000 | Task 2.3, 2.4, 3.3 |
| Palette 5 temi (Rosa/Giallo/Blu/BN/Arcobaleno) | Task 2.2 (palette identica) |
| Ping-pong feedback post-processing | Task 2.6, 3.4, 4.2 |
| Audio 2048 bin + bass/mid/treble | Task 4.1 (copia esistente) |
| Trigger mutazione (mid > 0.55) | Task 4.2 |
| UI play/pause/slider/equalizer | Task 4.1 (copia esistente) |
| Scie lunghe persistenti (fade 0.92) | Task 2.6 (fsPost) |
| Unico file, Three.js CDN, no cache | Task 1.1 |

**Nessun gap trovato.**

---

## Placeholder Scan

- Nessun "TBD", "TODO", "implement later"
- Nessun "add appropriate error handling" senza codice
- Codice GLSL completo in ogni step
- Funzioni audio/UI copiate integralmente dall'esistente
- Tutti i path dei file sono esatti

**Nessun placeholder trovato.**

---

## Type Consistency Check

- Uniform names: `uTime`, `uBass`, `uTreble`, `uMid`, `uTheme`, `uSeed` — consistenti in tutti gli shader e il JS.
- `lastMutationTime`, `MUTATION_COOLDOWN`, `frameCount` — mantenuti dall'esistente.
- Attributo particelle: `aPhase` (float) — dichiarato in JS come BufferAttribute(…, 1) e usato in GLSL.

**Nessuna inconsistenza trovata.**
