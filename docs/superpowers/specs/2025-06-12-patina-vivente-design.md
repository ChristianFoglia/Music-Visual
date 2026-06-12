# Design: Patina Vivente — Reaction-Diffusion Layer per Nature Visual

**Data:** 2025-06-12  
**Progetto:** Music Visual — Branch nature-visual  
**File principale:** `index.html` (backup: `funziona.html`)  
**Approccio:** A — Strato aggiuntivo (reaction-diffusion come patina sovrapposta alle scie fluide)

---

## 1. Obiettivo

Aggiungere un secondo strato visivo — un effetto **reaction-diffusion (RD)** simulato via shader — sopra il feedback loop attuale (scie fluide). L'utente potrà controllarne l'intensità con uno slider "Organicità" da 0 a 1.

**Metafora visiva:** Come una muffa luminescente che cresce sul vetro di un acquario, sopra l'inchiostro colorato che scorre in fondo.

---

## 2. Architettura Render

Il rendering diventa a **3 passi** (dal ping-pong attuale a 2 passi):

```
Passo 1: Feedback Scie (esistente)
  rtA → zoom+blur+fade → rtB

Passo 2: Render scena 3D (sfera + particelle) sopra rtB
  rtB + scene → rtB_composite

Passo 3: Reaction-Diffusion (nuovo)
  rtB_composite + rd_state → rd_output

Passo 4: Display finale
  rd_output → schermo
```

In pratica:
1. Le scie fluide vengono calcolate come ora (ping-pong A↔B)
2. La scena 3D viene renderizzata sopra rtB (come ora)
3. Il reaction-diffusion legge il frame appena creato e applica il suo pattern organico
4. Il risultato finale viene mostrato

---

## 3. Componenti Nuovi

### 3.1 RD Render Targets

Due nuovi `WebGLRenderTarget`, chiamati `rdA` e `rdB`, con le stesse opzioni di `rtA/rtB`:
- `minFilter: THREE.LinearFilter`
- `magFilter: THREE.LinearFilter`
- `format: THREE.RGBAFormat`
- `type: THREE.UnsignedByteType`

Dimensioni: `window.innerWidth × window.innerHeight`, aggiornati su `onResize`.

### 3.2 RD Shader (Fragment)

Il reaction-diffusion viene simulato in uno shader che applica un'approssimazione della **Gray-Scott model** in tempo reale:

**Formula semplificata per GPU:**
```glsl
// Legge pixel vicini (up, down, left, right)
vec4 laplacian = ...;

// Simula A (sostanza che si diffonde) e B (sostanza che reagisce)
float a = texture2D(tPrev, uv).r;
float b = texture2D(tPrev, uv).g;

float reaction = a * b * b;
float newA = a + (diffusionA * lapA - reaction + feed * (1.0 - a)) * dt;
float newB = b + (diffusionB * lapB + reaction - (kill + feed) * b) * dt;
```

**Parametri audio-driven:**
| Parametro | Base | Con bassi forti | Effetto visivo |
|---|---|---|---|
| `feed` | 0.054 | +bass×0.02 | Più "nutriente", la muffa cresce |
| `kill` | 0.063 | +treble×0.01 | Più "morte", pattern più ramificati |
| `diffusionB` | 0.5 | +bass×0.2 | Diffusione più veloce = scie organiche |
| `dt` | 1.0 | fisso | Passo temporale |

**Inizializzazione:** Lo stato iniziale (`rdA`) viene precompilato con:
- Centro: blob bianco (`a=1, b=1`) per avviare la reazione
- Bordi: nero (`a=0, b=0`)
- Piccole macchie randomiche sparse per varietà

### 3.3 RD Composite Shader (Fragment)

Questo shader miscela le scie (da `rtB_composite`) con il pattern RD (da `rdB`):

```glsl
uniform sampler2D tScene;   // rtB_composite (scie + 3D)
uniform sampler2D tRD;      // rdB (pattern reaction-diffusion)
uniform float uOrganic;     // slider 0-1
uniform float uBass;        // audio
uniform float uTime;        // tempo

void main() {
    vec4 scene = texture2D(tScene, vUv);
    vec4 rd = texture2D(tRD, vUv);
    
    // Il pattern RD è in bianco/nero, lo coloriamo con il tema attuale
    vec3 rdColor = mix(vec3(0.0), themeColor, rd.g);
    
    // Bassi accendono il RD: quando suona forte, il RD diventa più visibile
    float rdIntensity = uOrganic * (0.5 + uBass * 0.5);
    
    // Mischia scena e RD con additive blending
    vec3 final = scene.rgb + rdColor * rdIntensity * rd.g;
    
    // Glow extra sui bordi del pattern RD
    final += rd.g * uOrganic * 0.3 * vec3(0.5, 1.0, 0.8);
    
    gl_FragColor = vec4(final, 1.0);
}
```

### 3.4 UI — Nuovo Slider "Organicità"

Aggiunto sotto lo slider "Tema":

```html
<div class="slider-row">
  <label>Organicità</label>
  <input type="range" id="slider-organic" min="0" max="1" step="0.01" value="0">
  <span id="organic-label">0%</span>
</div>
```

**Comportamento:**
- 0% → RD disabilitato, visual originale
- 50% → RD visibile ma tenue, come una patina sottile
- 100% → RD pieno, pattern che dominano l'immagine

---

## 4. Loop di Animazione Aggiornato

```javascript
function animate(time) {
  requestAnimationFrame(animate);
  const elapsed = time * 0.001;

  // 1. Audio + smoothing (come ora)
  // ...

  // 2. Tema + Organicità
  const themeVal = parseFloat(document.getElementById('slider-theme').value);
  const organicVal = parseFloat(document.getElementById('slider-organic').value);

  // 3. Aggiorna uniforms sfera (come ora)
  // ...

  // 4. Particelle (come ora)
  // ...

  // ============================================
  // PASSO 1: Feedback Scie (esistente)
  // ============================================
  feedbackMat.uniforms.tDiffuse.value = rtA.texture;
  feedbackMat.uniforms.uZoom.value = 0.998 - bass * 0.004;
  feedbackMat.uniforms.uFade.value = 0.97 + bass * 0.03;
  feedbackMat.uniforms.uBlur.value = 0.8 + bass * 0.5;
  
  renderer.setRenderTarget(rtB);
  renderer.clear();
  renderer.render(feedbackScene, orthoCamera);

  // ============================================
  // PASSO 2: Render scena 3D sopra rtB
  // ============================================
  renderer.autoClear = false;
  renderer.render(scene, camera, rtB);
  renderer.autoClear = true;

  // ============================================
  // PASSO 3: Reaction-Diffusion (NUOVO)
  // ============================================
  if (organicVal > 0.0) {
    // 3a: Simula un passo RD: rdA → rdB
    rdSimMat.uniforms.tPrev.value = rdA.texture;
    rdSimMat.uniforms.uFeed.value = 0.054 + bass * 0.02;
    rdSimMat.uniforms.uKill.value = 0.063 + treble * 0.01;
    rdSimMat.uniforms.uDiffB.value = 0.5 + bass * 0.2;
    
    renderer.setRenderTarget(rdB);
    renderer.clear();
    renderer.render(rdSimScene, orthoCamera);

    // 3b: Compositing finale: rtB + rdB → schermo
    rdCompositeMat.uniforms.tScene.value = rtB.texture;
    rdCompositeMat.uniforms.tRD.value = rdB.texture;
    rdCompositeMat.uniforms.uOrganic.value = organicVal;
    rdCompositeMat.uniforms.uBass.value = bass;
    rdCompositeMat.uniforms.uTime.value = elapsed;
    
    renderer.setRenderTarget(null);
    renderer.render(rdCompositeScene, orthoCamera);

    // 3c: Swap RD ping-pong
    const rdTemp = rdA; rdA = rdB; rdB = rdTemp;
  } else {
    // Se organicità = 0, salta RD e mostra rtB direttamente (come prima)
    renderer.setRenderTarget(null);
    displayMat.uniforms.tDiffuse.value = rtB.texture;
    renderer.render(displayScene, orthoCamera);
  }

  // ============================================
  // PASSO 4: Swap scie ping-pong (come ora)
  // ============================================
  const temp = rtA; rtA = rtB; rtB = temp;

  // Equalizer (come ora)
  if (freqData) updateEq(freqData);
}
```

---

## 5. Ottimizzazioni

| Problema | Soluzione |
|---|---|
| **Performance** | Se `organicVal == 0`, salta completamente il passo RD (nessun overhead) |
| **Risoluzione RD** | Possiamo ridurre `rdA/rdB` a metà risoluzione (`w/2, h/2`) per GPU deboli |
| **Inizializzazione RD** | Blob iniziale centrato + seme random per varietà a ogni refresh |
| **Colori RD** | Il pattern è in scala di grigio, colorato in compositing dal tema attivo |

---

## 6. Criteri di Successo

- [ ] Il visual funziona esattamente come prima quando "Organicità" è a 0
- [ ] A 100% si vedono chiaramente pattern organici tipo corallo/muffa/venature
- [ ] I pattern reagiscono ai bassi (crescono quando la musica è intensa)
- [ ] Il passaggio 0% → 100% è graduale e controllabile in tempo reale
- [ ] Nessun calo di frame rate significativo su GPU medie

---

## 7. Prossimi Passi (dopo questo)

1. **Simmetria speculare** (toggle) per creare "volti" psichedelici
2. **Trails individuali** per le particelle
3. **Trigger mutazione** (cambio seed del noise su soglia audio)
4. **Wireframe toggle** (pulsante UI)
5. **Slider intensità feedback** (zoom/fade separato)

---

*Design approvato dall'utente. Pronto per implementazione.*
