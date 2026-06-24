# Especificación técnica — BoomBox base maker

> **Propósito de este documento.** Es una especificación completa (spec-driven) para que un modelo de IA reconstruya exactamente la aplicación "BoomBox base maker". El entregable es **un único archivo HTML autocontenido** (HTML + CSS + JavaScript en línea, sin dependencias externas, sin build, sin frameworks). Debe abrirse con doble clic en cualquier navegador moderno y funcionar sin servidor.

---

## 0. Resumen ejecutivo

BoomBox es un mini-DAW (estación de audio) orientado a crear **bases de boom-bap para freestyle**. Todo el audio se **sintetiza y renderiza en el navegador** con la Web Audio API. El usuario programa patrones en un secuenciador por pasos, agrupa patrones en "variantes", encadena variantes en un "arreglo", y exporta el resultado a un archivo **WAV** real. Puede además cargar sus propios samples, guardar proyectos en archivos `.json`, y elegir entre bases y presets precargados.

Idioma de la interfaz: **español**. Sin telemetría, sin red: 100% local.

---

## 1. Restricciones técnicas (obligatorias)

1. **Un solo archivo `.html`**. Todo el CSS en un `<style>` y todo el JS en un `<script>` al final del `<body>`.
2. **Sin librerías externas, sin CDN, sin imports.** Solo APIs nativas del navegador (Web Audio API, FileReader, Blob, OfflineAudioContext).
3. **No usar `localStorage` ni `sessionStorage`.** La persistencia es solo vía descarga/carga de archivos `.json`.
4. Debe funcionar en **Chrome, Safari (incl. iOS) y Firefox**. Incluir desbloqueo de audio para iOS (ver §11).
5. Audio sintetizado por osciladores y ruido; los samples del usuario son opcionales.
6. El código debe ser legible y comentado en español.

---

## 2. Modelo musical y constantes

- **STEPS = 16**: cada patrón tiene 16 pasos (equivale a 2 compases de 4/4 en semicorcheas, o 1 compás de semicorcheas según se interprete; tratar "1 patrón = 16 pasos = 1 bloque de repetición").
- **BPM por defecto = 90**, rango editable 60–180.
- **Swing por defecto = 22%** (0–60%). El swing **atrasa las corcheas impares** (pasos con índice impar) una fracción `stepDur * swing`.
- `stepDur = 60 / bpm / 4` (duración de un paso en segundos).
- **Escala melódica** (semitonos sobre la nota base de cada instrumento): `SCALE = [0,2,3,5,7,8,10,12,14,15]`. Es una menor extendida; todos los selectores de nota ofrecen solo estos 10 grados sobre la base del instrumento. Esto garantiza consonancia.
- Conversión nota MIDI → frecuencia: `N(n) = 440 * 2^((n-69)/12)`.
- Nombres de nota: array `['C','C#','D','D#','E','F','F#','G','G#','A','A#','B']`; `midiName(m) = NOTE_NAMES[m%12] + (floor(m/12)-1)`.

---

## 3. Instrumentos del secuenciador

Hay **15 instrumentos**, cada uno es una fila del secuenciador. Campos: `id`, `name` (etiqueta visible), `color` (variable CSS), `melodic` (si tiene selector de nota por paso), `base` (nota MIDI base, solo melódicos), y `trig` (función que dispara el sonido).

| id | name | melodic | base | color (var) |
|----|------|---------|------|-------------|
| kick | Kick | no | — | --hot |
| snare | Snare | no | — | --gold |
| hat | Hi-hat | no | — | --cool |
| ohat | Open hat | no | — | --cool |
| e808 | 808 bajo | sí | 33 | --green |
| upr | Contrabajo | sí | 33 | --green |
| chord | Acordes | sí | 45 | --violet |
| rhodes | Rhodes | sí | 57 | --violet |
| piano | Piano | sí | 48 | --violet |
| violin | Violín | sí | 57 | --cool |
| flute | Flauta | sí | 69 | --cool |
| lead | Lead | sí | 69 | --cool |
| shaker | Shaker | no | — | --gold |
| rim | Rimshot | no | — | --gold |
| conga | Conga | no | — | --hot |

El orden en la UI es exactamente el de la tabla.

---

## 4. Motor de audio (síntesis)

### 4.1 Cadena de master (bus principal)

Construir una función `buildChain(ctx)` que arme y devuelva `{ctx, master, verbSend, bus}`. Ruteo:

```
master (gain 0.85)
  → highpass 30 Hz (saca subgraves/barro)
  → [ dry (gain 0.82) ] ───────────────┐
  → [ waveshaper saturador → wet 0.16 ] ┤→ compresor "glue" → highshelf +3dB @8.5kHz → out (0.9) → destination
```

- **Saturador** (WaveShaper): curva `tanh`-like suave; fórmula de la curva: para n=2048 muestras, `x = i*2/n - 1; c[i] = (1+k)*x / (1+k*|x|)` con `k=1.8`; `oversample="4x"`.
- **Compresor (glue)**: threshold −16, ratio 3, attack 0.005, release 0.16, knee 6.
- **Reverb por convolución** (send/return compartido): impulse response sintético de 1.8 s, ruido blanco decreciente `(rand*2-1) * (1 - i/len)^3` en 2 canales. Pre-filtro highpass 500 Hz antes del convolver. `verbSend` (gain 1) → preHP → convolver → `verbRet` (gain 0.2) → glue.
- **Buses por instrumento**: un `GainNode` por cada id de instrumento, conectado a master, con ganancia inicial `VOL[id]` (default 1). Además un bus `'tom'` (gain 1) para los fills y un bus `'crackle'` (gain 0.5) para el vinilo.

La función `buildChain` se usa **tanto en vivo** (sobre el `AudioContext` global `AC`) **como en el render offline** (sobre un `OfflineAudioContext`).

### 4.2 Helpers

- `noiseBuf(ctx, secs)`: AudioBuffer mono de ruido blanco.
- `aenv(gainNode, t, a, d, peak)`: envolvente exponencial — sube a `peak` en `a` s, baja a ~0 en `d` s más.

### 4.3 Funciones de instrumento

Cada función recibe `(C, t, ...)` donde `C` es la cadena (o una versión "ruteada" donde `C.master` apunta al bus del instrumento). Especificación de cada timbre:

- **I_kick(C,t,g=1)**: seno con barrido de pitch 170→46 Hz en 0.11 s; envolvente pico `g` con decay ~0.36 s; pasa por un waveshaper `tanh(x*2)` para saturar el cuerpo; **click de ataque** = ruido por bandpass 2000 Hz, envolvente muy corta (0.02 s) a `g*0.4`.
- **I_snare(C,t,g=0.85)**: dos parciales triangulares (190 y 340 Hz) con envolvente corta, enviados a reverb (0.12); más cuerpo de **ruido** por bandpass 2200 Hz + highpass 900 Hz, envolvente 0.17 s a `g*0.85`, reverb 0.22.
- **I_hat(C,t,g=0.4,open=false)**: ruido por highpass 8800 Hz + peaking 11500 Hz (+6 dB, Q2); decay 0.04 s (cerrado) o 0.2 s (abierto).
- **I_808(C,t,freq,g=0.7)**: seno con glide `freq*1.5 → freq` en 0.04 s; envolvente 0.5 s; waveshaper `tanh(x*1.6)`. (Bajo sub-grave con cola.)
- **I_chord(C,t,root,g=0.3)**: acorde menor 7 = intervalos `[0,3,7,10]`; cada nota = 2 saws detuneados ±7 cents por lowpass que cierra 1400→700 Hz; envolvente 0.7 s; reverb 0.16.
- **I_rhodes(C,t,note,g=0.3)**: **FM** — portadora seno a `f`, moduladora seno a `2f`, índice de modulación que decae `f*1.2 → f*0.15` en 0.4 s (timbre de campana/Rhodes); lowpass 3000; envolvente 0.6 s; reverb 0.18.
- **I_upright(C,t,note,g=0.5)** (contrabajo): triangular + seno detuneado −4 cents; lowpass que cierra 900→350 Hz; ataque de dedo = ruido bandpass `f*4` muy corto; envolvente 0.45 s.
- **I_piano(C,t,note,g=0.32)**: **aditivo** — 7 parciales seno con pesos `[1, .5, .32, .18, .1, .06, .04]`, leve inarmonicidad (`detune = mult*0.5` cents); martillo = ruido bandpass `f*4` (0.015 s, `g*0.3`); lowpass 5500→2200 Hz; envolvente con ataque 0.006 s, caída a `g*0.3` en 0.5 s y a 0 en 1.6 s; reverb 0.16.
- **I_violin(C,t,note,g=0.26)**: 3 saws detuneados (−5/+6/0 cents) con **vibrato** (LFO 5.5 Hz, profundidad `f*0.008`); dos formantes peaking (550 Hz +6dB Q1.2; 1100 Hz +4dB Q1.5); lowpass 4200; envolvente de arco (ataque lento 0.12 s, sostenido, release a 0 en ~0.95 s); reverb 0.28.
- **I_flute(C,t,note,g=0.3)**: seno + triangular a `2f` (gain 0.12); vibrato LFO 5 Hz profundidad `f*0.006`; **soplo de aire** = ruido bandpass `f*1.5` con envolvente 0.18 s a `g*0.18`; envolvente ataque 0.06 s, sostenido, release a 0 ~0.8 s; reverb 0.3.
- **I_lead(C,t,note,g=0.24)**: cuadrada por lowpass 2400 (Q2); envolvente 0.22 s; reverb 0.18.
- **I_shaker(C,t,g=0.3)**: ruido highpass 6000 Hz, envolvente 0.05 s.
- **I_rim(C,t,g=0.5)** (rimshot): triangular 420 Hz muy corta + ruido bandpass 2500 (Q2) corto.
- **I_conga(C,t,freq=210,g=0.55)**: seno con glide `freq*1.3→freq`, envolvente 0.14 s; pizca de ruido bandpass `freq*2`; reverb 0.12. (En el secuenciador se llama con tono fijo 210 Hz, no melódico.)
- **I_tom(C,t,freq=180,g=0.7)**: seno con glide `freq*1.4→freq`, waveshaper suave, envolvente 0.18 s; reverb 0.15. (Solo se usa en los fills.)
- **makeCrackle(C,t,dur)** (vinilo): AudioBuffer de ruido donde cada muestra es, con probabilidad 0.00025 un chasquido `(rand*2-1)*0.35`, si no un piso muy bajo `(rand*2-1)*0.006`; pasa por highpass 2500 + lowpass 7000; gain 0.7. (Valores ya "suavizados".)

---

## 5. Estado de la aplicación

```
VOL[id]            // ganancia 0..1.5 por instrumento (default 1) + VOL['crackle']=0.5
variants[]         // [{ name, patt }]   patt[id] = { on:[16 bool], notes:[16 int], mute:bool }
curVar             // índice de la variante mostrada
arrangement[]      // [{ v: índiceVariante, reps: int }]
bpm, swing         // números
fillsOn, fillEvery // bool, int (cada cuántos compases entra un fill)
crackleOn          // bool
SAMPLES[id]        // { buf:AudioBuffer, b64:string, name:string, root?:int, vol?:number }
                   //   id de instrumento → reemplaza síntesis;  id "loop:<nombre>" → loop paralelo
gallery[]          // [{ name, state }] proyectos guardados en memoria
```

`emptyPattern()` crea un `patt` con todos los instrumentos en `on:false`, `notes` lleno con la base del instrumento, `mute:false`.

---

## 6. Secuenciador (UI principal)

- Una fila por instrumento. Estructura de cada fila:
  - **Etiqueta** de ancho fijo (212 px, no se encoge — esto es clave para que las columnas de pasos queden alineadas entre todas las filas): swatch de color, nombre (ancho fijo 54px con elipsis), botón **M** (mute), **deslizador de volumen** (range 0–1.5, paso 0.05, estilizado y visible sobre fondo oscuro), botón **⊕** de carga de sample, y —solo si el instrumento es melódico y tiene sample cargado— un **selector de nota raíz** (celeste).
  - **Grilla de pasos**: `display:grid; grid-template-columns:repeat(16,1fr); gap:3px`. Celdas cuadradas. La celda en posición `4n+1` tiene fondo más claro (marca de pulso). Celda activa = color del instrumento con glow.
  - Para instrumentos **melódicos**, debajo de la grilla va una **fila de notas** (`note-row`) con la misma etiqueta de 212 px y un grid de 16 `<select>`; cada select solo es visible si el paso correspondiente está activo, y ofrece las notas `SCALE` sobre la base del instrumento (texto = `midiName`).
- **Interacción**:
  - Clic en celda → alterna el paso; si lo activa, reproduce un preview del sonido.
  - Cambiar un select de nota → guarda la nota en `patt[id].notes[step]` y preview.
  - Botón **M** → alterna `mute`.
  - Deslizador → actualiza `VOL[id]` y, si suena, `liveChain.bus[id].gain` en tiempo real.
- **Resaltado de reproducción** (playhead): la columna del paso que suena se marca; se aplica con un `setTimeout` sincronizado al tiempo real del audio.

---

## 7. Presets

Botones que cargan un patrón en la **variante actual** (`applyPreset(name)` limpia y rellena). Hay 9 presets de patrón + "Vaciar". Etiquetas y comportamiento:

| botón | name | descripción |
|-------|------|-------------|
| Clásico | classic | boom-bap base: kick [0,7,8], snare [4,12], hats en pares, 808 + acordes |
| Head-nod | laid | kick sincopado [0,6,8,11], swing marcado |
| Hard / dramático | hard | doble kick [0,3,8,11], open hats, más oscuro |
| Doble snare | jazzy | hats en semicorcheas, 808 activo |
| Jazz-hop | rhodes | Rhodes protagonista + contrabajo + shaker, sin 808 |
| Dusty soul | dusty | contrabajo + Rhodes + rimshot (en vez de snare) + conga |
| Latino | latin | congas y shaker al frente, contrabajo saltón |
| Trío jazz | trio | walking bass en contrabajo + Rhodes + rimshot tipo escobillas |
| Cámara (piano/violín/flauta) | camara | piano acordes + violín sostenido + flauta melodía |
| Vaciar | clearpat | limpia el patrón actual |

**Regla de notas**: toda nota usada en cualquier preset debe pertenecer a `SCALE` sobre la base del instrumento (validar). Ejemplos representativos de patrones (paso entre 0–15; nota MIDI):

- *classic*: kick {0,7,8}; snare {4,12}; hat {0,2,4,6,8,10,12,14}; ohat {14}; e808 {0:33,7:33,8:38,10:36}; chord {0:45,8:52}.
- *trio*: kick {0,10}; rim {4,12}; hat {2,6,10,14}; upr (walking) {0:33,2:36,4:38,6:40,8:41,10:40,12:38,14:36}; rhodes {0:57,3:60,6:64,8:62,11:65,14:60}.
- *camara*: kick {0,8}; rim {4,12}; hat {2,6,10,14}; shaker {0,4,8,12}; upr {0:33,4:40,8:36,12:38}; piano {0:48,2:55,8:53,10:60}; violin {0:60,8:59}; flute {2:72,5:76,8:79,11:76,14:72}.

(El modelo puede componer los demás respetando el espíritu de cada estilo y la regla de escala.)

---

## 8. Variantes y arreglo

- **Variantes**: pestañas sobre el secuenciador. Botón "+ Nueva" agrega una variante (nombre A, B, C… por letra) con patrón vacío. Cada pestaña tiene una ✕ para borrarla (si hay más de una). Cambiar de pestaña cambia `curVar` y redibuja.
- **Arreglo**: una línea de tiempo de bloques. Botones "+ A / + B / …" agregan un bloque de esa variante con `reps` (campo numérico 1–16). Cada bloque muestra `Nombre ×reps` y una ✕ para quitarlo. Color del bloque según la variante.
- **Reproducir arreglo** (botón "▶ Tocar arreglo"): expande el arreglo a una secuencia de patrones (un patrón por repetición) y los reproduce en orden; al terminar, para. Mientras suena:
  - se **resalta el bloque** actual en la línea de tiempo (borde brillante);
  - el **secuenciador cambia a la pestaña** de la variante que suena (solo cuando cambia, no en cada compás);
  - el texto de estado muestra "▶ Arreglo · variante X".

---

## 9. Transporte (barra superior)

- **▶ Play**: reproduce en loop la variante actual.
- **■ Stop**: detiene todo, limpia resaltados, detiene crackle y loops.
- **BPM**: input numérico.
- **Swing**: range 0–60, muestra el %.
- **Fills**: botón On/Off + campo "cada N comp." (2–16). Cuando está On, en el **último compás de cada ciclo de N** la batería normal (`kick,snare,hat,ohat`) se **reemplaza** por un fill elegido al azar entre 4 tipos; los instrumentos melódicos siguen sonando. Tipos de fill: redoble de caja con crescendo; toms descendentes (usa I_tom, mantiene kick en 1); mitad caja mitad toms; corte seco con kick final. Detección: `isFillBar(barIdx) = fillsOn && ((barIdx+1) % fillEvery === 0)`.
- **Vinyl**: botón On/Off. Activa el crackle de fondo (continuo, loopea en vivo, cubre toda la duración en el export). Bus de volumen propio (`'crackle'`, default 0.5).
- **● Exportar WAV**: ver §10.
- **? Ayuda**: abre un overlay modal con instrucciones (cerrable por botón ✕, clic fuera, o tecla Escape).

---

## 10. Exportar a WAV (render offline)

1. Determinar la secuencia: si hay arreglo, expandirlo (un patrón por repetición); si no, usar 2 repeticiones de la variante actual.
2. `dur = totalSteps * stepDur + 1.0` (cola). Crear `OfflineAudioContext(2, ceil(44100*dur), 44100)` y una cadena con `buildChain(off)`.
3. Recorrer cada bloque (= compás) y cada paso. Tiempo de cada paso: `t_bloque + step*stepDur + (step impar ? stepDur*swing : 0)`. **Importante**: el tiempo debe incluir `step*stepDur` (un bug clásico es omitirlo y que todos los golpes caigan al inicio del compás).
4. Para cada instrumento activo: si tiene sample cargado, reproducir el sample (afinado por nota si es melódico, ver §12); si no, llamar a su función de síntesis. Respetar fills (reemplazo de batería) e `isFillBar`.
5. Agregar el **crackle** (si on) cubriendo toda la duración, y los **loops** en paralelo (repetidos hasta cubrir `dur`), cada uno a su volumen.
6. `await off.startRendering()`, codificar el AudioBuffer a WAV PCM 16-bit (header RIFF/WAVE estándar de 44 bytes + data), crear un Blob `audio/wav` y disparar descarga `boombox_base_<bpm>bpm.wav`.

Incluir una función `encodeWAV(audioBuffer)` que escriba el header RIFF correctamente (chunks `RIFF`, `WAVE`, `fmt ` PCM mono/estéreo 16-bit, `data`).

---

## 11. Compatibilidad iOS / desbloqueo de audio

- Crear `AC = new (AudioContext||webkitAudioContext)()`.
- En el primer gesto del usuario (`touchend`/`mousedown`/`click` en `document`), llamar `AC.resume()` y reproducir un buffer silencioso de 1 muestra para "destrabar" el audio en Safari iOS. Hacerlo una sola vez.
- En `startLoop`/`startArrange`, reasegurar `AC.resume()`.

---

## 12. Samples del usuario

- Botón **⊕** por fila: `<input type="file" accept="audio/*">`. Al cargar: leer como ArrayBuffer, `decodeAudioData` → `buf`, y convertir a base64 → `b64`. Guardar en `SAMPLES[id] = {buf, b64, name}`.
- Si el instrumento es **melódico**, al cargar asignar `root = base` del instrumento y mostrar el **selector de nota raíz** (rango ~C2–C6) para indicar en qué nota está grabado el sample.
- **Reproducción de sample melódico**: `playbackRate = 2^((notaObjetivo - root)/12)` (afina el sample subiendo/bajando velocidad). La duración efectiva se divide por el rate.
- **Reemplazo**: en `triggerIns`, si existe sample para el id, usarlo en vez de la síntesis (pasando la nota si es melódico, `null` si es percusión). Volver a clic en ⊕ (cuando está cargado) quita el sample.
- **Loops/Breaks**: sección aparte. Botón que carga uno o varios archivos como `SAMPLES['loop:'+nombre] = {buf,b64,name,vol:0.7}`. Cada loop se lista con su duración, un deslizador de volumen y una ✕. En reproducción corren en paralelo (loop continuo en vivo; repetido hasta cubrir `dur` en el export). Cada loop a su `vol`.

---

## 13. Proyectos (galería + archivos .json)

- **Barra de proyectos** (arriba): desplegable de la galería, botones "Cargar", "Guardar actual", "Eliminar", "⬇ Descargar .json", "⬆ Importar .json".
- `captureState()` serializa: `bpm, swing, fillsOn, fillEvery, crackleOn, vol (copia de VOL), variants (name+patt), arrangement, samples` (solo `{b64,name,vol,root}` por id; **no** el AudioBuffer).
- `restoreState(st)` aplica todo, reconstruye las variantes asegurando que existan todos los instrumentos, reconstruye los AudioBuffers de los samples desde base64 (`decodeAudioData`), refresca la UI y los selectores de la barra de transporte.
- "Guardar actual" pide un nombre (`prompt`) y agrega `{name, state:captureState()}` a `gallery`.
- "Descargar .json" exporta `{app:'boombox', version:1, projects:gallery}` como Blob descargable (`boombox_proyectos.json`). Si la galería está vacía, exporta la base actual.
- "Importar .json" concatena los proyectos del archivo a la galería (acepta tanto `{projects:[...]}` como un proyecto suelto `{state:...}`).

---

## 14. Bases precargadas

Al iniciar, la galería arranca con **9 bases originales** (en este orden) y se carga la primera. Cada una define BPM, swing, fills, vinyl, volúmenes y un arreglo de 2 variantes (A y B). Son originales (no reproducen temas existentes). Estilos:

1. **Golden — boom-bap** (90 BPM): dorado clásico, contrabajo + Rhodes + vinyl.
2. **Grimy — NY crudo** (87 BPM): pesado y oscuro, 808, fills cada 4, vinyl.
3. **Smooth — head-nod** (86 BPM): relajado, Rhodes + contrabajo + conga, sin fills.
4. **Punch — enérgico** (93 BPM): doble kick, hats activos, fills cada 4.
5. **Battle — oscura** (88 BPM): agresiva, 808 al frente, poco swing.
6. **Cypher — boom-bap** (90 BPM): clásico de cyphers, swing 26%, fills cada 8, vinyl.
7. **Soul — emotiva** (84 BPM): melódica, Rhodes protagonista, rimshot suave.
8. **Hybrid — trap-bap** (140 BPM): híbrido enérgico, 808 fuerte, hats rápidos.
9. **Dusty — jazz trío** (82 BPM): jazzy, walking bass, escobillas, vinyl.

Usar un helper `mkPatt(drums, mel)` (recibe `{id:[pasos]}` y `{id:[[paso,nota]]}`) y `mkBase(name, opts)` para construirlas de forma compacta. Todas las notas deben respetar `SCALE`.

---

## 15. Paleta y estilo visual

Tema oscuro. Variables CSS:

```
--bg:#0d0f14;  --panel:#151823;  --panel2:#1b1f2d;  --ink:#f0ece3;  --muted:#7d8296;
--hot:#ff5a3c; --gold:#ffc24b;   --cool:#46c8ff;    --green:#3ddc97; --violet:#b388ff;
--line:#272c3a; --grid:#10131b;  --cell:#222838;    --beat:#2c3346;
```

- Tipografía: `Inter, system-ui, sans-serif`. Títulos con degradado `hot→gold` en el texto.
- Botones redondeados, hover sutil. Botones de acción primaria en verde; "rec/exportar" en rojo (`--hot`); botones On activos en celeste (`--cool`).
- Deslizadores de volumen **estilizados** (track gris `#3a4258`, perilla dorada con borde oscuro) para ser visibles sobre el fondo. Estilizar `::-webkit-slider-thumb`, `::-moz-range-thumb` y `::-moz-range-track`.
- Layout centrado, ancho máx ~1100 px. La barra de transporte es sticky arriba.
- Overlay de ayuda: panel centrado con backdrop translúcido y blur.

---

## 16. Comportamiento del scheduler (en vivo)

- Reloj de "look-ahead": un `setTimeout` cada ~25 ms que agenda los pasos cuyo tiempo cae dentro de `AC.currentTime + 0.12`.
- Avanza `curStep` 0→15; al volver a 0 incrementa `barCount`. En modo arreglo, al completar un patrón avanza `seqPos`; al pasarse del final, detiene.
- Cada paso agenda los instrumentos activos (vía `triggerIns`), aplica swing a los pasos impares, dispara fills cuando corresponde, y programa el resaltado visual con `setTimeout` al tiempo real.

---

## 17. Criterios de aceptación (checklist)

La reconstrucción es correcta si:

- [ ] Abre como un solo `.html` sin errores de consola y sin red.
- [ ] Se oye audio tras el primer clic (incl. iOS).
- [ ] Los 15 instrumentos suenan y se programan en la grilla; las columnas de las 16 posiciones quedan **alineadas verticalmente** entre todas las filas.
- [ ] Los melódicos muestran selector de nota por paso activo, limitado a la escala.
- [ ] Los 9 presets cargan patrones coherentes y todas sus notas están en escala.
- [ ] Variantes: crear, borrar, cambiar de pestaña y editar por separado.
- [ ] Arreglo: encadenar bloques con repeticiones; al tocarlo se resaltan bloque y pestaña en sincronía con el audio.
- [ ] Swing atrasa las corcheas; BPM cambia la velocidad.
- [ ] Fills On reemplazan la batería del último compás de cada N, variados al azar.
- [ ] Vinyl agrega crackle suave de fondo.
- [ ] Volumen por instrumento en vivo y reflejado en el export.
- [ ] Cargar sample en una fila reemplaza el sonido; en melódicos, el sample se **afina** según la nota y la nota raíz indicada.
- [ ] Loops corren en paralelo y se incluyen en el WAV.
- [ ] Exportar WAV produce un archivo válido cuyo contenido coincide con lo que suena (golpes distribuidos en el tiempo, no amontonados).
- [ ] Guardar/Descargar/Importar `.json` preserva todo el estado, incluidos samples (base64) y nota raíz.
- [ ] Arranca con 9 bases precargadas y carga la primera.
- [ ] Botón de Ayuda abre un overlay con instrucciones, cerrable por ✕/clic afuera/Escape.

---

## 18. Sugerencia de orden de implementación

1. Esqueleto HTML + CSS (paleta, layout, transporte, secuenciador vacío).
2. Motor de audio: `buildChain`, helpers, funciones de instrumento.
3. Estado + `emptyPattern` + render del secuenciador + interacción de celdas.
4. Scheduler en vivo + Play/Stop + swing + BPM.
5. Variantes + presets.
6. Arreglo + reproducción de arreglo + resaltados.
7. Fills + vinyl.
8. Volumen por instrumento.
9. Export WAV (cuidando `step*stepDur`).
10. Samples (instrumento + nota raíz + afinación) y loops.
11. Proyectos (galería + .json).
12. Bases precargadas.
13. Overlay de ayuda.
14. Repasar el checklist §17.

> **Nota de alcance/sonido.** Es una base electrónica boom-bap sintetizada de buen nivel; los timbres acústicos (piano, violín, flauta, contrabajo) son aproximaciones por síntesis y se notarán "sintéticos" frente a samples reales. Esto es esperado y aceptable; el usuario puede cargar samples propios para mayor realismo.
