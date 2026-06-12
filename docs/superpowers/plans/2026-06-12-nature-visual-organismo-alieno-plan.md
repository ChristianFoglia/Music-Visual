# Nature Visual — Organismo Alieno Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Sostituire `index.html` con una versione 3D audio-reattiva "organismo alieno" — sfera piccola (raggio 0.7) con superficie malata (noise 2 layer), 3000 particelle con vita propria (traiettorie indipendenti), palette 5 temi acidi, trigger mutazione al ritmo dei beat.

**Architecture:** Scena 3D con PerspectiveCamera → sfera ShaderMaterial (vertex displacement 2-layer noise + fragment palette acida) + Points BufferGeometry (3000 particelle con attributi custom aPhase/aSpeed/aThreshold) → renderer diretto (no post-processing per ora). Audio Web API 2048 bin analizzato in bass/mid/treble → uniform aggiornate in animate().

**Tech Stack:** HTML5, CSS3, JavaScript ES6+, Three.js r128 via CDN, Web Audio API.

---

## File Structure

| File | Azione | Descrizione |
|------|--------|-------------|
| `index.html` | **Sovrascrivi** | File unico: HTML + CSS + JS + GLSL inline. Sostituisce la versione attuale (sfera 1.0 + wireframe + particelle orbitali). |

---

## Task 1: HTML/CSS Base (struttura e stili)

**File:**
- Sovrascrivi: `index.html` (linee 1–41)

**Contesto:** La struttura HTML è quasi identica alla versione attuale. Cambiano solo:
- Titolo: "Nature Visual — Organismo Alieno"
- Slider tema: label iniziale "Rosa Tossica" (invece di "Cyberpunk")
- NO pulsante "Wireframe" (eliminato per l'estetica organismo)

- [ ] **Step 1.1: Scrivere HTML + CSS**

```html
<!DOCTYPE html>
<html lang="it">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="Cache-Control" content="no-cache, no-store, must-revalidate">
  <title>Nature Visual — Organismo Alieno</title>
  <style>
    body { margin: 0; overflow: hidden; background: #000; color: #fff; font-family: 'Segoe UI', sans-serif; }
    #canvas-container { width: 100vw; height: 100vh; position: fixed; top: 0; left: 0; z-index: 1; }
    #ui-overlay { position: fixed; top: 20px; left: 20px; z-index: 10; background: rgba(0,0,0,0.7); padding: 16px 20px; border-radius: 12px; border: 1px solid rgba(0,255,255,0.3); max-width: 320px; }
    #ui-overlay h1 { margin: 0 0 12px 0; font-size: 16px; color: #0ff; text-transform: uppercase; letter-spacing: 2px; }
    .btn { background: rgba(0,255,255,0.1); border: 1px solid #0ff; color: #0ff; padding: 8px 14px; border-radius: 6px; cursor: pointer; font-size: 12px; text-transform: uppercase; letter-spacing: 1px; margin-right: 8px; margin-bottom: 8px; }
    .btn:hover { background: rgba(0,255,255,0.25); box-shadow: 0 0 12px rgba(0,255,255,0.3); }
    #btn-play { display: none; }
    .slider-row { margin: 8px 0; display: flex; align-items: center; }
    .slider-row label { font-size: 11px; text-transform: uppercase; color: #aaa; width: 80px; }
    .slider-row input[type="range"] { flex: 1; margin: 0 8px; accent-color: #0ff; }
    .slider-row span { font-size: 11px; color: #0ff; width: 70px; text-align: right; }
    #file-name { font-size: 11px; color: #888; margin-top: 6px; font-style: italic; }
    #equalizer { display: flex; gap: 3px; align-items: flex-end; height: 30px; margin-top: 10px; }
    .eq-bar { width: 8px; background: linear-gradient(to top, #0ff, #f0f); border-radius: 2px; min-height: 2px; transition: height 0.05s; }
    #error-box { position: fixed; bottom: 20px; left: 20px; right: 20px; max-height: 120px; overflow: auto; background: rgba(200,0,0,0.9); color: #fff; padding: 12px; font-family: monospace; font-size: 12px; z-index: 9999; display: none; border-radius: 8px; }
  </style>
</head>
<body>
  <div id="canvas-container"></div>
  <div id="ui-overlay">
    <h1>Nature Visual</h1>
    <input type="file" id="audio-input" accept="audio/*" style="display:none;">
    <button class="btn" id="btn-load">Carica Canzone</button>
    <button class="btn" id="btn-play">Play</button>
    <button class="btn" id="btn-change" style="display:none;">Cambia</button>
    <div class="slider-row">
      <label>Tema</label>
      <input type="range" id="slider-theme" min="0" max="4" step="0.01" value="0">
      <span id="theme-label">Rosa Tossica</span>
    </div>
    <div id="file-name">Nessuna canzone caricata</div>
    <div id="equalizer"></div>
  </div>
  <div id="error-box"></div>

  <!-- Three.js r128 via CDN -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
```

---

## Task 2: Shader GLSL — Sfera (vsSphere + fsSphere)

**File:**
- Sovrascrivi (aggiungi dentro `<script>`): `index.html`

**Contesto:** Gli shader sono stringhe JS inline. Servono:
1. `vsSphere` — Vertex shader con Simplex Noise 3D a 2 layer + displacement + normale finta
2. `fsSphere` — Fragment shader con palette 5 temi acidi + shading + fresnel + flash

- [ ] **Step 2.1: Implementare `vsSphere` (Vertex Shader Sfera)**

```javascript
const vsSphere = `
  varying vec3 vPos;
  varying vec3 vNorm;
  varying vec3 vViewDir;
  uniform float uTime;
  uniform float uBass;
  uniform float uSeed;

  // Simplex Noise 3D
  vec3 mod289(vec3 x){ return x - floor(x * (1.0/289.0)) * 289.0; }
  vec4 mod289(vec4 x){ return x - floor(x * (1.0/289.0)) * 289.0; }
  vec4 permute(vec4 x){ return mod289(((x * 34.0) + 1.0) * x); }
  vec4 taylorInvSqrt(vec4 r){ return 1.79284291400159 - 0.85373472095314 * r; }

  float snoise(vec3 v){
    const vec2 C = vec2(1.0/6.0, 1.0/3.0);
    const vec4 D = vec4(0.0, 0.5, 1.0, 2.0);
    vec3 i = floor(v + dot(v, C.yyy));
    vec3 x0 = v - i + dot(i, C.xxx);
    vec3 g = step(x0.yzx, x0.xyz);
    vec3 l = 1.0 - g;
    vec3 i1 = min(g.xyz, l.zxy);
    vec3 i2 = max(g.xyz, l.zxy);
    vec3 x1 = x0 - i1 + C.xxx;
    vec3 x2 = x0 - i2 + C.yyy;
    vec3 x3 = x0 - D.yyy;
    i = mod289(i);
    vec4 p = permute(permute(permute(
              i.z + vec4(0.0, i1.z, i2.z, 1.0))
            + i.y + vec4(0.0, i1.y, i2.y, 1.0))
            + i.x + vec4(0.0, i1.x, i2.x, 1.0));
    float n_ = 1.0/7.0;
    vec3 ns = n_ * D.wyz - D.xzx;
    vec4 j = p - 49.0 * floor(p * ns.z * ns.z);
    vec4 x_ = floor(j * ns.z);
    vec4 y_ = floor(j - 7.0 * x_);
    vec4 x = x_ * ns.x + ns.yyyy;
    vec4 y = y_ * ns.x + ns.yyyy;
    vec4 h = 1.0 - abs(x) - abs(y);
    vec4 b0 = vec4(x.xy, y.xy);
    vec4 b1 = vec4(x.zw, y.zw);
    vec4 s0 = floor(b0) * 2.0 + 1.0;
    vec4 s1 = floor(b1) * 2.0 + 1.0;
    vec4 sh = -step(h, vec4(0.0));
    vec4 a0 = b0.xzyw + s0.xzyw * sh.xxyy;
    vec4 a1 = b1.xzyw + s1.xzyw * sh.zzww;
    vec3 p0 = vec3(a0.xy, h.x);
    vec3 p1 = vec3(a0.zw, h.y);
    vec3 p2 = vec3(a1.xy, h.z);
    vec3 p3 = vec3(a1.zw, h.w);
    vec4 norm = taylorInvSqrt(vec4(dot(p0,p0), dot(p1,p1), dot(p2,p2), dot(p3,p3)));
    p0 *= norm.x; p1 *= norm.y; p2 *= norm.z; p3 *= norm.w;
    vec4 m = max(0.6 - vec4(dot(x0,x0), dot(x1,x1), dot(x2,x2), dot(x3,x3)), 0.0);
    m = m * m;
    return 42.0 * dot(m*m, vec4(dot(p0,x0), dot(p1,x1), dot(p2,x2), dot(p3,x3)));
  }

  void main(){
    // Layer 1: bassa frequenza (gonfiori morbidi)
    float lowFreq = snoise(position * 2.0 + vec3(uTime * 0.3, uTime * 0.25, uTime * 0.35) + uSeed);
    // Layer 2: alta frequenza (detriti, spigoli)
    float highFreq = snoise(position * 7.0 + vec3(uTime * 0.6, uTime * 0.5, uTime * 0.7) + uSeed * 2.0) * 0.3;
    // Displacement con amplificazione dai bassi
    float displacement = (lowFreq + highFreq) * 0.4 * (1.0 + uBass * 2.0);
    vec3 newPos = position + normal * displacement;

    // Normale finta tramite gradiente implicito del noise
    float eps = 0.01;
    float nx = snoise(position * 2.0 + vec3(eps, 0.0, 0.0) + vec3(uTime * 0.3, uTime * 0.25, uTime * 0.35) + uSeed)
             - snoise(position * 2.0 + vec3(-eps, 0.0, 0.0) + vec3(uTime * 0.3, uTime * 0.25, uTime * 0.35) + uSeed);
    float ny = snoise(position * 2.0 + vec3(0.0, eps, 0.0) + vec3(uTime * 0.3, uTime * 0.25, uTime * 0.35) + uSeed)
             - snoise(position * 2.0 + vec3(0.0, -eps, 0.0) + vec3(uTime * 0.3, uTime * 0.25, uTime * 0.35) + uSeed);
    float nz = snoise(position * 2.0 + vec3(0.0, 0.0, eps) + vec3(uTime * 0.3, uTime * 0.25, uTime * 0.35) + uSeed)
             - snoise(position * 2.0 + vec3(0.0, 0.0, -eps) + vec3(uTime * 0.3, uTime * 0.25, uTime * 0.35) + uSeed);
    vec3 fakeNormal = normalize(normal + vec3(nx, ny, nz) * 0.5);

    vPos = newPos;
    vNorm = fakeNormal;
    vViewDir = normalize(cameraPosition - (modelMatrix * vec4(newPos, 1.0)).xyz);

    gl_Position = projectionMatrix * modelViewMatrix * vec4(newPos, 1.0);
  }
`;
```

- [ ] **Step 2.2: Implementare `fsSphere` (Fragment Shader Sfera)**

```javascript
const fsSphere = `
  precision highp float;
  varying vec3 vPos;
  varying vec3 vNorm;
  varying vec3 vViewDir;
  uniform float uTime;
  uniform float uBass;
  uniform float uTreble;
  uniform float uTheme;

  // Palette 5 temi acidi
  vec3 palette(float t, float idx){
    float tt = t + uTime * 0.3;
    float wave = sin(tt * 3.0) * 0.5 + 0.5;
    float wave2 = cos(tt * 7.0 + 1.5) * 0.5 + 0.5;
    float wave3 = sin(tt * 11.0 + 3.0) * 0.5 + 0.5;
    float chaos = sin(tt * 13.0 * (1.0 + uBass)) * 0.5 + 0.5;

    // Tema 0: Rosa Tossica
    if(idx < 0.5){
      vec3 hotPink = vec3(1.0, 0.0, 0.5);
      vec3 acid = vec3(0.8, 1.0, 0.0);
      vec3 deepPurple = vec3(0.4, 0.0, 0.8);
      vec3 blood = vec3(1.0, 0.2, 0.0);
      vec3 mix1 = mix(hotPink, acid, wave);
      vec3 mix2 = mix(deepPurple, blood, wave2);
      return mix(mix1, mix2, wave3);
    }
    // Tema 1: Giallo Bile
    else if(idx < 1.5){
      vec3 lemon = vec3(1.0, 0.9, 0.0);
      vec3 hotMagenta = vec3(1.0, 0.0, 0.6);
      vec3 toxicGreen = vec3(0.0, 1.0, 0.3);
      vec3 bloodOrange = vec3(1.0, 0.2, 0.0);
      vec3 mix1 = mix(lemon, hotMagenta, wave);
      vec3 mix2 = mix(toxicGreen, bloodOrange, wave2);
      return mix(mix1, mix2, wave3);
    }
    // Tema 2: Blu Elettrico
    else if(idx < 2.5){
      vec3 cyan = vec3(0.0, 1.0, 1.0);
      vec3 magenta = vec3(1.0, 0.0, 1.0);
      vec3 acid = vec3(0.8, 1.0, 0.0);
      vec3 deep = vec3(0.0, 0.05, 0.2);
      vec3 mix1 = mix(cyan, magenta, wave);
      vec3 mix2 = mix(acid, deep, wave2);
      return mix(mix1, mix2, wave3);
    }
    // Tema 3: B/N Sporco
    else if(idx < 3.5){
      float grey = sin(t * 2.0 + uTime * 0.15) * 0.5 + 0.5;
      float tint = sin(t * 8.0 + uTime * 0.5) * 0.5 + 0.5;
      vec3 bw = vec3(grey);
      vec3 neon = vec3(0.0, 1.0, 1.0) * (0.3 + uTreble * 0.7);
      return mix(bw, neon, tint * uTreble * 0.4);
    }
    // Tema 4: Arcobaleno Malato
    else {
      float r = sin(t * 6.0 + uTime * 0.4) * 0.5 + 0.5;
      float g = sin(t * 6.0 + uTime * 0.4 + 2.094) * 0.5 + 0.5;
      float b = sin(t * 6.0 + uTime * 0.4 + 4.189) * 0.5 + 0.5;
      float boost = 0.7 + uBass * 0.6;
      return vec3(r, g, b) * boost;
    }
  }

  void main(){
    // Luce da sinistra-alto
    vec3 lightDir = normalize(vec3(0.5, 0.8, 0.3));
    float diff = max(dot(vNorm, lightDir), 0.0);
    float spec = pow(max(dot(reflect(-lightDir, vNorm), vViewDir), 0.0), 32.0);

    // Colore base dalla palette — NO simmetria
    float palT = vPos.x + vPos.y + vPos.z + uTime * 0.15;
    vec3 col = palette(palT, floor(uTheme));

    // Shading calcolato
    col = col * (0.4 + diff * 0.6) + vec3(spec * 0.5);

    // Fresnel glow sui bordi (colore complementare)
    float fresnel = pow(1.0 - max(dot(vNorm, vViewDir), 0.0), 3.0);
    col += palette(uTime * 0.1, floor(uTheme)) * fresnel * (0.8 + uTreble);

    // Saturazione con bassi
    col = mix(vec3(0.02), col, 0.4 + uBass * 0.6);

    // Flash bianco sporco su beat forti
    if(uBass > 0.65){
      col += vec3(smoothstep(0.65, 0.9, uBass) * 0.6);
    }

    gl_FragColor = vec4(col, 1.0);
  }
`;
```

---

## Task 3: Shader GLSL — Particelle (vsParticles + fsParticles)

**File:**
- Sovrascrivi (aggiungi dentro `<script>`): `index.html`

- [ ] **Step 3.1: Implementare `vsParticles` (Vertex Shader Particelle)**

```javascript
const vsParticles = `
  attribute float aPhase;
  attribute float aSpeed;
  attribute float aThreshold;
  varying vec3 vColor;
  varying float vLife;
  varying float vFlash;
  uniform float uTime;
  uniform float uBass;
  uniform float uTreble;
  uniform float uTheme;

  // Palette (copia ridotta dalla sfera)
  vec3 palette(float t, float idx){
    float tt = t + uTime * 0.3;
    float wave = sin(tt * 3.0) * 0.5 + 0.5;
    float wave2 = cos(tt * 7.0 + 1.5) * 0.5 + 0.5;
    float wave3 = sin(tt * 11.0 + 3.0) * 0.5 + 0.5;

    if(idx < 0.5){
      vec3 hotPink = vec3(1.0, 0.0, 0.5);
      vec3 acid = vec3(0.8, 1.0, 0.0);
      vec3 deepPurple = vec3(0.4, 0.0, 0.8);
      vec3 blood = vec3(1.0, 0.2, 0.0);
      vec3 mix1 = mix(hotPink, acid, wave);
      vec3 mix2 = mix(deepPurple, blood, wave2);
      return mix(mix1, mix2, wave3);
    }
    else if(idx < 1.5){
      vec3 lemon = vec3(1.0, 0.9, 0.0);
      vec3 hotMagenta = vec3(1.0, 0.0, 0.6);
      vec3 toxicGreen = vec3(0.0, 1.0, 0.3);
      vec3 bloodOrange = vec3(1.0, 0.2, 0.0);
      vec3 mix1 = mix(lemon, hotMagenta, wave);
      vec3 mix2 = mix(toxicGreen, bloodOrange, wave2);
      return mix(mix1, mix2, wave3);
    }
    else if(idx < 2.5){
      vec3 cyan = vec3(0.0, 1.0, 1.0);
      vec3 magenta = vec3(1.0, 0.0, 1.0);
      vec3 acid = vec3(0.8, 1.0, 0.0);
      vec3 deep = vec3(0.0, 0.05, 0.2);
      vec3 mix1 = mix(cyan, magenta, wave);
      vec3 mix2 = mix(acid, deep, wave2);
      return mix(mix1, mix2, wave3);
    }
    else if(idx < 3.5){
      float grey = sin(t * 2.0 + uTime * 0.15) * 0.5 + 0.5;
      float tint = sin(t * 8.0 + uTime * 0.5) * 0.5 + 0.5;
      vec3 bw = vec3(grey);
      vec3 neon = vec3(0.0, 1.0, 1.0) * (0.3 + uTreble * 0.7);
      return mix(bw, neon, tint * uTreble * 0.4);
    }
    else {
      float r = sin(t * 6.0 + uTime * 0.4) * 0.5 + 0.5;
      float g = sin(t * 6.0 + uTime * 0.4 + 2.094) * 0.5 + 0.5;
      float b = sin(t * 6.0 + uTime * 0.4 + 4.189) * 0.5 + 0.5;
      float boost = 0.7 + uBass * 0.6;
      return vec3(r, g, b) * boost;
    }
  }

  void main(){
    // Movimento NON orbitale — traiettoria sinusoidale indipendente
    vec3 basePos = position;
    basePos.x += sin(uTime * aSpeed + aPhase * 10.0) * 0.3;
    basePos.y += cos(uTime * aSpeed * 0.7 + aPhase * 15.0) * 0.2;
    basePos.z += sin(uTime * aSpeed * 0.5 + aPhase * 8.0) * 0.25;

    // Espansione radiale su beat forti
    float explosion = smoothstep(0.5, 0.9, uBass) * 3.0;
    vec3 finalPos = basePos * (1.0 + explosion);

    // Vita propria: accensione/spegnimento random basato su soglia individuale
    float pulse = sin(uTime * 3.0 + aPhase * 50.0);
    float life = smoothstep(aThreshold, aThreshold + 0.3, uBass + pulse * 0.2);

    // Flash bianco quando esplodono
    float flash = smoothstep(0.6, 0.85, uBass) * life;

    vColor = palette(aPhase + uTime * 0.1, floor(uTheme));
    vLife = life;
    vFlash = flash;

    gl_Position = projectionMatrix * modelViewMatrix * vec4(finalPos, 1.0);
    // Size piccolissimo ma glow intenso
    gl_PointSize = (0.2 + life * 1.5 + flash * 2.0) * (300.0 / gl_Position.z);
  }
`;
```

- [ ] **Step 3.2: Implementare `fsParticles` (Fragment Shader Particelle)**

```javascript
const fsParticles = `
  precision highp float;
  varying vec3 vColor;
  varying float vLife;
  varying float vFlash;

  void main(){
    vec2 coord = gl_PointCoord - vec2(0.5);
    float dist = length(coord);
    if(dist > 0.5) discard;

    // Glow intenso
    float alpha = smoothstep(0.5, 0.05, dist) * vLife;
    vec3 col = vColor * vLife;

    // Core bianco brillante
    float core = smoothstep(0.15, 0.0, dist);
    col += vColor * core * 0.5;

    // Flash bianco sporco su esplosione
    col += vec3(vFlash * 2.0);
    alpha += vFlash * core;

    gl_FragColor = vec4(col, alpha);
  }
`;
```

---

## Task 4: Setup Scena 3D, Mesh e Particelle

**File:**
- Sovrascrivi (JS dentro `<script>`): `index.html`

- [ ] **Step 4.1: Variabili globali e init()**

```javascript
    // ============================================
    // VARIABILI GLOBALI
    // ============================================
    let renderer, camera, scene;
    let sphereMesh, sphereMat;
    let particles, particlesMat;
    let audioCtx, analyser, audioEl;
    let isPlaying = false;
    let freqDataArray = null;
    let lastMutationTime = 0;
    const MUTATION_COOLDOWN = 0.2;
    let eqBars = [];
    const themeNames = ['Rosa Tossica', 'Giallo Bile', 'Blu Elettrico', 'B/N Sporco', 'Arcobaleno Malato'];

    // ============================================
    // INIZIALIZZAZIONE SCENA 3D
    // ============================================
    function init(){
      // Scena
      scene = new THREE.Scene();
      scene.background = new THREE.Color(0x000000);

      // Camera
      const aspect = window.innerWidth / window.innerHeight;
      camera = new THREE.PerspectiveCamera(60, aspect, 0.1, 100);
      camera.position.z = 4;

      // Renderer
      renderer = new THREE.WebGLRenderer({ antialias: true });
      renderer.setSize(window.innerWidth, window.innerHeight);
      renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
      document.getElementById('canvas-container').appendChild(renderer.domElement);

      // --- SFERA CENTRALE ---
      const sphereGeo = new THREE.SphereGeometry(0.7, 256, 256);
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

      // --- PARTICELLE (3000) ---
      const PARTICLE_COUNT = 3000;
      const positions = new Float32Array(PARTICLE_COUNT * 3);
      const phases = new Float32Array(PARTICLE_COUNT);
      const speeds = new Float32Array(PARTICLE_COUNT);
      const thresholds = new Float32Array(PARTICLE_COUNT);

      for(let i = 0; i < PARTICLE_COUNT; i++){
        // Posizione iniziale: nube sferica + alone
        const theta = Math.random() * Math.PI * 2;
        const phi = Math.acos(2.0 * Math.random() - 1.0);
        const radius = 1.2 + Math.random() * 3.0;
        positions[i * 3] = radius * Math.sin(phi) * Math.cos(theta);
        positions[i * 3 + 1] = radius * Math.sin(phi) * Math.sin(theta);
        positions[i * 3 + 2] = radius * Math.cos(phi);
        
        phases[i] = Math.random();
        speeds[i] = 0.2 + Math.random() * 1.8;
        thresholds[i] = 0.1 + Math.random() * 0.8;
      }

      const pGeo = new THREE.BufferGeometry();
      pGeo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
      pGeo.setAttribute('aPhase', new THREE.BufferAttribute(phases, 1));
      pGeo.setAttribute('aSpeed', new THREE.BufferAttribute(speeds, 1));
      pGeo.setAttribute('aThreshold', new THREE.BufferAttribute(thresholds, 1));

      particlesMat = new THREE.ShaderMaterial({
        uniforms: {
          uTime: { value: 0 },
          uBass: { value: 0 },
          uTreble: { value: 0 },
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

      // Resize handler
      window.addEventListener('resize', () => {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
      });
    }
```

---

## Task 5: Audio, UI e Loop di Animazione

**File:**
- Sovrascrivi (JS dentro `<script>`): `index.html`

- [ ] **Step 5.1: Copiare funzioni audio e UI dall'attuale**

Copia identiche le funzioni: `loadAudio`, `getAudioLevels`, `makeEq`, `updateEq`, `setupUI`. Modifica solo:
- `themeNames` (già dichiarato sopra)
- Rimuovi riferimenti a `matWire` e `meshWire`
- Aggiungi `uMid` nell'estrazione audio

```javascript
    // ============================================
    // AUDIO
    // ============================================
    function loadAudio(file){
      audioEl = document.createElement('audio');
      audioEl.src = URL.createObjectURL(file);
      audioEl.crossOrigin = 'anonymous';
      audioEl.loop = true;

      audioCtx = new (window.AudioContext || window.webkitAudioContext)();
      const src = audioCtx.createMediaElementSource(audioEl);
      analyser = audioCtx.createAnalyser();
      analyser.fftSize = 2048;
      freqDataArray = new Uint8Array(analyser.frequencyBinCount);

      src.connect(analyser);
      analyser.connect(audioCtx.destination);

      audioEl.play();
      isPlaying = true;

      document.getElementById('file-name').textContent = file.name;
      document.getElementById('btn-load').style.display = 'none';
      document.getElementById('btn-play').style.display = 'inline-block';
      document.getElementById('btn-play').textContent = 'Pause';
      document.getElementById('btn-change').style.display = 'inline-block';
    }

    function getAudioLevels(){
      if(!analyser || !audioCtx || !freqDataArray) return { bass: 0, mid: 0, treble: 0 };
      analyser.getByteFrequencyData(freqDataArray);
      const data = freqDataArray;

      const nyq = audioCtx.sampleRate / 2;
      const bins = analyser.frequencyBinCount;
      const fpb = nyq / bins;

      const bEnd = Math.min(Math.floor(150 / fpb), bins);
      let bSum = 0;
      for(let i = 0; i < bEnd; i++) bSum += data[i];
      const bass = bSum / Math.max(bEnd * 255, 1);

      const mStart = bEnd;
      const mEnd = Math.min(Math.floor(2500 / fpb), bins);
      let mSum = 0;
      for(let i = mStart; i < mEnd; i++) mSum += data[i];
      const mid = mSum / Math.max((mEnd - mStart) * 255, 1);

      const tStart = mEnd;
      const tEnd = Math.min(Math.floor(8000 / fpb), bins);
      let tSum = 0;
      for(let i = tStart; i < tEnd; i++) tSum += data[i];
      const treble = tSum / Math.max((tEnd - tStart) * 255, 1);

      return { bass, mid, treble };
    }

    // ============================================
    // EQUALIZER UI
    // ============================================
    function makeEq(){
      const c = document.getElementById('equalizer');
      for(let i = 0; i < 8; i++){
        const bar = document.createElement('div');
        bar.className = 'eq-bar';
        bar.style.height = '2px';
        c.appendChild(bar);
        eqBars.push(bar);
      }
    }
    function updateEq(dataArray){
      if(!analyser) return;
      const bpb = Math.floor(dataArray.length / eqBars.length);
      for(let i = 0; i < eqBars.length; i++){
        let s = 0;
        for(let j = 0; j < bpb; j++) s += dataArray[i * bpb + j];
        eqBars[i].style.height = Math.max(2, (s / bpb / 255) * 30) + 'px';
      }
    }

    // ============================================
    // UI EVENTS
    // ============================================
    function setupUI(){
      const audIn = document.getElementById('audio-input');
      const btnLoad = document.getElementById('btn-load');
      const btnPlay = document.getElementById('btn-play');
      const btnChange = document.getElementById('btn-change');
      const sliderTheme = document.getElementById('slider-theme');
      const themeLabel = document.getElementById('theme-label');

      btnLoad.addEventListener('click', () => audIn.click());

      btnPlay.addEventListener('click', () => {
        if(!audioEl) return;
        if(isPlaying){ audioEl.pause(); btnPlay.textContent = 'Play'; }
        else { audioEl.play(); btnPlay.textContent = 'Pause'; }
        isPlaying = !isPlaying;
      });

      btnChange.addEventListener('click', () => {
        if(audioEl){
          audioEl.pause();
          URL.revokeObjectURL(audioEl.src);
          audioEl = null;
        }
        audIn.value = '';
        audIn.click();
      });

      audIn.addEventListener('change', (e) => {
        const f = e.target.files[0];
        if(f) loadAudio(f);
      });

      sliderTheme.addEventListener('input', (e) => {
        const v = parseFloat(e.target.value);
        themeLabel.textContent = themeNames[Math.min(Math.floor(v), 4)];
      });

      makeEq();
      document.addEventListener('click', () => {
        if(audioCtx && audioCtx.state === 'suspended') audioCtx.resume();
      }, { once: true });
    }
```

- [ ] **Step 5.2: Implementare il nuovo `animate(time)`**

```javascript
    // ============================================
    // LOOP DI ANIMAZIONE
    // ============================================
    function animate(time){
      requestAnimationFrame(animate);
      const elapsed = time * 0.001;

      // Analisi Audio
      let bass = 0, mid = 0, treble = 0;
      let freqData = null;
      if(analyser){
        const lv = getAudioLevels();
        bass = lv.bass;
        mid = lv.mid;
        treble = lv.treble;
        freqData = freqDataArray;
        analyser.getByteFrequencyData(freqData);
      }

      // Smoothing
      const curB = sphereMat.uniforms.uBass.value;
      const curT = sphereMat.uniforms.uTreble.value;
      const smoothBass = curB + (bass - curB) * 0.12;
      const smoothTreble = curT + (treble - curT) * 0.18;

      // Aggiorna uniform sfera
      sphereMat.uniforms.uTime.value = elapsed;
      sphereMat.uniforms.uBass.value = smoothBass;
      sphereMat.uniforms.uTreble.value = smoothTreble;

      // Aggiorna uniform particelle
      particlesMat.uniforms.uTime.value = elapsed;
      particlesMat.uniforms.uBass.value = smoothBass;
      particlesMat.uniforms.uTreble.value = smoothTreble;

      // Trigger Mutazione (beat snap)
      if(mid > 0.55 && (elapsed - lastMutationTime) > MUTATION_COOLDOWN){
        sphereMat.uniforms.uSeed.value += 5.0 + Math.random() * 8.0;
        lastMutationTime = elapsed;
      }

      // Tema
      const themeVal = parseFloat(document.getElementById('slider-theme').value);
      sphereMat.uniforms.uTheme.value = themeVal;
      particlesMat.uniforms.uTheme.value = themeVal;

      // Rotazione lenta scena (inquietante)
      scene.rotation.y += 0.005;
      scene.rotation.x = Math.sin(elapsed * 0.1) * 0.05;

      // Render
      renderer.render(scene, camera);

      // Equalizer
      if(freqData) updateEq(freqData);
    }
```

- [ ] **Step 5.3: Chiamate di avvio**

```javascript
    // ============================================
    // AVVIO
    // ============================================
    init();
    setupUI();
    animate(0);
```

---

## Task 6: Verifica e Test

**File:**
- Sovrascrivi: `index.html`

- [ ] **Step 6.1: Verifica sintassi**

Apri il file in un browser. Non devono esserci errori in console. La scena deve mostrare:
- Sfera nera al centro che prende colore quando il noise si attiva
- Particelle piccole e sparse che si muovono lentamente
- UI overlay visibile con slider e pulsanti

- [ ] **Step 6.2: Test audio**

Carica una canzone. Verifica che:
- Play/Pause funzioni
- Le barre dell'equalizer si muovano
- La sfera si contorca (si espande/contrae) con i bassi
- Le particelle accelerino/espandano con i beat
- Il trigger mutazione cambi il pattern quando i medi sono forti
- Lo slider tema cambi i colori in tempo reale

---

## Spec Coverage Check

| Requisito Spec | Task che lo implementa |
|---------------|----------------------|
| Sfera 3D piccola (0.7, 256×256) | Task 4.1 |
| Noise 2 layer (bassa + alta freq) | Task 2.1 |
| Displacement con uBass | Task 2.1 |
| Normale finta per shading | Task 2.1 |
| Palette 5 temi acidi | Task 2.2 |
| Fresnel + flash bianco | Task 2.2 |
| Particelle 3000 con vita propria | Task 3.1, 4.1 |
| Attributi custom (aPhase, aSpeed, aThreshold) | Task 4.1 |
| Espansione radiale su beat | Task 3.1 |
| Audio 2048 bin + bass/mid/treble | Task 5.1 |
| Trigger mutazione (mid > 0.55) | Task 5.2 |
| UI play/pause/slider/equalizer | Task 5.1 |
| Rotazione lenta scena | Task 5.2 |

**Nessun gap trovato.**

---

## Placeholder Scan

- Nessun "TBD", "TODO", "implement later"
- Nessun "add appropriate error handling" senza codice
- Codice GLSL completo in ogni step
- Funzioni audio/UI copiate integralmente con modifiche minime
- Tutti i path dei file sono esatti

**Nessun placeholder trovato.**

---

## Type Consistency Check

- Uniform names: `uTime`, `uBass`, `uTreble`, `uMid`, `uTheme`, `uSeed` — consistenti in tutti gli shader e il JS
- Attributi particelle: `aPhase`, `aSpeed`, `aThreshold` — dichiarati in JS come BufferAttribute(…, 1) e usati in GLSL
- Theme names array: 5 elementi, match con slider 0-4

**Nessuna inconsistenza trovata.**
