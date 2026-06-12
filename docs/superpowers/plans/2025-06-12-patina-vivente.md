# Patina Vivente — Reaction-Diffusion Layer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Aggiungere uno strato reaction-diffusion (RD) controllabile via slider "Organicità" sopra le scie fluide esistenti nel visualizer `index.html`.

**Architecture:** Due nuovi render target (`rdA`, `rdB`) simulano un passo Gray-Scott RD per frame. Un terzo passo composita le scie 3D con il pattern RD, colorandolo dal tema attivo. Quando il slider è a 0, il passo RD viene saltato per preservare performance.

**Tech Stack:** Three.js (WebGLRenderer, ShaderMaterial, WebGLRenderTarget), Web Audio API (AnalyseNode esistente), HTML/CSS vanilla.

---

## File Structure

| File | Azione | Responsabilità |
|---|---|---|
| `C:\Users\Christian portatile\Documents\GitHub\Music Visual\index.html` | **Modifica** | Aggiunge shader RD, scene RD, loop RD, UI slider, inizializzazione |
| `C:\Users\Christian portatile\Documents\GitHub\Music Visual\funziona.html` | Invariato | Backup del visual originale |
| Browser (Chrome/Edge) | **Test visivo** | Verifica che RD appaia a 100%, assenza di artefatti, performance accettabili |

---

## Task 1: Aggiungere UI Slider "Organicità"

**File:**
- Modifica: `C:\Users\Christian portatile\Documents\GitHub\Music Visual\index.html:33-39` (UI overlay)
- Modifica: `index.html:411-436` (setupUI)

- [ ] **Step 1.1: Aggiungere HTML dello slider**

Aggiungere il blocco slider subito dopo lo slider "Tema":

```html
    <div class="slider-row">
      <label>Organicità</label>
      <input type="range" id="slider-organic" min="0" max="1" step="0.01" value="0">
      <span id="organic-label">0%</span>
    </div>
```

- [ ] **Step 1.2: Aggiungere listener in setupUI()**

Dopo il listener di `sldThm`, aggiungere:

```javascript
      const sldOrg=document.getElementById('slider-organic');
      const lblOrg=document.getElementById('organic-label');
      sldOrg.addEventListener('input',(e)=>{
        const v=parseFloat(e.target.value);
        lblOrg.textContent=Math.round(v*100)+'%';
      });
```

- [ ] **Step 1.3: Verifica visiva**

Aprire `index.html` nel browser. Controllare che sotto "Tema" appaia la riga:
`Organicità [━━━━━━━○━━━━━━━] 0%`
Muovere lo slider: il numero deve cambiare da 0% a 100%.

- [ ] **Step 1.4: Commit intermedio**

```bash
cd "C:\Users\Christian portatile\Documents\GitHub\Music Visual"
git add index.html
git commit -m "ui: add organic slider control"
```

---

## Task 2: Aggiungere Variabili Globali e Render Target RD

**File:**
- Modifica: `index.html:60-63` (sezione Ping-Pong Feedback)
- Modifica: `index.html:287-320` (init)
- Modifica: `index.html:325-330` (onResize)

- [ ] **Step 2.1: Dichiarare variabili RD**

Aggiungere dopo `let feedbackMat, displayMat;`:

```javascript
    // --- REACTION-DIFFUSION ---
    let rdA, rdB;
    let rdSimScene, rdCompositeScene;
    let rdSimMat, rdCompositeMat;
```

- [ ] **Step 2.2: Creare render target e scene in init()**

Dopo la creazione di `displayScene` (riga ~320), aggiungere:

```javascript
      // --- RD RENDER TARGETS ---
      rdA=new THREE.WebGLRenderTarget(w,h,rtOpts);
      rdB=new THREE.WebGLRenderTarget(w,h,rtOpts);

      // RD Simulation: applica un passo Gray-Scott
      rdSimScene=new THREE.Scene();
      rdSimMat=new THREE.ShaderMaterial({
        uniforms:{
          tPrev:{value:null},
          uFeed:{value:0.054},
          uKill:{value:0.063},
          uDiffB:{value:0.5},
          uTime:{value:0}
        },
        vertexShader:vsFeedback,
        fragmentShader:fsRD, // definito in Task 3
        depthWrite:false
      });
      rdSimScene.add(new THREE.Mesh(planeGeo,rdSimMat));

      // RD Composite: mischia scene 3D + pattern RD
      rdCompositeScene=new THREE.Scene();
      rdCompositeMat=new THREE.ShaderMaterial({
        uniforms:{
          tScene:{value:null},
          tRD:{value:null},
          uOrganic:{value:0},
          uBass:{value:0},
          uTime:{value:0},
          uTheme:{value:0}
        },
        vertexShader:vsDisplay,
        fragmentShader:fsRDComposite, // definito in Task 4
        depthWrite:false
      });
      rdCompositeScene.add(new THREE.Mesh(planeGeo,rdCompositeMat));
```

- [ ] **Step 2.3: Aggiornare onResize**

Aggiungere in `onResize()`:

```javascript
      rdA.setSize(w,h); rdB.setSize(w,h);
```

- [ ] **Step 2.4: Verifica visiva**

Aprire la console DevTools (F12). Nessun errore `THREE is not defined` o simile. `rdA`, `rdB` devono esistere come `WebGLRenderTarget`.

- [ ] **Step 2.5: Commit**

```bash
git add index.html
git commit -m "feat: add RD render targets and scenes"
```

---

## Task 3: Scrivere Shader Reaction-Diffusion Simulation

**File:**
- Modifica: `index.html:240-256` (dopo fsDisplay)

- [ ] **Step 3.1: Inserire shader fsRD**

Aggiungere dopo `fsDisplay` (prima della funzione `init`):

```javascript
    const fsRD = `
      precision mediump float;
      uniform sampler2D tPrev;
      uniform float uFeed;
      uniform float uKill;
      uniform float uDiffB;
      uniform float uTime;
      varying vec2 vUv;

      void main(){
        float texel=1.0/256.0;
        vec2 uv=vUv;

        // Campiona i 4 vicini + centro
        vec4 c=texture2D(tPrev,uv);
        vec4 n=texture2D(tPrev,uv+vec2(0.0,texel));
        vec4 s=texture2D(tPrev,uv+vec2(0.0,-texel));
        vec4 e=texture2D(tPrev,uv+vec2(texel,0.0));
        vec4 w=texture2D(tPrev,uv+vec2(-texel,0.0));

        // Laplaciano (media dei vicini - centro)
        vec4 lap=(n+s+e+w)-4.0*c;

        float a=c.r;
        float b=c.g;

        float diffA=1.0;   // diffusione di A
        float diffB=uDiffB; // diffusione di B, controllata dall'audio

        float reaction=a*b*b;
        float newA=a+(diffA*lap.r-reaction+uFeed*(1.0-a))*0.8;
        float newB=b+(diffB*lap.g+reaction-(uKill+uFeed)*b)*0.8;

        // Clamp per stabilità
        newA=clamp(newA,0.0,1.0);
        newB=clamp(newB,0.0,1.0);

        gl_FragColor=vec4(newA,newB,0.0,1.0);
      }
    `;
```

- [ ] **Step 3.2: Verifica sintassi**

Aprire il file nel browser. Se il browser mostra schermo nero e la console ha errori tipo `syntax error` nello shader, correggere virgole/punti e virgola.

- [ ] **Step 3.3: Commit**

```bash
git add index.html
git commit -m "feat: add Gray-Scott RD simulation shader"
```

---

## Task 4: Scrivere Shader RD Composite

**File:**
- Modifica: `index.html` subito dopo fsRD

- [ ] **Step 4.1: Inserire shader fsRDComposite**

Aggiungere dopo `fsRD`:

```javascript
    const fsRDComposite = `
      precision mediump float;
      uniform sampler2D tScene;
      uniform sampler2D tRD;
      uniform float uOrganic;
      uniform float uBass;
      uniform float uTime;
      uniform float uTheme;
      varying vec2 vUv;

      vec3 themeColor(float idx){
        if(idx<0.5) return vec3(1.0,0.0,0.5);
        else if(idx<1.5) return vec3(1.0,0.9,0.0);
        else if(idx<2.5) return vec3(0.0,0.4,1.0);
        else if(idx<3.5) return vec3(0.8,0.8,0.8);
        else return vec3(1.0,0.0,1.0);
      }

      void main(){
        vec4 scene=texture2D(tScene,vUv);
        vec4 rd=texture2D(tRD,vUv);

        // Colore del tema attivo
        float tIdx=floor(uTheme);
        float tNext=min(tIdx+1.0,4.0);
        float tMix=fract(uTheme);
        vec3 c0=themeColor(tIdx);
        vec3 c1=themeColor(tNext);
        vec3 themeCol=mix(c0,c1,tMix);

        // Il pattern RD è in B (green channel = concentrazione B)
        float pattern=rd.g;

        // Intensità base + boost dai bassi
        float intensity=uOrganic*(0.4+uBass*0.6);

        // Colora il pattern con il tema + glow ciano
        vec3 rdCol=themeCol*pattern*intensity*1.5;
        rdCol+=vec3(0.2,0.8,0.6)*pattern*intensity*0.5;

        // Compositing: scie + RD (additive)
        vec3 final=scene.rgb+rdCol;

        // Vignetta leggera per concentrare l'attenzione al centro
        float vig=1.0-length(vUv-0.5)*0.5;
        final*=vig;

        gl_FragColor=vec4(final,1.0);
      }
    `;
```

- [ ] **Step 4.2: Verifica sintassi**

Stesso check del Task 3: browser, console, nessun errore shader.

- [ ] **Step 4.3: Commit**

```bash
git add index.html
git commit -m "feat: add RD compositing shader with theme coloring"
```

---

## Task 5: Integrare RD nel Loop di Animazione

**File:**
- Modifica: `index.html:530-561` (sezione PING-PONG FEEDBACK)

- [ ] **Step 5.1: Aggiornare la logica del loop animate()**

Sostituire l'intero blocco dal commento `// ============================================` a riga 530 fino al commento `// Passo 4: swap` (riga 558) con:

```javascript
      // ============================================
      // PING-PONG FEEDBACK + REACTION-DIFFUSION
      // ============================================

      // Passo 1: Feedback Scie (rtA → rtB)
      feedbackMat.uniforms.tDiffuse.value=rtA.texture;
      feedbackMat.uniforms.uZoom.value=0.998-bass*0.004;
      feedbackMat.uniforms.uFade.value=0.97+bass*0.03;
      feedbackMat.uniforms.uBlur.value=0.8+bass*0.5;
      renderer.setRenderTarget(rtB);
      renderer.clear();
      renderer.render(feedbackScene,orthoCamera);

      // Passo 2: Render scena 3D sopra rtB (senza cancellare)
      renderer.autoClear=false;
      renderer.render(scene,camera,rtB);
      renderer.autoClear=true;

      // Passo 3: Reaction-Diffusion (se attivo)
      const organicVal=parseFloat(document.getElementById('slider-organic').value);
      if(organicVal>0.0){
        // 3a: Simula un passo RD
        rdSimMat.uniforms.tPrev.value=rdA.texture;
        rdSimMat.uniforms.uFeed.value=0.054+bass*0.02;
        rdSimMat.uniforms.uKill.value=0.063+treble*0.01;
        rdSimMat.uniforms.uDiffB.value=0.5+bass*0.2;
        rdSimMat.uniforms.uTime.value=elapsed;
        renderer.setRenderTarget(rdB);
        renderer.clear();
        renderer.render(rdSimScene,orthoCamera);

        // 3b: Compositing finale: rtB (scie+3D) + rdB (pattern)
        rdCompositeMat.uniforms.tScene.value=rtB.texture;
        rdCompositeMat.uniforms.tRD.value=rdB.texture;
        rdCompositeMat.uniforms.uOrganic.value=organicVal;
        rdCompositeMat.uniforms.uBass.value=bass;
        rdCompositeMat.uniforms.uTime.value=elapsed;
        rdCompositeMat.uniforms.uTheme.value=themeVal;
        renderer.setRenderTarget(null);
        renderer.render(rdCompositeScene,orthoCamera);

        // 3c: Swap RD ping-pong
        const rdTemp=rdA; rdA=rdB; rdB=rdTemp;
      }else{
        // Se Organicità = 0, mostra rtB direttamente (comportamento originale)
        renderer.setRenderTarget(null);
        displayMat.uniforms.tDiffuse.value=rtB.texture;
        renderer.render(displayScene,orthoCamera);
      }

      // Passo 4: Swap scie ping-pong
      const temp=rtA; rtA=rtB; rtB=temp;
```

- [ ] **Step 5.2: Verifica visiva**

Aprire nel browser, caricare una canzone. Con slider Organicità a 0 → deve funzionare **esattamente come prima** (scie fluide, nessuna differenza visibile). Aumentare a 100% → deve apparire un pattern organico simile a coralli o macchie che pulsa con la musica.

- [ ] **Step 5.3: Commit**

```bash
git add index.html
git commit -m "feat: integrate RD simulation into main render loop"
```

---

## Task 6: Inizializzazione dello Stato RD

**File:**
- Modifica: `index.html:261-322` (funzione init)

- [ ] **Step 6.1: Aggiungere inizializzazione RD con seme**

Dopo la creazione di `rdB` (riga ~295), aggiungere questa funzione di inizializzazione chiamata immediatamente:

```javascript
      // Inizializza lo stato RD con un seme centrale + macchie random
      function initRD(){
        const rdCanvas=document.createElement('canvas');
        rdCanvas.width=512; rdCanvas.height=512;
        const ctx=rdCanvas.getContext('2d');
        const imgData=ctx.createImageData(512,512);
        const d=imgData.data;
        for(let i=0;i<512*512;i++){
          const x=i%512, y=Math.floor(i/512);
          const dx=x-256, dy=y-256;
          const dist=Math.sqrt(dx*dx+dy*dy);
          // Centro: bianco (A=1, B=1)
          let a=0.0, b=0.0;
          if(dist<20){a=1.0;b=1.0;}
          // Macchie random sparse
          else if(Math.random()<0.003){a=1.0;b=1.0;}
          d[i*4]=Math.floor(a*255);
          d[i*4+1]=Math.floor(b*255);
          d[i*4+2]=0;
          d[i*4+3]=255;
        }
        ctx.putImageData(imgData,0,0);
        const tex=new THREE.CanvasTexture(rdCanvas);
        tex.minFilter=THREE.LinearFilter;
        tex.magFilter=THREE.LinearFilter;
        // Copia texture iniziale su rdA
        const initMat=new THREE.ShaderMaterial({
          uniforms:{tSeed:{value:tex}},
          vertexShader:vsDisplay,
          fragmentShader:`
            precision mediump float;
            uniform sampler2D tSeed;
            varying vec2 vUv;
            void main(){gl_FragColor=texture2D(tSeed,vUv);}
          `,
          depthWrite:false
        });
        const initScene=new THREE.Scene();
        initScene.add(new THREE.Mesh(planeGeo,initMat));
        renderer.setRenderTarget(rdA);
        renderer.render(initScene,orthoCamera);
        renderer.setRenderTarget(null);
      }
      initRD();
```

- [ ] **Step 6.2: Verifica visiva**

Aprire il browser. Con Organicità a 100%, il pattern deve apparire immediatamente (non dopo 10 secondi di accumulo). Deve esserci un blob centrale + piccole macchie sparse.

- [ ] **Step 6.3: Commit**

```bash
git add index.html
git commit -m "feat: initialize RD state with central seed and random spots"
```

---

## Task 7: Test Finale e Polish

**File:**
- Modifica: `index.html` (solo se necessario)

- [ ] **Step 7.1: Test visivo completo**

Eseguire questa checklist nel browser:

1. **Organicità 0%:** Il visual è identico a `funziona.html` (nessuna differenza)
2. **Organicità 50%:** Si vede un pattern sottile, simile a una patina, sopra le scie
3. **Organicità 100%:** Pattern chiaro, corallino, che pulsa con i bassi
4. **Cambio tema:** Il pattern RD cambia colore (rosa, giallo, blu, b/n, arcobaleno)
5. **Pausa canzone:** Pattern si ritrae lentamente, tornano le scie pure
6. **Bassi forti:** Pattern diventa più denso/ramificato
7. **Alti forti:** Pattern diventa più "nervoso"
8. **Nessun errore console:** DevTools → Console deve essere vuoto

- [ ] **Step 7.2: Performance check**

Aprire DevTools → Performance → Record 5 secondi con Organicità a 100%. FPS deve rimanere ≥ 30 su GPU integrata moderna. Se scende sotto 30, ridurre `rdCanvas` in `initRD()` da 512×512 a 256×256.

- [ ] **Step 7.3: Commit finale**

```bash
git add index.html
git commit -m "feat: complete RD patina vivente layer with full audio reactivity"
```

---

## Self-Review Checklist

**Spec coverage:**
- [x] Nuovo slider "Organicità" 0-1 → Task 1
- [x] Due render target RD → Task 2
- [x] Shader Gray-Scott RD simulation → Task 3
- [x] Shader compositing RD con tema colorato → Task 4
- [x] Loop animate() con passo condizionale → Task 5
- [x] Inizializzazione RD con seme → Task 6
- [x] Parametri audio-driven (feed/kill/diffB) → Task 5
- [x] Ottimizzazione: skip RD quando organic=0 → Task 5

**Placeholder scan:** Nessun TBD/TODO trovato. Ogni task ha codice completo e comandi esatti.

**Type consistency:**
- `uOrganic` → float 0-1, usato come `organicVal` in JS
- `uFeed`, `uKill`, `uDiffB` → float, mappati da bass/treble
- `rdA`, `rdB` → WebGLRenderTarget, swap corretto in Task 5 e 6

---

## Execution Handoff

**Plan complete and saved to `docs/superpowers/plans/2025-06-12-patina-vivente.md`.**

**Two execution options:**

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration  
**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints

**Which approach?**
