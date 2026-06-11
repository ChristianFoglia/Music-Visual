# Design: Visual Audio-Reattivo Three.js + GLSL

**Data:** 2026-06-11
**Progetto:** Poster Psicofarmaci

## Obiettivo
File HTML autonomo che fonda Three.js con un Fragment Shader GLSL personalizzato, creando un visual audio-reattivo in stile TouchDesigner.

## Architettura
Singolo file `.html` monolite (~700-900 righe). Carica Three.js da CDN (unpkg/jsDelivr). Nessun server richiesto: doppio clic e funziona.

### Componenti
1. **Three.js Scene**: Canvas fullscreen, camera perspective, renderer WebGL.
2. **Geometrie**: Sfera (128x128 segmenti) e Piano (200x200 segmenti). Switch istantaneo con transizione morbida di scala (una scala a 0 mentre l'altra scala a 1).
3. **Shader GLSL**: Vertex deforma con 3D Simplex Noise + audio bassi. Fragment colora con palette controllata da uniform.
4. **Web Audio API**: File input per canzone, AudioContext + AnalyserNode. Estrae bassi (0-100Hz) e alti (2000-8000Hz).
5. **UI Overlay**: Pulsanti Carica/Cambia Canzone, slider Forma, slider Tema Colore.

## Uniforms
- `u_time`: tempo in secondi
- `u_bass`: intensità bassi (0.0-1.0), smussata
- `u_treble`: intensità alti (0.0-1.0), smussata
- `u_colorTheme`: 0=Cyberpunk, 1=Magma, 2=Ocean, 3=Psychedelic
- `u_shapeMix`: 0.0=Sfera visibile, 1.0=Piano visibile (transition)

## Shader
- **Vertex**: 3D Simplex Noise inline (no librerie esterne). Deformazione radiale per sfera, verticale per piano. Amplificata da `u_bass`.
- **Fragment**: Colore basato su distanza dal centro + energia del vertice. Fresnel neon glow. Interpolazione tra 4 palette tramite `u_colorTheme`.

## Audio Flow
1. Utente clicca "Carica Canzone" → file picker.
2. File letto con `URL.createObjectURL` → `<audio>` nascosto.
3. `AudioContext` → `MediaElementSource` → `AnalyserNode`.
4. FFT data analizzata ogni frame. Valori medi per bassi/alti.
5. Lerp verso target per smoothness.

## UI
- Overlay in alto a sinistra, sfondo semi-trasparente scuro.
- Pulsanti: Carica Canzone, Cambia Canzone.
- Slider: Forma (Sfera ↔ Piano), Tema Colore (0-3).
- Mini equalizer a barre per feedback audio attivo.
- Nome file in riproduzione mostrato.

## Estetica
- Sfondo: nero profondo (#050505).
- Colori palette: Cyberpunk (viola/azzurro/fucsia), Magma (arancio/rosso/oro), Ocean (verde/blu/turchese), Psychedelic (arcobaleno HSL ciclico).
- Neon glow sui bordi via Fresnel term.

## File Output
`audio-visual.html` nella root del progetto.
