# Bitácora del refactor de latencia — decoder → oráculo por estado

> Registro íntegro de decisiones, causas y lógicas de la optimización de latencia
> del decoder (Sujeto VIII — call_me_maybe): desde el pre-índice de vocabulario
> hasta el oráculo por estado (Modelo A). Documento de cabecera de la refactorización:
> integra, contextualiza y cruza los demás docs (`CONTEXTO_REFACTOR.md`,
> `ANEXO_OPTIMIZACION_LATENCIA.md`, `BITACORA_BUGS.md`, `ESTADO_ACTUAL.md`).
>
> Última actualización: 2026-09-25 noche.

---

## 0. Resumen ejecutivo

El cuello de botella del pipeline era el **forward del modelo sin KV-cache**
(~2.6 s en esta CPU, i7-7700HQ): cada token que el decoder deja pasar cuesta un
forward completo, y cada forward reprocesa todo el contexto. Con ~30-40 forwards
por prompt, la suite de 11 prompts (Task 4.3) tardaba **34.9 min** en la corrida original (18/09; 35.0 con carga).

La estrategia fue **sacar el cómputo determinista del modelo**: todo lo que es
predecible desde el estado (estructura JSON, keys, comas, cierres) se inyecta
estáticamente SIN forward. Eso produjo una cascada de tres fases — pre-índice,
header estático (Opt2) y oráculo por estado — que llevó la suite de la corrida original (34.9 min) a
**7.6 min** (-78%), con forwards reducidos de ~638/314 a **133** y la estructura a 0
forwards.

| Hito | Suite completa | P2 aislado | P8 aislado | Forwards |
|---|---|---|---|---|
| Original Task 4.3 (18/09) | 34.9 min | — | — | — |
| Ref pre-refactor (23/09) | 27.6 min | 113.2 s | 242.5 s | ~638 |
| Opt2 header estático (24/09) | 15.1' (-45%) | 48.5 s (-57%) | 125.9 s (-48%) | 314 |
| Oráculo Nivel 1 (25/09) | **7.6' (-50% sobre Opt2)** | **16.8 s (-85%)** | **80.4 s (-67%)** | **133** |

Dejó documentado además el **límite físico** del KPI (<5 min) en esta CPU
(sección 6): con la estructura a 0 forwards, el residuo son los *values libres*
que el modelo elige — irreductibles sin semántica del contenido ni KV-cache
(ambos fuera del subject).

---

## 1. El problema original (causa raíz de toda la refactorización)

- **Forward sin KV-cache**: el SDK local (`llm_sdk`) no cachea los KV-states;
  cada llamada de extensión reprocesa el contexto completo desde cero.
  Medido: **2.6 s por forward** con threads=4 (óptimo: threads=6 → 28x PEOR,
  bandwidth-bound — benchmark de scaling 23/09).
- **Precedentes medidos antes del refactor (para la serie histórica)**:
  - 17-18/09 (Task 4.2, smoke con el modelo real): `generate()` completo
    **~308 s para UN solo prompt** · forward real **2.842 s/step** @32 tokens
    · proyección 11 prompts ≈ **55+ min** (incumplimiento >10x del KPI).
  - 18/09 (Task 4.3 ORIGINAL, 6 vCPU/15Gi, threads=4): **34.9 min de
    generación** (35.0 con carga; por prompt: 122-330 s, el más caro = regex
    larga de substitute) · accuracy fn 11/11 (100%) · full 10/11 (91%, con
    scoring que relajaba equivalentes funcionales) · KPI a **~7x**.
  - 23/09 (ref pre-refactor, ya con pre-índice Opt1): **27.6 min** (1656.4 s),
    **~638 forwards (58/p)**, KPI ~5.5x — es la corrida que sirvió de
    baseline contra Opt2 (tabla de la sección 0).
  - Fuentes: `docs/tracking/PROGRESS_TRACKER.md` (commits `35b359d`, `2d95ef2`)
    y `docs/tracking/ESTADO_ACTUAL.md` (versiones 24/09, commits `80bf441`,
    `2da476b`).
- **Consecuencia**: el costo total ≈ forwards × 2.6 s + overhead. Reducir
  **forwards** es la única palanca real de latencia (el costo por forward es
  fijo e intocable: no se puede tocar `llm_sdk`).
- **Criterio del subject**: suite de 11 prompts < 5 min en CPU real + bar M14
  (accuracy full ≥ 90%).
- **Estrategia derivada**: *"todo lo determinista no se pregunta al modelo"* —
  si el próximo texto se puede derivar del estado + esquema, se inyecta sin
  forward. De ahí salieron las tres fases.

## 2. Fase Opt1 — pre-índice de vocabulario (`tokens_starting_with`)

- **Qué**: índice en `Vocab` que agrupa las ~151k entradas por 1er carácter
  decodificado (`id2decoded` + `tokens_starting_with`).
- **Causa**: el filtro del vocabulario (qué tokens son válidos dado el estado)
  recorría linealmente el vocab completo en cada decisión — costo O(|V|) por
  paso, significativo a 2.6 s por forward.
- **Lógica**: el prefijo actual (p.ej. `"shre`→ solo tokens que empiezan con
  `k`) está contenido en `tokens_starting_with[k]` — la selección de candidatos
  pasa de O(|V|) a O(candidatos).
- **Resultado**: ya estaba integrado ANTES de esta refactorización (base de
  `Opt0`); se mantuvo invariante durante todo el proceso.

## 3. Fase A — header estático (`Opt2`, commit `d6592d0`)

- **Observación**: los primeros ~20 tokens de todo output son la misma
  estructura: `"\n\n{\n  \"name\": \""` + el nombre de la función + todo el
  armado de `"parameters": {` — sintaxis 100% determinista, el modelo solo
  elige el value del name.
- **Qué**: `STATIC_HEADER` (byte-exacto) inyectado con `encode()` del SDK
  **sin forward** + hook de métricas por `DecoderPhase`.
- **Medido (24/09)**: suite 15.1' (**-45%** vs original), P2 48.5 s (-57%),
  P8 125.9 s (-48%), forwards 314. KPI <5' a 3x.
- **BUG-012 (lección fundacional de byte-exactness)**: el header debe ser
  byte-exacto al formato natural del modelo — **incluidas las 2 newlines
  iniciales antes del `{`**. Cortarlas (por "limpieza") rompía el caso
  "Greet shrek". Lección generalizada: cualquier inyección estática debe ser
  prefijo exacto de lo que el modelo habría emitido (`emitted + inyectado =
  prefijo del canónico`).
- **Lógica del siguiente paso**: quedaba un 2º tramo natural — entre el cierre
  del value de `"name"` y la apertura de `"parameters": {` — mismo patrón:
  determinista, ¿por qué forwardearlo?

## 4. Fase B1 — 2º tramo lineal (`STATIC_TAILS` + `_match_static_tail`)

- **Qué**: tramo fijo `,\n  "parameters": {` disparado por trigger lineal al
  detectar el cierre del value de `name`.
- **Medido (aislado, 25/09 madrugada)**: P8 56→49 forwards (-7 exactos) ✅;
  **P2 CORRUPTO** → **BUG-013**.
- **Causa raíz de BUG-013**: el trigger `current_key=="name"` es **ambivalente**:
  al terminar de emitir el value de `name` (del ROOT), el decoder detecta el
  cierre **también** para el param interno `"name"` de `fn_greet` — el estado
  `current_key` queda residual al bajar de depth y el trigger matchea donde no
  debe → JSON corrupto (con `success=true`, el peor tipo de fallo).
- **Descartado** (derivable): `current_key=="name"` — no es derivable porque
  el caso sin parámetros (`"parameters": {}`) deja `current_key=="parameters"`
  y el param interno `name` lo vuelve ambiguo.
- **Conclusión**: el trigger LINEAL por key no sirve — el gate correcto necesita
  más estado (depth, keys pendientes). Eso generaliza a un **oráculo por estado**.

## 5. Fase B2 — oráculo por estado (Modelo A, Nivel 1) — implementado 25/09

### 5.1 Diseño (fundamentos y lógica)

El oráculo es una **tabla de tramos keyed por estado completo**: dado
`(estado actual, depth, gates)`, decide si hay texto estático inyectable y
cuál. Reemplaza al trigger lineal.

- **E (ya emitido)**: `E = _trailing_ws(emitted)` — el whitespace/sintaxis que
  el modelo YA emitió en tokens anteriores (solo en fases ciegas:
  VALUE_END, IN_OBJECT, PARAMS_OBJECT, KEY_END, COLON). Evita duplicar
  sintaxis que llegó fusionada en un token BPE (heredero de BUG-012).
- **K (próxima key canónica pendiente)**: `ord = tuple(F.parameters)` (orden de
  inserción de la definición — respetado por decisión explícita y testeado),
  `R = required_keys_remaining()`, `Knext = primer k de ord con k ∈ R`.
- **`align(canon, emitted)`**: `canon[len(E):]` si `startswith(E)`, sino `None`.
- **Dominios disjuntos por construcción** (unicidad del tramo) + **minimalidad**
  (`emitted + inyectado = prefijo del canónico`): 1 sola respuesta correcta por
  estado, sin ambigüedad — elimina de raíz la clase de bug BUG-013.

### 5.2 Contrato de la tabla (Nivel 1, cerrado el 25/09)

Definitions: `WS = " \t\n\r"` · `OP(k) = '"' si type=="string" sino ''` ·
`KEY(k) = '"'+k+'": '+OP(k)` · `ENTRY(k) = '\n    ' + KEY(k)` · `ρ = R ≠ ∅`.

| Tramo | Estado | Gates | Texto |
|---|---|---|---|
| T1 | VALUE_END, depth 0 | **N**=1 (`depth==0 ∧ current_key=="name" ∧ keys_enclosed==∅`) | `',\n  "parameters": {' + ENTRY(K1)` |
| T2 | IN_OBJECT, depth 0 | N=1 | `'\n  "parameters": {' + ENTRY(K1)` |
| T3 | PARAMS_OBJECT, depth 1 | ρ=1 | `ENTRY(Knext)` |
| T4 | VALUE_END, depth 1 | ρ=1 | `',\n    ' + KEY(Knext)` |
| T5 | VALUE_END, depth 1 | ρ=0 | `'\n  }\n}'` |
| T6 | VALUE_END, depth 0 | N=0 ∧ ρ=0 | `'\n}'` |

- **Gate N** reemplaza a `¬has_seen_params_object()` de la propuesta original;
  verificación contra `state.py` mostró que (5.3.a) el flag sticky no es
  derivable en la dirección segura.
- **`ord = ∅`** (función sin parámetros): T1/T2 → `None` — el modelo emite el
  `"parameters": {}` fusionado (decisión usuaria para evitar la costura
  BUG-012); el ROOT lo cierra T6.

### 5.3 Correcciones sobre la marcha (verificadas contra la state machine real)

- **(a) P NO puede gatear T6 — hallazgo del probe fn_empty**: la flag
  `has_seen_params_object()` es **sticky** y solo se enciende si el token
  termina en PARAMS_OBJECT (`schema.update()` corre con el estado
  POST-token). El token BPE fusionado `"parameters": {}` trae `{` y `}` en
  UN tocho → la flag queda `False` en el caso real → T6 exigiéndola nunca
  respondía → el `}` del ROOT faltaba (probe: SUCCESS=False). **Solución**:
  con `N=0 ∧ ρ=0` el ROOT solo puede cerrarse (el flanco del `}` prematuro lo
  cubre el pase fino del camino normal, no el oráculo) → T6 prescinde de P.
  => El `}` final que faltaba lo inyecta T6: **SUCCESS=True** (25.1 s,
  output byte-exacto `'\n\n{\n  "name": "fn_empty",\n  "parameters": {}\n'`).
- **(b) `VALUE_END + ',' → PARAMS_OBJECT` (NO `IN_OBJECT`)**: en este decoder,
  la coma post-value cierra el value y aterriza en PARAMS_OBJECT. Reparto
  T3/T4 resultante: los values NUMBER dejan `IN_NUMBER_VALUE` ABIERTO hasta el
  siguiente char → el post-coma real de un number cae a T3 (PARAMS_OBJECT);
  T4 (VALUE_END d1) cubre solo values que cierran en su propio token (strings).
- **(c) Fases ciegas para `E`**: VALUE_END, IN_OBJECT, PARAMS_OBJECT, KEY_END,
  COLON (definidas en `_WS_BLIND_PHASES`).

### 5.4 Implementación (working tree, SIN commitear — 25/09)

- `src/decoder/constrained_generator.py`: `_WS`, `_WS_BLIND_PHASES`,
  `_trailing_ws`, `_align`, `_gate_n`, `_op`, `_key`, `_entry`, `_knext`,
  `_tramp_t1`…`_tramp_t6`, `_TRAMPS`, `_next_static_text(state, schema, emitted)`.
  `STATIC_TAILS`/`_match_static_tail` **eliminados**. `generate()` acumula
  `emitted` en el loop (sin tocar `state.py` ni `schema_validator.py`);
  `_inject_static_header(..., emitted_parts: list[str] | None = None)`.
- `tests/test_constrained_generator.py`: `TestLevel1Oracle` (canónicos por
  tramo con `state.simulate(C)` como fuente de verdad, dominios disjuntos,
  regresión BUG-013, insertion order, align E), `TestInjectOracleText`,
  `TestOracleEndToEnd` (fn_add_numbers 9 fwd; fn_empty 7 fwd). Suite total:
  **195 tests GREEN** · flake8 0 · mypy Success.
- Niño: resolver BUG-013 cerrado sin volver a abrir BUG-012/BUG-011.

## 6. Mediciones y límite físico del KPI (25/09 noche)

### 6.1 Suite completa (Task 4.3, modelo real, threads=4, warm-up descartado)

| Métrica | Opt2 (24/09) | Oráculo N1 (25/09, c1) | Oráculo N1 (25/09, c2 governor libre) |
|---|---|---|---|
| Timing generación | 15.1' | 7.6' | **6.5'** |
| Forwards totales | 314 | **133** | **133** |
| Costo por forward | — | 3.42 s | **2.93 s (-14%)** |
| Accuracy fn | 100% | **100%** | **100%** |
| Accuracy full (M14) | 82% | **82%** | **82%** (P9/P10 semántico) |
| KPI <5' | 3x | incumplido | **~1.3x** |

> Corrida 2 (25/09 noche): el usuario quitó la limitación de performance
> (ahorro de batería) del entorno. La máquina no expone cpufreq (entorno
> virtualizado) — la limitación quedó como variable no visible, pero el efecto
> se mide en el timing: mismos 133 fwd, -14% de costo por forward (el grueso
> en IN_STRING_VALUE: 372.3 s → 303.1 s).

Desglose por fase (solo 3 fases con forwards — el resto a 0):

| Fase | Forwards | Tiempo | Naturaleza |
|---|---|---|---|
| IN_STRING_VALUE | 114 | 372.3 s (82%) | values libres + fn_names |
| IN_NUMBER_VALUE | 13 | 58.3 s | numbers del modelo |
| COLON | 6 | 23.9 s | tokens fusionados (residuo) |

### 6.2 Costo real y piso físico

- Costo real: **2.93 s/forward** con governor libre (389.1 s / 133); 3.42 s en
  la corrida 1 (limitación de batería activa, ~14% de penalización).
- Palancas restantes: **B′** (~31 fwd de fn_name → ~97 fwd → **~4.7-5.5'**
  según governor) y **Nivel 2** (T7–T10, tokens fusionados — ataca los 6
  COLON + colas de values). Juntas: piso real **~5.0'** con governor libre.
- **Conclusión**: con la estructura a 0 forwards, el residuo son los ~96
  *values libres* (string + number) que el modelo decide — irreductibles sin
  semántica del contenido ni KV-cache. **El KPI <5' queda AL FILO** con
  governor libre + B′ (~4.7' estimado); sin B′ el techo honesto de la
  optimización estructural es ~5.5'.

## 7. Palancas diferidas y decisiones pendientes

### 7.1 B′ — autocompletar `fn_name` por unicidad del trie (validado, diferido)

Cuando el prefijo emitido del fn_name es unívoco en el trie de funciones, el
resto es determinista → inyectar sin forward. Umbrales probados (25/09):

| Función | Único desde | Resto a inyectar |
|---|---|---|
| `fn_add_numbers` | `fn_a` | `dd_numbers"` |
| `fn_greet` | `fn_gr` | `eet"` |
| `fn_reverse_string` | `fn_r` | `everse_string"` |
| `fn_get_square_root` | `fn_ge` | `t_square_root"` |
| `fn_substitute_string_with_regex` | `fn_s` | `ubstitute_string_with_regex"` |

**Ojo**: `fn_g` NO es unívoco (compartido entre `fn_greet` y
`fn_get_square_root`) — la unicidad la decide el trie con el prefijo emitido,
no una posición fija. Diferido hasta decidir la estrategia KPI (7.3).

### 7.2 Nivel 2 (T7–T10 — tokens fusionados, diferido)

Cubre los casos donde la sintaxis llega fusionada en un token del modelo
(p.ej. los 6 forwards COLON residuales, `": "` con apertura de value). Solo
tiene sentido tras B′ y según la decisión KPI (aporte marginal ~1-2%).

### 7.3 Decisiones que requieren usuario

- **Estrategia KPI**: (a) implementar B′ igual (7.6' → ~5.5', deja el proyecto
  en su piso físico), (b) aceptar el KPI como métrica por prompt / documentar
  el techo de CPU en la negociación, (c) hardware. *Recomendación técnica:
  B′ sí — es la última palanca de forwards y deja medido el límite real.*
- **Scoring M14 (task 7)**: P9 `regex='([0-9]+)'` y P10 `([aeiouAEIOU])` +
  `replacement='****'` — el modelo produce regex funcionalmente equivalentes
  con grupo de captura y reemplazo semánticamente correcto. Decidir relajar
  (equivalencia funcional) vs exigir (byte-exacto).

## 8. Referencias cruzadas (mapa de la refactorización)

- `docs/design/CONTEXTO_REFACTOR.md` — estrategia general; §2.5 = FASE 2
  oráculo por estado (estado implementado).
- `docs/design/ANEXO_OPTIMIZACION_LATENCIA.md` — decisiones detalladas D1–D9
  (incl. registros de la fase oráculo, probing de formato, refutación del
  "40% titubeo", refutación B5 etc.).
- `docs/tracking/BITACORA_BUGS.md` — BUG-011 (escapes en name, resuelto
  `794a470`), BUG-012 (byte-exactness del header, resuelto `d6592d0`),
  BUG-013 (trigger viciado, resuelto 25/09 por oráculo).
- `docs/tracking/ESTADO_ACTUAL.md` — estado dinámico + pendientes.
- `docs/RESUMEN_EJECUTIVO_NEGOCIACION_PLAZOS.md` — serie histórica de métricas
  (con fechas) + gráfica de evolución, para la negociación de extensión.
- `data/output/metrics_run.json` + `metrics_run.log` — métricas por fase.
- `~/scratch/call_me_maybe_task42/` — task43_accuracy.py (suite),
  bench_p8_vs_p2.py (bench aislado), vocab_study.py (estudio de tokens).
- Código: `src/decoder/constrained_generator.py` (oráculo, working tree),
  `src/decoder/state.py` + `src/decoder/schema_validator.py` (intocados).