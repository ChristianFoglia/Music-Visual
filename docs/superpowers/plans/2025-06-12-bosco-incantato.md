# Bosco Incantato — Sistema a Strati Organici Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Trasformare il visualizer aggiungendo 3 strati indipendenti (Nuvole, Radici, Muschio) con shader artistici che simulano fenomeni naturali, controllabili da 3 slider dedicati.

**Architecture:** Ogni strato ha il proprio ping-pong render target (A↔B) e il proprio shader custom. Un mixer finale unisce la scena 3D con tutti gli strati attivi via additive blending. Gli strati possono essere saltati quando il loro slider è a 0.

**Tech Stack:** Three.js (WebGLRenderer, ShaderMaterial, WebGLRenderTarget), Web Audio API (AnalyseNode esistente), HTML/CSS vanilla.

---

## File Structure

| File | Azione | Responsabilità |
|---|---|---|
| `C:\Users\Christian portatile\Documents\GitHub\Music Visual\index.html` | **Modifica** | Sostituisce vecchio RD con nuovo sistema a 3 strati. Aggiunge shader, scene, variabili, UI, loop aggiornato |
| `C:\Users\Christian portatile\Documents\GitHub\Music Visual\funziona.html` | Invariato | Backup del visual originale |
| Browser | **Test visivo** | Verifica look di ogni strato, performance, nessun artefatto |

---

## Task 1: Pulire Vecchio Reaction-Diffusion

**File:**
- Modifica: `C:\Users\Christian portatile\Documents\GitHub\Music Visual\index.html`

- [ ] **Step 1.1: Rimuovere vecchie variabili RD**

Sostituire:
```javascript
    // --- REACTION-DIFFUSION ---
    let rdA, rdB;
    let rdSimScene, rdCompositeScene;
    let rdSimMat, rdCompositeMat;
```

Con:
```javascript
    // --- STRATI ORGANICI ---
    let rtNuvoleA, rtNuvoleB;
    let rtRadiciA, rtRadiciB;
    let rtMuschioA, rtMuschioB;
    let nuvoleScene, radiciScene, muschioScene, mixerScene;
    let nuvoleMat, radiciMat, muschioMat, mixerMat;
```

- [ ] **Step 1.2: Rimuovere vecchi shader fsRD e fsRDComposite**

Trovare e rimuovere questi blocchi dallo script (erano aggiunti dopo fsDisplay).

**NOTA:** Se non esistono ancora (perché il file attuale potrebbe essere il backup), saltare.

- [ ] **Step 1.3: Rimuovere vecchi render target e scene RD da init()**

Nel blocco init(), trovare e rimuovere il blocco `// --- RD RENDER TARGETS ---` e tutto ciò che segue fino a prima di `window.addEventListener('resize',onResize);`.

**Verifica:** Il file deve compilare senza errori dopo questa rimozione. Nessun riferimento a `rdA`, `rdB`, `rdSimMat`, `rdCompositeMat` rimasto.

- [ ] **Step 1.4: Rimuovere vecchio loop RD dall'animate()**

Sostituire il blocco RD nel loop animate (se presente) con il nuovo sistema.

---

## Task 2: Aggiungere UI con 3 Slider

**File:**
- Modifica: `C:\Users\Christian portatile\Documents\GitHub\Music Visual\index.html:33-42`

- [ ] **Step 2.1: Sostituire slider "Organicità" con 3 slider**

Trovare il blocco HTML del vecchio slider Organicità (o aggiungere dopo Tema):

```html
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

- [ ] **Step 2.2: Aggiornare setupUI() con i 3 listener**

Trovare e rimuovere il listener del vecchio slider-organic e sostituire con:

```javascript
      const sldNuvole=document.getElementById('slider-nuvole');
      const lblNuvole=document.getElementById('nuvole-label');
      sldNuvole.addEventListener('input',(e)=>{
        const v=parseFloat(e.target.value);
        lblNuvole.textContent=Math.round(v*100)+'%';
      });

      const sldRadici=document.getElementById('slider-radici');
      const lblRadici=document.getElementById('radici-label');
      sldRadici.addEventListener('input',(e)=>{
        const v=parseFloat(e.target.value);
        lblRadici.textContent=Math.round(v*100)+'%';
      });

      const sldMuschio=document.getElementById('slider-muschio');
      const lblMuschio=document.getElementById('muschio-label');
      sldMuschio.addEventListener('input',(e)=>{
        const v=parseFloat(e.target.value);
        lblMuschio.textContent=Math.round(v*100)+'%';
      });
```

- [ ] **Step 2.3: Verifica visiva**

Aprire `index.html` nel browser. Verificare che appaiano 3 nuove righe sotto "Tema". Muovere ogni slider: i numeri devono cambiare da 0% a 100%.

- [ ] **Step 2.4: Commit**

```bash
cd "C:\Users\Christian portatile\Documents\GitHub\Music Visual"
git add index.html
git commit -m "ui: replace organic slider with nuvole/radici/muschio controls"
```

---

## Task 3: Setup Variabili Globali e Render Target

**File:**
- Modifica: `C:\Users\Christian portatile\Documents\GitHub\Music Visual\index.html`

- [ ] **Step 3.1: Aggiornare dichiarazione variabili globali**

Sostituire (o aggiungere se mancante):
```javascript
    // --- STRATI ORGANICI ---
    let rtNuvoleA, rtNuvoleB;
    let rtRadiciA, rtRadiciB;
    let rtMuschioA, rtMuschioB;
    let nuvoleScene, radiciScene, muschioScene, mixerScene;
    let nuvoleMat, radiciMat, muschioMat, mixerMat;
```

- [ ] **Step 3.2: Creare render target e scene in init()**

Dopo la creazione di `displayScene` (o dove era il vecchio blocco RD), aggiungere:

```javascript
      // --- NUVOLE SETUP ---
      rtNuvoleA=new THREE.WebGLRenderTarget(w,h,rtOpts);
      rtNuvoleB=new THREE.WebGLRenderTarget(w,h,rtOpts);
      nuvoleScene=new THREE.Scene();
      nuvoleMat=new THREE.ShaderMaterial({
        uniforms:{tPrev:{value:null},uTime:{value:0},uBass:{value:0},uTheme:{value:0}},
        vertexShader:vsFeedback,fragmentShader:fsNuvole,
        depthWrite:false
      });
      nuvoleScene.add(new THREE.Mesh(planeGeo,nuvoleMat));

      // --- RADICI SETUP ---
      rtRadiciA=new THREE.WebGLRenderTarget(w,h,rtOpts);
      rtRadiciB=new THREE.WebGLRenderTarget(w,h,rtOpts);
      radiciScene=new THREE.Scene();
      radiciMat=new THREE.ShaderMaterial({
        uniforms:{tPrev:{value:null},uTime:{value:0},uBeat:{value:0},uTreble:{value:0},uTheme:{value:0}},
        vertexShader:vsFeedback,fragmentShader:fsRadici,
        depthWrite:false
      });
      radiciScene.add(new THREE.Mesh(planeGeo,radiciMat));

      // --- MUSCHIO SETUP ---
      rtMuschioA=new THREE.WebGLRenderTarget(w,h,rtOpts);
      rtMuschioB=new THREE.WebGLRenderTarget(w,h,rtOpts);
      muschioScene=new THREE.Scene();
      muschioMat=new THREE.ShaderMaterial({
        uniforms:{tPrev:{value:null},uTime:{value:0},uTreble:{value:0},uVolume:{value:0},uTheme:{value:0}},
        vertexShader:vsFeedback,fragmentShader:fsMuschio,
        depthWrite:false
      });
      muschioScene.add(new THREE.Mesh(planeGeo,muschioMat));

      // --- MIXER FINALE ---
      mixerScene=new THREE.Scene();
      mixerMat=new THREE.ShaderMaterial({
        uniforms:{
          tScene:{value:null},tNuvole:{value:null},tRadici:{value:null},tMuschio:{value:null},
          uNuvole:{value:0},uRadici:{value:0},uMuschio:{value:0},uBass:{value:0}
        },
        vertexShader:vsDisplay,fragmentShader:fsMixer,
        depthWrite:false
      });
      mixerScene.add(new THREE.Mesh(planeGeo,mixerMat));
```

- [ ] **Step 3.3: Aggiornare onResize**

Aggiungere:
```javascript
      rtNuvoleA.setSize(w,h); rtNuvoleB.setSize(w,h);
      rtRadiciA.setSize(w,h); rtRadiciB.setSize(w,h);
      rtMuschioA.setSize(w,h); rtMuschioB.setSize(w,h);
```

- [ ] **Step 3.4: Verifica**

Aprire console DevTools: nessun errore. `rtNuvoleA`, `rtRadiciA`, `rtMuschioA` devono esistere come WebGLRenderTarget.

- [ ] **Step 3.5: Commit**

```bash
git add index.html
git commit -m "feat: add render targets and scenes for nuvole/radici/muschio layers"
```

---

## Task 4: Scrivere Shader Nuvole

**File:**
- Modifica: `C:\Users\Christian portatile\Documents\GitHub\Music Visual\index.html`

- [ ] **Step 4.1: Aggiungere shader fsNuvole**

Dopo `fsDisplay` (o dove c'erano i vecchi shader RD), aggiungere:

```javascript
    const fsNuvole = `
      precision highp float;
      uniform sampler2D tPrev;
      uniform float uTime;
      uniform float uBass;
      uniform float uTheme;
      varying vec2 vUv;

      // Simplex Noise 3D (stesso codice del vsMain)
      vec3 mod289(vec3 x){return x-floor(x*(1.0/289.0))*289.0;}
      vec4 mod289(vec4 x){return x-floor(x*(1.0/289.0))*289.0;}
      vec4 permute(vec4 x){return mod289(((x*34.0)+1.0)*x);}
      vec4 taylorInvSqrt(vec4 r){return 1.79284291400159-0.85373472095314*r;}
      float snoise(vec3 v){
        const vec2 C=vec2(1.0/6.0,1.0/3.0);
        const vec4 D=vec4(0.0,0.5,1.0,2.0);
        vec3 i=floor(v+dot(v,C.yyy));
        vec3 x0=v-i+dot(i,C.xxx);
        vec3 g=step(x0.yzx,x0.xyz);
        vec3 l=1.0-g;
        vec3 i1=min(g.xyz,l.zxy);
        vec3 i2=max(g.xyz,l.zxy);
        vec3 x1=x0-i1+C.xxx;
        vec3 x2=x0-i2+C.yyy;
        vec3 x3=x0-D.yyy;
        i=mod289(i);
        vec4 p=permute(permute(permute(i.z+vec4(0.0,i1.z,i2.z,1.0))+i.y+vec4(0.0,i1.y,i2.y,1.0))+i.x+vec4(0.0,i1.x,i2.x,1.0));
        float n_=0.142857142857; vec3 ns=n_*D.wyz-D.xzx;
        vec4 j=p-49.0*floor(p*ns.z*ns.z);
        vec4 x_=floor(j*ns.z); vec4 y_=floor(j-7.0*x_);
        vec4 x=x_*ns.x+ns.yyyy; vec4 y=y_*ns.x+ns.yyyy;
        vec4 h=1.0-abs(x)-abs(y);
        vec4 b0=vec4(x.xy,y.xy); vec4 b1=vec4(x.zw,y.zw);
        vec4 s0=floor(b0)*2.0+1.0; vec4 s1=floor(b1)*2.0+1.0;
        vec4 sh=-step(h,vec4(0.0));
        vec4 a0=b0.xzyw+s0.xzyw*sh.xxyy;
        vec4 a1=b1.xzyw+s1.xzyw*sh.zzww;
        vec3 p0=vec3(a0.xy,h.x); vec3 p1=vec3(a0.zw,h.y);
        vec3 p2=vec3(a1.xy,h.z); vec3 p3=vec3(a1.zw,h.w);
        vec4 norm=taylorInvSqrt(vec4(dot(p0,p0),dot(p1,p1),dot(p2,p2),dot(p3,p3)));
        p0*=norm.x; p1*=norm.y; p2*=norm.z; p3*=norm.w;
        vec4 m=max(0.6-vec4(dot(x0,x0),dot(x1,x1),dot(x2,x2),dot(x3,x3)),0.0);
        m=m*m;
        return 42.0*dot(m*m,vec4(dot(p0,x0),dot(p1,x1),dot(p2,x2),dot(p3,x3)));
      }

      vec2 warp(vec2 p, float t){
        float n1=snoise(vec3(p*2.0,t*0.3));
        float n2=snoise(vec3(p*3.0+50.0,t*0.5));
        return p+vec2(n1,n2)*0.15;
      }

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
        if(idx<0.5){
          return mix(vec3(1.0,0.2,0.6),vec3(0.4,0.1,0.6),sin(t*3.0)*0.5+0.5);
        }else if(idx<1.5){
          return mix(vec3(1.0,0.8,0.2),vec3(1.0,0.3,0.1),sin(t*2.5)*0.5+0.5);
        }else if(idx<2.5){
          return mix(vec3(0.0,0.5,0.9),vec3(0.0,0.9,0.7),sin(t*3.5)*0.5+0.5);
        }else if(idx<3.5){
          return mix(vec3(0.7,0.7,0.8),vec3(0.3,0.3,0.4),sin(t*2.0)*0.5+0.5);
        }else{
          return vec3(sin(t*5.0)*0.5+0.5,sin(t*5.0+2.1)*0.5+0.5,sin(t*5.0+4.2)*0.5+0.5);
        }
      }

      void main(){
        vec2 uv=vUv;
        float t=uTime;

        vec2 p=uv*3.0;
        p=warp(p,t);
        p=warp(p,t*0.7+100.0);

        float n=fbm(p,t);
        float n2=fbm(p*1.5+vec2(50.0),t*0.8);

        float densita=0.3+uBass*0.7;
        float forma=smoothstep(0.3-densita*0.2,0.7+densita*0.3,n);

        float tIdx=floor(uTheme);
        float tMix=fract(uTheme);
        vec3 c0=paletteNuvole(n2+uBass,tIdx);
        vec3 c1=paletteNuvole(n2+uBass,min(tIdx+1.0,4.0));
        vec3 col=mix(c0,c1,tMix);

        vec4 prev=texture2D(tPrev,uv);
        vec3 nuovo=col*forma*0.15;
        vec3 finale=prev.rgb*0.95+nuovo;

        gl_FragColor=vec4(finale,1.0);
      }
    `;
```

- [ ] **Step 4.2: Verifica**

Aprire il file nel browser. Nella console, nessun errore di sintassi shader. Lo shader `fsNuvole` deve essere definito globalmente.

- [ ] **Step 4.3: Commit**

```bash
git add index.html
git commit -m "feat: add nuvole domain-warping + FBM shader"
```

---

## Task 5: Scrivere Shader Radici

**File:**
- Modifica: `C:\Users\Christian portatile\Documents\GitHub\Music Visual\index.html`

- [ ] **Step 5.1: Aggiungere shader fsRadici**

Dopo `fsNuvole`, aggiungere:

```javascript
    const fsRadici = `
      precision highp float;
      uniform sampler2D tPrev;
      uniform float uTime;
      uniform float uBeat;
      uniform float uTreble;
      uniform float uTheme;
      varying vec2 vUv;

      float hash(vec2 p){return fract(sin(dot(p,vec2(127.1,311.7)))*43758.5453);}

      vec3 paletteRadici(float t, float idx){
        if(idx<0.5) return vec3(1.0,0.4,0.7);
        else if(idx<1.5) return vec3(1.0,0.8,0.0);
        else if(idx<2.5) return vec3(0.0,0.8,1.0);
        else if(idx<3.5) return vec3(0.9,0.9,0.9);
        else return vec3(0.8,0.2,1.0);
      }

      void main(){
        vec2 uv=vUv;
        vec4 prev=texture2D(tPrev,uv);

        vec2 center=uv-0.5;
        float dist=length(center);
        float angle=atan(center.y,center.x);

        float spawn=0.0;
        if(uBeat>0.6){
          float seed=hash(vec2(floor(uTime*10.0),0.0));
          float angSeed=seed*6.28318;
          float distSeed=0.02+hash(vec2(seed,1.0))*0.15;
          float lineDist=abs(fract((angle-angSeed)/6.28318+0.5)-0.5);
          spawn=smoothstep(0.05+hash(vec2(seed,2.0))*0.1,0.0,lineDist)*smoothstep(0.02,0.0,abs(dist-distSeed));
        }

        float texel=1.0/256.0;
        float n=texture2D(tPrev,uv+vec2(0.0,texel)).r;
        float s=texture2D(tPrev,uv+vec2(0.0,-texel)).r;
        float e=texture2D(tPrev,uv+vec2(texel,0.0)).r;
        float w=texture2D(tPrev,uv+vec2(-texel,0.0)).r;
        float grow=(n+s+e+w)*0.25;

        float noise=hash(uv*100.0+uTime*50.0);
        grow+=noise*uTreble*0.3;

        float tIdx=floor(uTheme);
        vec3 col=paletteRadici(uTime*0.5+uTreble,tIdx);

        float newBright=clamp(spawn*0.8+grow*0.6,0.0,1.0);
        vec3 nuovo=col*newBright;

        vec3 finale=prev.rgb*0.90+nuovo;

        gl_FragColor=vec4(finale,1.0);
      }
    `;
```

- [ ] **Step 5.2: Verifica**

Nessun errore console. Shader `fsRadici` definito.

- [ ] **Step 5.3: Commit**

```bash
git add index.html
git commit -m "feat: add radici procedural branching shader"
```

---

## Task 6: Scrivere Shader Muschio

**File:**
- Modifica: `C:\Users\Christian portatile\Documents\GitHub\Music Visual\index.html`

- [ ] **Step 6.1: Aggiungere shader fsMuschio**

Dopo `fsRadici`, aggiungere:

```javascript
    const fsMuschio = `
      precision highp float;
      uniform sampler2D tPrev;
      uniform float uTime;
      uniform float uTreble;
      uniform float uVolume;
      uniform float uTheme;
      varying vec2 vUv;

      float hash(vec2 p){return fract(sin(dot(p,vec2(127.1,311.7)))*43758.5453);}

      void main(){
        vec2 uv=vUv;
        vec4 prev=texture2D(tPrev,uv);

        float edgeDist=min(min(uv.x,1.0-uv.x),min(uv.y,1.0-uv.y));
        float growth=0.02+uTreble*0.03;

        float seed=hash(vec2(floor(uv.x*50.0),floor(uv.y*50.0))+uTime*0.1);
        float threshold=0.7-uTreble*0.3;

        float texel=1.0/256.0;
        float n=texture2D(tPrev,uv+vec2(0.0,texel)).g;
        float s=texture2D(tPrev,uv+vec2(0.0,-texel)).g;
        float e=texture2D(tPrev,uv+vec2(texel,0.0)).g;
        float w=texture2D(tPrev,uv+vec2(-texel,0.0)).g;
        float avg=(n+s+e+w)*0.25;

        float newGrowth=0.0;
        if(edgeDist<0.15 && seed>threshold){
          newGrowth=0.5;
        }
        if(avg>0.1 && hash(uv*100.0+uTime)<avg*growth*10.0){
          newGrowth=max(newGrowth,avg*0.8);
        }

        float edgeIrid=0.0;
        if(newGrowth>0.1 || avg>0.1){
          edgeIrid=sin(uTime*3.0+edgeDist*20.0)*0.5+0.5;
        }

        float tIdx=floor(uTheme);
        vec3 baseCol;
        if(tIdx<0.5) baseCol=vec3(0.6,0.2,0.4);
        else if(tIdx<1.5) baseCol=vec3(0.7,0.6,0.1);
        else if(tIdx<2.5) baseCol=vec3(0.1,0.5,0.6);
        else if(tIdx<3.5) baseCol=vec3(0.5,0.5,0.5);
        else baseCol=vec3(0.5,0.2,0.8);

        vec3 iridCol=vec3(
          sin(uTime*2.0+edgeDist*15.0)*0.5+0.5,
          sin(uTime*2.0+edgeDist*15.0+2.1)*0.5+0.5,
          sin(uTime*2.0+edgeDist*15.0+4.2)*0.5+0.5
        );

        vec3 col=mix(baseCol,iridCol,edgeIrid*uVolume);

        float muschio=clamp(newGrowth+avg*0.95,0.0,1.0);
        vec3 finale=prev.rgb*0.92+col*muschio*0.15;

        gl_FragColor=vec4(finale,1.0);
      }
    `;
```

- [ ] **Step 6.2: Verifica**

Nessun errore console.

- [ ] **Step 6.3: Commit**

```bash
git add index.html
git commit -m "feat: add muschio edge-growth + iridescent shader"
```

---

## Task 7: Scrivere Shader Mixer Finale

**File:**
- Modifica: `C:\Users\Christian portatile\Documents\GitHub\Music Visual\index.html`

- [ ] **Step 7.1: Aggiungere shader fsMixer**

Dopo `fsMuschio`, aggiungere:

```javascript
    const fsMixer = `
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

        vec3 finale=scene;
        finale+=nuvole*uNuvole*(0.5+uBass*0.5);
        finale+=radici*uRadici;
        finale+=muschio*uMuschio;

        float vig=1.0-length(vUv-0.5)*0.3;
        finale*=vig;
        finale=clamp(finale,0.0,1.0);

        gl_FragColor=vec4(finale,1.0);
      }
    `;
```

- [ ] **Step 7.2: Verifica**

Nessun errore console. Tutti e 4 i nuovi shader (`fsNuvole`, `fsRadici`, `fsMuschio`, `fsMixer`) devono essere definiti globalmente.

- [ ] **Step 7.3: Commit**

```bash
git add index.html
git commit -m "feat: add final mixer shader for all layers"
```

---

## Task 8: Integrare Loop di Animazione

**File:**
- Modifica: `C:\Users\Christian portatile\Documents\GitHub\Music Visual\index.html`

- [ ] **Step 8.1: Aggiornare getAudio() per ritornare anche beat**

Modificare la funzione `getAudio()` per calcolare anche il beat:

```javascript
    function getAudio(){
      if(!analyser||!audioCtx) return{bass:0,treble:0,beat:0};
      const data=new Uint8Array(analyser.frequencyBinCount);
      analyser.getByteFrequencyData(data);
      const nyq=audioCtx.sampleRate/2, bins=analyser.frequencyBinCount, fpb=nyq/bins;
      const bEnd=Math.min(Math.floor(150/fpb),bins); let bSum=0;
      for(let i=0;i<bEnd;i++) bSum+=data[i];
      const bass=bSum/Math.max(bEnd*255,1);
      const tSt=Math.min(Math.floor(2000/fpb),bins);
      const tEn=Math.min(Math.floor(8000/fpb),bins); let tSum=0;
      for(let i=tSt;i<tEn;i++) tSum+=data[i];
      const treble=tSum/Math.max((tEn-tSt)*255,1);
      // Beat detection: bassi sopra soglia
      const beat=bass>0.5?bass:0;
      return{bass,treble,beat};
    }
```

- [ ] **Step 8.2: Aggiornare animate() con i nuovi passi**

Trovare il blocco del loop animate e sostituire il passo ping-pong finale (e eventuali vecchi passi RD) con:

```javascript
      // Leggi slider
      const nuvoleVal=parseFloat(document.getElementById('slider-nuvole').value);
      const radiciVal=parseFloat(document.getElementById('slider-radici').value);
      const muschioVal=parseFloat(document.getElementById('slider-muschio').value);

      // ============================================
      // PASSO 1: Scie fluide + scena 3D (rtA → rtB)
      // ============================================
      feedbackMat.uniforms.tDiffuse.value=rtA.texture;
      feedbackMat.uniforms.uZoom.value=0.998-bass*0.004;
      feedbackMat.uniforms.uFade.value=0.97+bass*0.03;
      feedbackMat.uniforms.uBlur.value=0.8+bass*0.5;
      renderer.setRenderTarget(rtB);
      renderer.clear();
      renderer.render(feedbackScene,orthoCamera);

      renderer.autoClear=false;
      renderer.render(scene,camera,rtB);
      renderer.autoClear=true;

      // ============================================
      // PASSO 2: Nuvole (se attivo)
      // ============================================
      if(nuvoleVal>0.0){
        nuvoleMat.uniforms.tPrev.value=rtNuvoleA.texture;
        nuvoleMat.uniforms.uTime.value=elapsed;
        nuvoleMat.uniforms.uBass.value=bass;
        nuvoleMat.uniforms.uTheme.value=themeVal;
        renderer.setRenderTarget(rtNuvoleB);
        renderer.clear();
        renderer.render(nuvoleScene,orthoCamera);
        let tmp=rtNuvoleA; rtNuvoleA=rtNuvoleB; rtNuvoleB=tmp;
      }

      // ============================================
      // PASSO 3: Radici (se attivo)
      // ============================================
      if(radiciVal>0.0){
        radiciMat.uniforms.tPrev.value=rtRadiciA.texture;
        radiciMat.uniforms.uTime.value=elapsed;
        radiciMat.uniforms.uBeat.value=a.beat;
        radiciMat.uniforms.uTreble.value=treble;
        radiciMat.uniforms.uTheme.value=themeVal;
        renderer.setRenderTarget(rtRadiciB);
        renderer.clear();
        renderer.render(radiciScene,orthoCamera);
        let tmp=rtRadiciA; rtRadiciA=rtRadiciB; rtRadiciB=tmp;
      }

      // ============================================
      // PASSO 4: Muschio (se attivo)
      // ============================================
      if(muschioVal>0.0){
        muschioMat.uniforms.tPrev.value=rtMuschioA.texture;
        muschioMat.uniforms.uTime.value=elapsed;
        muschioMat.uniforms.uTreble.value=treble;
        muschioMat.uniforms.uVolume.value=(bass+treble)*0.5;
        muschioMat.uniforms.uTheme.value=themeVal;
        renderer.setRenderTarget(rtMuschioB);
        renderer.clear();
        renderer.render(muschioScene,orthoCamera);
        let tmp=rtMuschioA; rtMuschioA=rtMuschioB; rtMuschioB=tmp;
      }

      // ============================================
      // PASSO 5: Mixer finale
      // ============================================
      mixerMat.uniforms.tScene.value=rtB.texture;
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
```

- [ ] **Step 8.3: Verifica visiva**

Aprire nel browser:
1. Tutti e 3 gli slider a 0 → visual identico a `funziona.html`
2. `Nuvole` a 100% → masse fluide colorate che fluttuano
3. `Radici` a 100% → filamenti luminosi dal centro
4. `Muschio` a 100% → macchie ai bordi con bordi brillanti
5. Tutti insieme → ecosistema completo

- [ ] **Step 8.4: Commit**

```bash
git add index.html
git commit -m "feat: integrate all 3 organic layers into render loop"
```

---

## Task 9: Inizializzazione degli Stati

**File:**
- Modifica: `C:\Users\Christian portatile\Documents\GitHub\Music Visual\index.html`

- [ ] **Step 9.1: Aggiungere inizializzazione a nero per tutti i render target**

Alla fine della funzione `init()`, prima di `window.addEventListener('resize',onResize);`, aggiungere:

```javascript
      // Inizializza tutti i render target a nero
      const blackMat=new THREE.MeshBasicMaterial({color:0x000000});
      const blackScene=new THREE.Scene();
      blackScene.add(new THREE.Mesh(planeGeo,blackMat));

      renderer.setRenderTarget(rtNuvoleA); renderer.render(blackScene,orthoCamera);
      renderer.setRenderTarget(rtNuvoleB); renderer.render(blackScene,orthoCamera);
      renderer.setRenderTarget(rtRadiciA); renderer.render(blackScene,orthoCamera);
      renderer.setRenderTarget(rtRadiciB); renderer.render(blackScene,orthoCamera);
      renderer.setRenderTarget(rtMuschioA); renderer.render(blackScene,orthoCamera);
      renderer.setRenderTarget(rtMuschioB); renderer.render(blackScene,orthoCamera);

      renderer.setRenderTarget(null);
```

- [ ] **Step 9.2: Verifica**

Nessun errore console. Gli strati partono da nero (invisibile) finché non vengono attivati dallo slider.

- [ ] **Step 9.3: Commit**

```bash
git add index.html
git commit -m "feat: initialize all organic layer render targets to black"
```

---

## Task 10: Test Finale e Polish

**File:**
- Modifica: `C:\Users\Christian portatile\Documents\GitHub\Music Visual\index.html` (solo se necessario)

- [ ] **Step 10.1: Test visivo completo**

Checklist:
- [ ] Slider Nuvole 0-100%: effetto fluido, onirico, si attorciglia
- [ ] Slider Radici 0-100%: germogli dal centro sui beat
- [ ] Slider Muschio 0-100%: macchie dai bordi, bordi iridescenti
- [ ] Tema cambia colore di tutti e 3 gli strati
- [ ] Bassi espandono Nuvole e germogliano Radici
- [ ] Alti accelerano crescita Muschio
- [ ] Tutti a 0 = identico a `funziona.html`
- [ ] Console: zero errori

- [ ] **Step 10.2: Performance check**

DevTools → Performance → Record 5 secondi con tutti gli slider a 100%.
FPS deve rimanere ≥ 30. Se cala:
- Ridurre risoluzione render target a w/2, h/2
- O ridurre il numero di iterazioni FBM in fsNuvole da 5 a 4

- [ ] **Step 10.3: Commit finale**

```bash
git add index.html
git commit -m "feat: complete bosco incantato organic layers system"
```

---

## Self-Review Checklist

**Spec coverage:**
- [x] 3 slider separati (Nuvole, Radici, Muschio) → Task 2
- [x] Shader Nuvole (domain warping + FBM) → Task 4
- [x] Shader Radici (procedural branching) → Task 5
- [x] Shader Muschio (edge-growth + iridescence) → Task 6
- [x] Shader Mixer finale → Task 7
- [x] Loop animate con passi condizionali → Task 8
- [x] Inizializzazione a nero → Task 9
- [x] Reattività audio per ogni strato → Task 4,5,6,8

**Placeholder scan:** Nessun TBD/TODO. Ogni step ha codice completo.

**Type consistency:**
- `uBeat` → float (bassi > soglia), usato in fsRadici e getAudio
- `uNuvole`, `uRadici`, `uMuschio` → float 0-1 dal DOM
- Tutti i render target: `rt*A`, `rt*B` swap corretto in Task 8

---

## Execution Handoff

**Plan complete and saved to `docs/superpowers/plans/2025-06-12-bosco-incantato.md`.**

**Two execution options:**

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration  
**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?**
