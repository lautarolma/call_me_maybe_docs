# ESTADO_ACTUAL — call_me_maybe (42 School)

> Archivo de estado DINÁMICO — importado con `@` desde `CLAUDE.md`. Se actualiza al inicio/fin de cada sesión. TODO lo que cambia entre sesiones va acá; CLAUDE.md se mantiene casi estático (prompt caching). Formato COMPACTO a propósito: detalle fino on-demand en `docs/design/`.

**Última actualización**: 2026-09-25 noche (oráculo Nivel 1 implementado — BUG-013 resuelto, suite completa 7.6' / 133 fwd)

## HEAD · tests · working tree

- **HEAD**: cierre 24/09 — 7 commits locales **PUSHEADOS a origin/main** (`a4b054d` → bump docs). Submodule `call_me_maybe_docs` **pusheado** (3790ef6 → nota pendientes). Remotos VERIFICADOS 25/09: origin/main == local == `f876719`.
- **Working tree (25/09 noche) — SIN commitear, listo para commit con OK**:
  - `src/decoder/constrained_generator.py`: 2º tramo lineal (BUG-013) reemplazado por **oráculo por estado Nivel 1** (`_next_static_text` + `_TRAMPS` T1–T6, funciones puras, `emitted` por iteración, `_inject_static_header` con acumulador opcional).
  - `tests/test_constrained_generator.py`: clase `TestLevel1Oracle` (canónicos por tramo, dominios disjuntos, BUG-013, insertion order, align E) + `TestInjectOracleText` (simulate() como fuente de verdad) + `TestOracleEndToEnd` (P2-style 9 fwd, fn_empty con T6).
- **Suite: 195 tests GREEN — VERIFICADO (25/09 noche)** · **flake8 0** · **mypy Success** ✅.
- **Opt2** (`d6592d0`): header estático inyectado con `encode()` sin forward + hook de métricas. **BUG-011** (`794a470`): fix real en `state.py._step_string` (rechaza `\` en `current_key=="name" and depth==0`); guard viejo en `schema_validator.py` REMOVIDO (quedaba muerto).
- **BUG-012** (con Opt2, en `d6592d0`): `STATIC_HEADER` debe ser byte-exacto al formato natural del modelo — incluidas las 2 newlines iniciales antes de `{` (ver CONTEXTO_REFACTOR §2.2). Cortarlas rompía "Greet shrek". Restauradas → completan bien.
- **BUG-013 — ✅ RESUELTO (25/09 noche)**: oráculo por estado con gate **N** (`depth==0 ∧ current_key=="name" ∧ keys_enclosed==∅`) — ver BITACORA + ANEXO D8 (hallazgo: el flag `has_seen_params_object` es sticky y NO puede gatear T6 con tokens fusionados).
- **Stash**: `stash@{0}` refactor-metrics descartado · `stash@{1}` anexo-reverted. NO tocar.
- **Scratch FUERA del repo** (correr SIEMPRE con `cwd = repo`): `~/scratch/call_me_maybe_task42/` (task43_accuracy.py, bench_p8_vs_p2.py, vocab_study.py). Backup muerto `~/scratch/PENDING_DELETE__call_me_maybe_backup_mario_validation/` (borrar cuando se descarte).
- `data/output/` (git-ignored): `metrics_run.json` + `metrics_run.log` — destino de métricas.

## Mediciones — suite Task 4.3 (Qwen3-0.6B, threads=4, warm-up descartado) · 24/09 vs ref 23/09

| Métrica | 23/09 ref | 24/09 | Δ |
|---|---|---|---|
| generación | 1656.4 s (27.6') | **908.6 s (15.1')** | **-45%** |
| total c/carga | — | 16.8' | — |
| avg/prompt | 150.6 s | 82.6 s | -45% |
| forwards | ~638 (58/p) | **314 (28.5/p)** | **-51%** |
| KPI <5' | 5.5x | **3x INCUMPLIDO** | — |

- **accuracy fn: 11/11 (100%)** · **accuracy full: 9/11 (82%) — bar M14 (≥90%) NO pasa**.
- Per prompt (s): 92.2 · 98.4 · 64.0 · 59.2 · 59.2 · 49.7 · 58.2 · 61.5 · 130.3 · 113.2 · 122.7.
- Forwards por fase (314): IN_STRING_VALUE **109 (35%)** · PARAMS_OBJECT 38 · COLON 36 · VALUE_END 33 · IN_KEY 33 · KEY_START 30 · IN_OBJECT 22 · IN_NUMBER_VALUE 13. **skips_if_single = 0 (3ra vez → M5 código muerto)**.
- JSON: `data/output/metrics_run.json` (warm_up_discarded=true, 908550 ms, 314 fwd, desglose por fase).
- ⚠ La suite completa (908.6s / 15.1') corrió con la versión ANTERIOR del header (sin fix BUG-012). La remedición aislada con BUG-012 resuelto está abajo — los tiempos individuales bajaron más.

## Remedición aislada P2/P8 post-Opt2 (sesión-tarde 24/09, con BUG-011/012 ya resueltos)

- **P2** (Greet shrek): 113.2s → **48.5s (-57%)** · **P8** (regex, peor caso): 242.5s → **125.9s (-48%)**. Aislado (2/11 prompts) — NO reconciliado todavía con el run completo de la suite.
- `skips_if_single = 0` en TODAS las fases, 3ra medición → **M5 (skip-if-single) = código muerto confirmado**.

## Benchmark 2º tramo + probe fn_empty (madrugada 25/09, working tree sucio BUG-013)

- **P8 con 2º tramo activo**: 56→49 forwards (−7 exactos), output válido ✅. **P2 CORRUPTO** → **BUG-013** (detalle en BITACORA): con el trigger viciado, el timing de esa corrida NO es limpio (forwards extra por output degenerado).
- **Probe paso 0 fn_empty (modelo real, Qwen3-0.6B, threads=4)**: función sin parámetros → **`"parameters": {}` — cierre INLINE** (sin newline ni indent). **Observación**: el probe terminó `SUCCESS=False` (faltó el `}` final del ROOT) — conectado a la causa raíz de BUG-013 (estado residual `current_key` al bajar de depth); diagnóstico en la implementación del oráculo.

## Benchmark oráculo Nivel 1 (noche 25/09, modelo real, threads=4, warm-up descartado)

| Caso | Ref 23/09 | Opt2 (d6592d0) | **Oráculo N1** | Δ vs ref |
|---|---|---|---|---|
| P2 fn_greet | 113.2 s | 48.5 s (-57%) | **16.8 s** | **-85%** |
| P8 regex | 242.5 s | 125.9 s (-48%) | **80.4 s** | **-67%** |

- **Forwards**: P2 = **7** (piso teórico: solo los chars del value `"shrek"`); P8 = 29 (los 3 values strings). Las fases de estructura (IN_OBJECT/PARAMS_OBJECT/VALUE_END) quedaron en **0 forwards** — toda la sintaxis sale por el oráculo.
- **Probe fn_empty post-fix**: **SUCCESS=True** (25.1 s) — el `}` final del ROOT lo inyecta T6; output byte-exacto `'\n\n{\n  "name": "fn_empty",\n  "parameters": {}\n'`.
- **Suite completa con oráculo (25/09 noche)**: **7.6 min de generación** (15.1' con Opt2 → **-50%**) · accuracy fn 11/11 (100%) · full 9/11 (82%, misma bar M14 — P9/P10 regex semántico) · **133 forwards** totales. Fases: IN_STRING_VALUE 114 fwd (372.3s — 82% del tiempo), IN_NUMBER_VALUE 13 (58.3s), COLON 6 (23.9s — tokens fusionados), IN_OBJECT/PARAMS_OBJECT/VALUE_END = **0 fwd** (estructura 100% oráculo). **KPI <5' INCUMPLIDO**.
- **Conclusión KPI (25/09 noche)**: con 3.42 s/fwd reales, B′ (~31 fwd) deja ~97 fwd → ~5.5'; B′+Nivel 2 → piso real **~5.6'**. El piso físico son los ~96 values libres del modelo (string+number) — irreductibles sin semántica/KV-cache (prohibidos). **KPI <5' inalcanzable en esta CPU** → decisión estratégica pendiente de usuario (hardware / aceptar KPI por prompt / redefinir).

## Estudio del vocabulario (24/09) — `~/scratch/call_me_maybe_task42/vocab_study.{py,_results.json}`

- `string_safe` = 96.87% de 151,643 → bucketizar IN_STRING_VALUE NO sirve (wildcard ≈ gratis).
- BPE fragmenta dígitos/puntuación: peor 2.17 chars/token ("Hello 34 I'm 233 years old" = 12 tokens); global 3.33; palabras 5–6.
- 395 tokens con backslash (incl. `\n` tid 1699, `\t`, `\\`, `\"`) · 1,344 con comilla (0.9%) · bucket " " = 53k (35%).
- Forwards de strings ≈ tokens BPE + comilla final, **EXACTO** (probe de segmentación 25/09: P8 7+13+3+2 = 25 == 25 medidos) → **NO hay ~40% titubeo (nota vieja REFUTADA)**. El margen está en la estructura (oráculo + B′), no en valores de strings libres.

## ⚠ Pendientes de la cancha (24/09)

1. **Bar M14 al 82%: los 2 fallos son SEMÁNTICOS, no del decoder.** JSON 100% válido en los 11. P9: `replacement='NUMBER'` vs "...with NUMBERS". P10: regex `([aeiouAEIOU])` (con grupo) vs expected sin paréntesis. → decidir: relajar expected o exigir.
2. **BUG-011 — RESUELTO y commiteado** (`794a470`): un token BPE que ES el escape completo (ej. `\n` fusionado, tid 1699) resuelve ESCAPE_IN_STRING→IN_STRING_VALUE DENTRO de `simulate()` → el `new_state` que ve el schema nunca queda en ESCAPE_IN_STRING → un guard ahí no lo atrapa (repro real: "Greet shrek" → `'{\n  "name": "f'`, success=False, 200 forwards). Fix real en `state.py`: rechaza `\` AL LEERLO si `current_key=="name" and depth==0`. Verificado: 170 green.

## Pistas verificadas para seguir optimizando (contexto próxima sesión)

- **El forward es compute-bound, no filter-bound**: costo casi uniforme (~4.4–5.9s) sin importar el candidate set (ROOT chico ≈ IN_STRING_VALUE con casi todo el vocab) — confirmado por profiling por fase + estudio de vocabulario. Acotar candidatos (trie/Opt1 en `IN_KEY`) NO abarata cada forward; solo evita forwards SI logra singleton (no evaluado a nivel token BPE).
- **Forwards de `IN_STRING_VALUE` = tokens BPE + comilla final, EXACTO** (probe 25/09) — NO hay margen en strings libres; prompts con dígitos/puntuación (P8) fragmentan peor. Irreductible sin SDK.
- **Opt2 es la palanca real para forwards NO-string** — pero byte-exacto (BUG-012); variante "optimizada" = riesgo de regresión silenciosa.
- **Fase 2 (oráculo por estado) es la palanca VIGENTE**: generaliza el 2º tramo a tabla de tramos por estado (E/K), fix BUG-013 de raíz. Espera contrato de interfaces del diseño consultado (`_next_static_text`, `emitted`, spec de tramos). Registro completo en ANEXO (25/09).

## Pendientes — PARA RETOMAR (orden del usuario, 24/09 noche + 25/09)

1. ✅ **Suite completa con oráculo Nivel 1 — CORRIDA (25/09 noche)**: 7.6' (vs 15.1' Opt2, -50%), 133 fwd, fn 100% / full 82%, KPI <5' incumplido (piso físico ~5.6' — ver sección benchmark).
2. ✅ **BUG-013 — RESUELTO (25/09 noche)**: oráculo por estado con gate N; hallazgo P-sticky (tokens fusionados) documentado en BITACORA + ANEXO D8.
3. ✅ **Oráculo por estado (Modelo A) — IMPLEMENTADO** (T1–T6, Nivel 1; Nivel 2 / tokens fusionados DIFERIDO hasta medir el residuo).
4. ✅ **Probe fn_empty — SUCCESS=True** (25.1s): el `}` del ROOT que faltaba lo inyecta T6.
5. ⏳ **B′ (autocompletar fn_name por trie)** — DIFERIDO (decisión usuario): recién después de medir el Modelo A completo.
6. ⏳ **Decisiones de estrategia (requieren usuario)**: KPI <5' (con oráculo se acerca) — aceptar por prompt / hardware-modelo / re-diseñar; bar M14 (82% vs ≥90%) — relajar expected (P9 `NUMBER`/`NUMBERS`, P10 regex con grupo) o exigir.
7. ⏳ Tasks 5.1–5.3 + DoD5 + 6.1–6.5 (`src/validator/` no existe; pipeline NO persiste output — a4b054d solo desacopló prints).

## Protocolo de actualización

- INICIO de sesión: verificar el real (`git status -sb`, `git log --oneline -3`); actualizar si difiere. CIERRE: reflejar HEAD, tests, mediciones. Estado desactualizado = peor que ninguno.
