# Design: Bosco Incantato — Sistema di Strati Organici AI Art per Nature Visual

**Data:** 2025-06-12  
**Progetto:** Music Visual — Branch nature-visual  
**File principale:** `index.html` (backup: `funziona.html`)  
**Approccio:** 1 — Sistema a strati multipli (Nuvole + Radici + Muschio) con mixer finale

---

## 1. Obiettivo

Trasformare il visualizer in un ecosistema visivo stratificato, composto da 3 strati indipendenti che simulano fenomeni naturali organici (nuvole, radici, muschio). Ogni strato ha il proprio comportamento, il proprio shader e la propria reattività audio. Un mixer finale unisce tutti gli strati con additive blending. L'utente controlla l'intensità di ogni strato tramite 3 slider dedicati.

**Metafora visiva:** Come guardare dentro un bosco incantato attraverso un vetro — vedi nuvole che fluttuano sopra, radici luminose che crescono dal centro, e muschio che si espande ai bordi, tutto animato dal vento della musica.

---

## 2. Architettura Render

Il rendering passa da 1 strato (attuale) a **5 passi** per frame:

```
Passo 1: Render Scena 3D (sfera + particelle) → rt_scene
Passo 2: Render Nuvole (domain warping + FBM, ping-pong A↔B) → rt_nuvole
Passo 3: Render Radici (procedural branching + trail, ping-pong A↔B) → rt_radici
Passo 4: Render Muschio (edge-growth + fresnel, ping-pong A↔B) → rt_muschio
Passo 5: MIXER FINALE: combina rt_scene + rt_nuvole + rt_radici + rt_muschio → schermo
```

**Ottimizzazione:** Se uno slider è a 0, il passo corrispondente viene saltato. Se tutti e 3 i nuovi slider sono a 0, il comportamento è identico al visual originale (solo scena 3D + scie fluide).

---

## 3. Strato Esistente — Scena 3D + Scie Fluide (Invariato)

Il sistema attuale di ping-pong feedback (rtA ↔ rtB) con zoom, blur e fade rimane **inalterato**. Continua a funzionare come base visiva. Le modifiche precedenti al file (reaction-diffusion) saranno **rimosse/sostituite** con il nuovo sistema a strati.

---

## 4. Strato 1 — NUVOLE (Fumi Colorati)

### 4.1 Concetto visivo

Massa di fumo/fluido colorato che fluttua e si attorciglia lentamente, simile a nuvole temporalesche illuminate dal tramonto o inchiostro che si espande in acqua.

### 4.2 Tecnica — Domain Warping + FBM

Invece di reaction-diffusion scientifico, usiamo **domain warping** (torcere le coordinate dello spazio multiple volte) combinato con **Fractal Brownian Motion** (somma di multiple ottave di noise).

**Vertex shader:** Pass-through (identico a vsFeedback esistente).

**Fragment shader — fsNuvole:**

```glsl
precision highp float;
uniform sampler2D tPrev;
uniform float uTime;
uniform float uBass;
uniform float uTheme;
uniform float uIntensity;
varying vec2 vUv;

// Funzioni noise standard (Simplex 3D)
// ... (stesse funzioni noise del vsMain esistente)

// Domain Warping: torsione multipla delle coordinate
vec2 warp(vec2 p, float t){
  float n1=snoise(vec3(p*2.0,t*0.3));
  float n2=snoise(vec3(p*3.0+50.0,t*0.5));
  return p+vec2(n1,n2)*0.15;
}

// FBM: somma di noise a frequenze crescenti
float fbm(vec2 p, float t){
  float val=0.0;
  float amp=0.5;
  for(int i=0;i<5;i++){
    val+=amp*snoise(vec3(p,t*0.2));
    p*=2.0;
    amp*=0.5;
  }
  return val;
}

vec3 paletteNuvole(float t, float idx){
  // Palette morbide e fluide per ogni tema
  if(idx<0.5){      // Rosa → rosa/nebbia viola
    return mix(vec3(1.0,0.2,0.6),vec3(0.4,0.1,0.6),sin(t*3.0)*0.5+0.5);
  }else if(idx<1.5){ // Giallo → oro/arancio caldo
    return mix(vec3(1.0,0.8,0.2),vec3(1.0,0.3,0.1),sin(t*2.5)*0.5+0.5);
  }else if(idx<2.5){ // Blu → ciano/oceano
    return mix(vec3(0.0,0.5,0.9),vec3(0.0,0.9,0.7),sin(t*3.5)*0.5+0.5);
  }else if(idx<3.5){ // B/N → grigio/nebbia argento
    return mix(vec3(0.7,0.7,0.8),vec3(0.3,0.3,0.4),sin(t*2.0)*0.5+0.5);
  }else{             // Arcobaleno → tutti i colori sfumati
    return vec3(sin(t*5.0)*0.5+0.5,sin(t*5.0+2.1)*0.5+0.5,sin(t*5.0+4.2)*0.5+0.5);
  }
}

void main(){
  vec2 uv=vUv;
  float t=uTime;

  // Domain warping: torsione delle coordinate
  vec2 p=uv*3.0;
  p=warp(p,t);
  p=warp(p,t*0.7+100.0);

  // FBM per forma nuvolosa
  float n=fbm(p,t);
  float n2=fbm(p*1.5+vec2(50.0),t*0.8);

  // Bassi espandono e densificano
  float densita=0.3+uBass*0.7;
  float forma=smoothstep(0.3-densita*0.2,0.7+densita*0.3,n);

  // Colore tematico + variabilità
  float tIdx=floor(uTheme);
  float tMix=fract(uTheme);
  vec3 c0=paletteNuvole(n2+uBass,tIdx);
  vec3 c1=paletteNuvole(n2+uBass,min(tIdx+1.0,4.0));
  vec3 col=mix(c0,c1,tMix);

  // Fade leggero per scie morbide
  vec4 prev=texture2D(tPrev,uv);
  vec3 nuovo=col*forma*0.15; // bassa intensità per non coprire

  // Accumulo con fade
  vec3 finale=prev.rgb*0.95+nuovo;

  gl_FragColor=vec4(finale,1.0);
}
```

### 4.3 Reattività audio

| Parametro | Effetto audio | Effetto visivo |
|---|---|---|
| `uBass` | Bassi forti | Nuvole più dense e spesse, espandono il loro raggio |
| `uTime` | Scorre sempre | Movimento continuo, anche in silenzio |
| `uTheme` | Palette attiva | Cambia colore base delle nuvole |

### 4.4 Ping-pong setup

Due render target dedicati: `rtNuvoleA`, `rtNuvoleB`. Ogni frame legge da uno e scrive nell'altro.

---

## 5. Strato 2 — RADICI (Filamenti Luminosi)

### 5.1 Concetto visivo

Linee sottili luminose che germogliano dal centro dello schermo e si ramificano verso l'esterno, come radici in time-lapse o connessioni neurali. Ogni beat genera nuovi germogli.

### 5.2 Tecnica — Trail Accumulation con Procedural Branching

**Fragment shader — fsRadici:**

```glsl
precision highp float;
uniform sampler2D tPrev;
uniform float uTime;
uniform float uBeat;     // trigger da bassi > soglia
uniform float uTreble;
uniform float uTheme;
uniform float uIntensity;
varying vec2 vUv;

// Hash per random
float hash(vec2 p){return fract(sin(dot(p,vec2(127.1,311.7)))*43758.5453);}

vec3 paletteRadici(float t, float idx){
  if(idx<0.5) return vec3(1.0,0.4,0.7);      // Rosa acceso
  else if(idx<1.5) return vec3(1.0,0.8,0.0);  // Oro
  else if(idx<2.5) return vec3(0.0,0.8,1.0); // Ciano
  else if(idx<3.5) return vec3(0.9,0.9,0.9);  // Bianco
  else return vec3(0.8,0.2,1.0);              // Viola
}

void main(){
  vec2 uv=vUv;
  vec4 prev=texture2D(tPrev,uv);

  // Coordinate dal centro
  vec2 center=uv-0.5;
  float dist=length(center);
  float angle=atan(center.y,center.x);

  // Generazione nuovi filamenti su beat
  float spawn=0.0;
  if(uBeat>0.6){ // soglia bassi
    // Crea un seme random basato su tempo e posizione
    float seed=hash(vec2(floor(uTime*10.0),0.0));
    float angSeed=seed*6.28318;
    float distSeed=0.02+hash(vec2(seed,1.0))*0.15;

    // Vicino alla linea del seme?
    float lineDist=abs(fract((angle-angSeed)/6.28318+0.5)-0.5);
    spawn=smoothstep(0.05+hash(vec2(seed,2.0))*0.1,0.0,lineDist)*smoothstep(0.02,0.0,abs(dist-distSeed));
  }

  // Espansione dei filamenti esistenti
  float grow=0.0;
  // Leggi vicini per diffusione
  float texel=1.0/256.0;
  float n=texture2D(tPrev,uv+vec2(0.0,texel)).r;
  float s=texture2D(tPrev,uv+vec2(0.0,-texel)).r;
  float e=texture2D(tPrev,uv+vec2(texel,0.0)).r;
  float w=texture2D(tPrev,uv+vec2(-texel,0.0)).r;
  grow=(n+s+e+w)*0.25;

  // Treble aggiunge "nervosismo" — filamenti più frastagliati
  float noise=hash(uv*100.0+uTime*50.0);
  grow+=noise*uTreble*0.3;

  // Colore
  float tIdx=floor(uTheme);
  vec3 col=paletteRadici(uTime*0.5+uTreble,tIdx);

  // Combinazione: spawn + grow + fade
  float newBright=clamp(spawn*0.8+grow*0.6,0.0,1.0);
  vec3 nuovo=col*newBright;

  // Accumulo: fade lento per scie lunghe
  vec3 finale=prev.rgb*0.90+nuovo;

  gl_FragColor=vec4(finale,1.0);
}
```

### 5.3 Reattività audio

| Parametro | Effetto audio | Effetto visivo |
|---|---|---|
| `uBeat` | Bassi > soglia | Genera nuovo germoglio dal centro |
| `uTreble` | Alti forti | Filamenti più frastagliati e nervosi |
| `uTheme` | Palette attiva | Cambia colore dei filamenti |

### 5.4 Ping-pong setup

Due render target dedicati: `rtRadiciA`, `rtRadiciB`.

---

## 6. Strato 3 — MUSCHIO (Macchie ai Bordi)

### 6.1 Concetto visivo

Macchie organiche che crescono dai bordi dello schermo verso il centro, come lichene su pietra. I bordi delle macchie hanno un effetto iridescente (cambiano colore come bolle di sapone).

### 6.2 Tecnica — Edge-Growth + Fresnel Iridescente

**Fragment shader — fsMuschio:**

```glsl
precision highp float;
uniform sampler2D tPrev;
uniform float uTime;
uniform float uTreble;
uniform float uVolume;
uniform float uTheme;
uniform float uIntensity;
varying vec2 vUv;

float hash(vec2 p){return fract(sin(dot(p,vec2(127.1,311.7)))*43758.5453);}

void main(){
  vec2 uv=vUv;
  vec4 prev=texture2D(tPrev,uv);

  // Distanza dai bordi (0 = bordo, 1 = centro)
  float edgeDist=min(min(uv.x,1.0-uv.x),min(uv.y,1.0-uv.y));

  // Il muschio cresce dal bordo verso il centro
  // Treble controlla velocità: più alti = cresce più veloce
  float growth=0.02+uTreble*0.03;

  // Soglia di crescita: i pixel vicini al bordo "germogliano"
  float seed=hash(vec2(floor(uv.x*50.0),floor(uv.y*50.0))+uTime*0.1);
  float threshold=0.7-uTreble*0.3; // più treble = soglia più bassa = più facile germogliare

  // Campiona vicini per diffusione morbida
  float texel=1.0/256.0;
  float n=texture2D(tPrev,uv+vec2(0.0,texel)).g;
  float s=texture2D(tPrev,uv+vec2(0.0,-texel)).g;
  float e=texture2D(tPrev,uv+vec2(texel,0.0)).g;
  float w=texture2D(tPrev,uv+vec2(-texel,0.0)).g;
  float avg=(n+s+e+w)*0.25;

  // Nuova crescita ai bordi
  float newGrowth=0.0;
  if(edgeDist<0.15 && seed>threshold){
    newGrowth=0.5; // germoglio
  }
  // Oppure vicino a muschio esistente
  if(avg>0.1 && hash(uv*100.0+uTime)<avg*growth*10.0){
    newGrowth=max(newGrowth,avg*0.8);
  }

  // Iridescenza ai bordi: colore cambia con angolo e tempo
  float edgeIrid=0.0;
  if(newGrowth>0.1 || avg>0.1){
    edgeIrid=sin(uTime*3.0+edgeDist*20.0)*0.5+0.5;
  }

  // Colore del muschio
  float tIdx=floor(uTheme);
  vec3 baseCol;
  if(tIdx<0.5) baseCol=vec3(0.6,0.2,0.4);      // Rosa muschio
  else if(tIdx<1.5) baseCol=vec3(0.7,0.6,0.1);  // Oro
  else if(tIdx<2.5) baseCol=vec3(0.1,0.5,0.6);  // Ciano
  else if(tIdx<3.5) baseCol=vec3(0.5,0.5,0.5);  // Grigio
  else baseCol=vec3(0.5,0.2,0.8);                 // Viola

  // Iridescenza = shift arcobaleno ai bordi
  vec3 iridCol=vec3(
    sin(uTime*2.0+edgeDist*15.0)*0.5+0.5,
    sin(uTime*2.0+edgeDist*15.0+2.1)*0.5+0.5,
    sin(uTime*2.0+edgeDist*15.0+4.2)*0.5+0.5
  );

  vec3 col=mix(baseCol,iridCol,edgeIrid*uVolume);

  // Accumulo
  float muschio=clamp(newGrowth+avg*0.95,0.0,1.0);
  vec3 finale=prev.rgb*0.92+col*muschio*0.15;

  gl_FragColor=vec4(finale,1.0);
}
```

### 6.3 Reattività audio

| Parametro | Effetto audio | Effetto visivo |
|---|---|---|
| `uTreble` | Alti/acuti | Velocità di crescita del muschio, più facile germogliare |
| `uVolume` | Volume generale | Intensità dell'iridescenza ai bordi |
| `uTheme` | Palette attiva | Colore base del muschio |

### 6.4 Ping-pong setup

Due render target dedicati: `rtMuschioA`, `rtMuschioB`.

---

## 7. Shader Mixer Finale

**Fragment shader — fsMixer:**

```glsl
precision mediump float;
uniform sampler2D tScene;
uniform sampler2D tNuvole;
uniform sampler2D tRadici;
uniform sampler2D tMuschio;
uniform float uNuvole;
uniform float uRadici;
uniform float uMuschio;
uniform float uBass;
varying vec2 vUv;

void main(){
  vec3 scene=texture2D(tScene,vUv).rgb;
  vec3 nuvole=texture2D(tNuvole,vUv).rgb;
  vec3 radici=texture2D(tRadici,vUv).rgb;
  vec3 muschio=texture2D(tMuschio,vUv).rgb;

  // Additive blending con controllo slider
  vec3 finale=scene;
  finale+=nuvole*uNuvole*(0.5+uBass*0.5);
  finale+=radici*uRadici;
  finale+=muschio*uMuschio;

  // Vignetta leggera per focus centrale
  float vig=1.0-length(vUv-0.5)*0.3;
  finale*=vig;

  // Clamp per evitare over-bright
  finale=clamp(finale,0.0,1.0);

  gl_FragColor=vec4(finale,1.0);
}
```

---

## 8. UI Aggiornata

Sostituzione dello slider "Organicità" con **3 slider separati**:

```html
<div class="slider-row">
  <label>Tema</label>
  <input type="range" id="slider-theme" min="0" max="4" step="0.01" value="0">
  <span id="theme-label">Rosa</span>
</div>
<div class="slider-row">
  <label>Nuvole</label>
  <input type="range" id="slider-nuvole" min="0" max="1" step="0.01" value="0">
  <span id="nuvole-label">0%</span>
</div>
<div class="slider-row">
  <label>Radici</label>
  <input type="range" id="slider-radici" min="0" max="1" step="0.01" value="0">
  <span id="radici-label">0%</span>
</div>
<div class="slider-row">
  <label>Muschio</label>
  <input type="range" id="slider-muschio" min="0" max="1" step="0.01" value="0">
  <span id="muschio-label">0%</span>
</div>
```

---

## 9. Loop di Animazione Aggiornato

```javascript
function animate(time){
  requestAnimationFrame(animate);
  const elapsed=time*0.001;

  // Audio (come ora)
  let a={bass:0,treble:0,beat:0};
  let freqData=null;
  if(analyser){
    a=getAudio();
    freqData=new Uint8Array(analyser.frequencyBinCount);
    analyser.getByteFrequencyData(freqData);
    // Beat detection: bassi > soglia
    a.beat=a.bass>0.5?a.bass:0;
  }

  // ... smoothing sfera, particelle, rotazione (tutti come ora)

  // Leggi slider
  const nuvoleVal=parseFloat(document.getElementById('slider-nuvole').value);
  const radiciVal=parseFloat(document.getElementById('slider-radici').value);
  const muschioVal=parseFloat(document.getElementById('slider-muschio').value);

  // ============================================
  // PASSO 1: Scie fluide + scena 3D (rtA → rtB)
  // ============================================
  // ... (come ora)

  // ============================================
  // PASSO 2: Nuvole (se attivo)
  // ============================================
  if(nuvoleVal>0){
    nuvoleMat.uniforms.tPrev.value=rtNuvoleA.texture;
    nuvoleMat.uniforms.uTime.value=elapsed;
    nuvoleMat.uniforms.uBass.value=bass;
    nuvoleMat.uniforms.uTheme.value=themeVal;
    renderer.setRenderTarget(rtNuvoleB);
    renderer.render(nuvoleScene,orthoCamera);
    // swap
    let tmp=rtNuvoleA; rtNuvoleA=rtNuvoleB; rtNuvoleB=tmp;
  }

  // ============================================
  // PASSO 3: Radici (se attivo)
  // ============================================
  if(radiciVal>0){
    radiciMat.uniforms.tPrev.value=rtRadiciA.texture;
    radiciMat.uniforms.uTime.value=elapsed;
    radiciMat.uniforms.uBeat.value=a.beat;
    radiciMat.uniforms.uTreble.value=treble;
    radiciMat.uniforms.uTheme.value=themeVal;
    renderer.setRenderTarget(rtRadiciB);
    renderer.render(radiciScene,orthoCamera);
    let tmp=rtRadiciA; rtRadiciA=rtRadiciB; rtRadiciB=tmp;
  }

  // ============================================
  // PASSO 4: Muschio (se attivo)
  // ============================================
  if(muschioVal>0){
    muschioMat.uniforms.tPrev.value=rtMuschioA.texture;
    muschioMat.uniforms.uTime.value=elapsed;
    muschioMat.uniforms.uTreble.value=treble;
    muschioMat.uniforms.uVolume.value=(bass+treble)*0.5;
    muschioMat.uniforms.uTheme.value=themeVal;
    renderer.setRenderTarget(rtMuschioB);
    renderer.render(muschioScene,orthoCamera);
    let tmp=rtMuschioA; rtMuschioA=rtMuschioB; rtMuschioB=tmp;
  }

  // ============================================
  // PASSO 5: Mixer finale
  // ============================================
  mixerMat.uniforms.tScene.value=rtB.texture; // rtB contiene scie+3D
  mixerMat.uniforms.tNuvole.value=rtNuvoleA.texture;
  mixerMat.uniforms.tRadici.value=rtRadiciA.texture;
  mixerMat.uniforms.tMuschio.value=rtMuschioA.texture;
  mixerMat.uniforms.uNuvole.value=nuvoleVal;
  mixerMat.uniforms.uRadici.value=radiciVal;
  mixerMat.uniforms.uMuschio.value=muschioVal;
  mixerMat.uniforms.uBass.value=bass;
  renderer.setRenderTarget(null);
  renderer.render(mixerScene,orthoCamera);

  // Swap scie ping-pong
  const temp=rtA; rtA=rtB; rtB=temp;

  if(freqData) updateEq(freqData);
}
```

---

## 10. Criteri di Successo

- [ ] Tutti e 3 gli slider appaiono nella UI e funzionano
- [ ] Quando tutti gli slider sono a 0, il visual è IDENTICO a `funziona.html`
- [ ] Nuvole: masse fluide, oniriche, che fluttuano e si attorcigliano
- [ ] Radici: filamenti che germogliano dal centro sui beat
- [ ] Muschio: macchie che crescono dai bordi con bordi iridescenti
- [ ] I colori cambiano con il tema attivo (tutti e 3 gli strati)
- [ ] Nessun calo FPS significativo quando tutti gli strati sono attivi
- [ ] Il mixer non produce artefatti visivi (banding, flickering)

---

## 11. Ottimizzazioni e Note

1. **Skip condizionale:** Se uno slider è 0, il suo passo viene saltato completamente
2. **Risoluzione ridotta:** Gli strati RD possono essere a metà risoluzione (w/2, h/2) se la performance lo richiede
3. **Init state:** Ogni strato inizializza il proprio ping-pong con canvas nero (o seme appropriato) in `init()`
4. **Cleanup:** Lo shader `fsRD` e `fsRDComposite` precedenti vengono **rimossi** e sostituiti dai nuovi shader `fsNuvole`, `fsRadici`, `fsMuschio`, `fsMixer`

---

*Design approvato dall'utente. Pronto per implementazione.*
