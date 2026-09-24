# ESTADO_ACTUAL — call_me_maybe (42 School)

> Archivo de estado DINÁMICO — importado con `@` desde `CLAUDE.md`. Se actualiza al inicio/fin de cada sesión. TODO lo que cambia entre sesiones va acá; CLAUDE.md se mantiene casi estático (prompt caching). Formato COMPACTO a propósito: detalle fino on-demand en `docs/design/`.

**Última actualización**: 2026-09-24, sesión tarde (fix definitivo BUG-011 en `state.py` + BUG-012 header Opt2 + remedición P2/P8)

## HEAD · tests · working tree

- **HEAD**: `a4b054d` refactor(pipeline) — **LOCAL, ahead 1 de origin/main (SIN push)**. Sobre `f2d58e1` (lint) → `d444ed0` (pre-índice) → `a1436b9` (164 green) → `e72fd25` (Mario).
- **Suite: 167 tests GREEN** (última corrida verificada, ANTES de los cambios de esta sesión-tarde) · **flake8 0** · **mypy Success** ✅.
- ⚠ **Cambios de esta sesión-tarde SIN re-correr pytest/flake8/mypy todavía** (pendiente próxima sesión) — no asumir verde:
  - `src/decoder/state.py` — fix DEFINITIVO de BUG-011 en `_step_string` (rechaza `\` cuando `current_key=="name" and depth==0`).
  - `src/decoder/schema_validator.py` — guard de BUG-011 anterior (ESCAPE_IN_STRING) removido (quedaba muerto) + docstrings corregidos apuntando al fix real en `state.py`.
  - `tests/test_state.py` — `test_escapes_skipped_in_buffer` (documentaba el comportamiento buggy) reemplazado por `test_escape_rejected_in_name_value`.
  - `tests/test_constrained_generator.py` (clase `TestFineValidationNameEscapeRejected`, del fix anterior en schema_validator.py) — queda parcialmente redundante con el fix nuevo en `state.py`; revisar si sigue aportando cobertura distinta.
  - `src/decoder/constrained_generator.py` — **Opt2**: header estático (`STATIC_HEADER`) inyectado con `model.encode()` sin pasar por forward/filtro + hook de métricas (`MetricsRun`).
  - `src/utils/metrics.py` (+109) — `PhaseMetrics` + `MetricsRun` (report/write_json, `warm_up_discarded`).
  - `tests/test_metrics.py` (nuevo, 3 tests).
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
- **Opt2 (working tree) = responsable del -45%**: cortó forwards a la mitad (~638 → 314).

## ⚠ Pendientes que dejaron la cancha marcada (24/09)

1. **Bar M14 al 82%: los 2 fallos son SEMÁNTICOS, no del decoder.** JSON 100% válido en los 11 prompts. P9: `replacement='NUMBER'` vs "...with NUMBERS". P10: regex `([aeiouAEIOU])` (con grupo) vs expected sin paréntesis. Brecha = contenido del modelo → decidir: relajar expected o exigir.
2. **BUG-011 — RESUELTO (sesión-tarde 24/09)**: el guard en `schema_validator.py` (ESCAPE_IN_STRING) era insuficiente — verificado por rastreo de código: un token BPE que ES el escape completo (ej. `\n` fusionado, tid 1699) resuelve ESCAPE_IN_STRING→IN_STRING_VALUE **dentro de `simulate()`**, así que el `new_state` final que ve el schema nunca queda en ESCAPE_IN_STRING → el guard no lo atrapa → `compute_allowed_ids` devolvía ese token como único candidato, el pase fino lo rechazaba, y sin más candidatos la generación se cortaba sin completar (repro: "Greet shrek" → `'{\n  "name": "f'`, `success=False`). **Fix real**: 3 líneas en `state.py._step_string` — rechaza el `\` AL LEERLO si `current_key=="name" and depth==0`, sin importar cómo el BPE fusione el escape. Guard viejo en `schema_validator.py` removido (quedaba muerto). **Pendiente**: re-correr suite para confirmar 169 green con este cambio.
3. **BUG-012 (nuevo, 24/09) — RESUELTO**: el header estático de Opt2 (`STATIC_HEADER` en `constrained_generator.py`) tiene que ser BYTE-EXACTO al formato natural que el modelo produce solo — incluidas las 2 newlines iniciales antes de `{` (mismo warning ya documentado en `CONTEXTO_REFACTOR.md` §2.2 sobre la política compacta abortada `7bf38ed`). Cortar esas newlines (por creer que ahorraba forwards — NO ahorraba nada, la inyección evita el forward sin importar el largo del texto) rompía "Greet shrek": el modelo quedaba en un estado fuera de distribución y tokenizaba el name como un "f" suelto en vez de un chunk natural, sin candidatos válidos en el step siguiente. Con las newlines restauradas, ambos casos de prueba completan bien.

## Estudio del vocabulario (24/09) — `~/scratch/call_me_maybe_task42/vocab_study.{py,_results.json}`

- `string_safe` = 96.87% de 151,643 → **bucketizar IN_STRING_VALUE NO sirve** (wildcard ≈ gratis).
- BPE fragmenta dígitos/puntuación: peor 2.17 chars/token ("Hello 34 I'm 233 years old" = 12 tokens); global 3.33; palabras 5–6.
- 395 tokens con backslash (incl. `\n` tid 1699, `\t`, `\\`, `\"`) · 1,344 con comilla (0.9%) · bucket " " = 53k (35%).
- Forwards de strings ≈ tokens BPE + ~40% titubeo → irreductible con SDK actual (roll K=4 inviable sin logits multi-step).

## Remedición aislada P2/P8 post-Opt2 (sesión-tarde 24/09, con BUG-011/012 ya resueltos)

- **P2** (Greet shrek): 113.2s → **48.5s (-57%)**. **P8** (regex, peor caso): 242.5s → **125.9s (-48%)**. Aislado (2/11 prompts) — NO reconciliado todavía con el run completo de la suite (908.6s/15.1' de la sección de arriba, que corresponde a una versión anterior del header sin el fix de BUG-012).
- `skips_if_single = 0` en TODAS las fases, 3ra medición consecutiva → **M5 (skip-if-single) es código muerto confirmado**, no solo en wildcards.

## Pistas verificadas para seguir optimizando (contexto para la próxima sesión)

- **El forward es compute-bound, no filter-bound**: costo por forward casi uniforme (~4.4-5.9s) sin importar el tamaño del candidate set (ROOT chico ≈ IN_STRING_VALUE con casi todo el vocab) — confirmado por 2 métodos independientes (profiling propio por fase + estudio de vocabulario). Implica: acotar candidatos (trie/Opt1 en `IN_KEY`) NO abarataría cada forward, solo evitaría ALGUNOS forwards SI logra singleton — no evaluado si el trie puede forzar singleton a nivel de token BPE (no de char).
- **Los forwards de `IN_STRING_VALUE` son ≈ tokens BPE + titubeo de reintento** — los prompts con dígitos/puntuación (P8) fragmentan peor en BPE y por eso concentran más forwards. Irreductible sin tocar el SDK (sin batch/KV-cache/speculative no hay cómo adivinar sin gastar otro forward).
- **Opt2 (header estático) es la palanca real para los forwards NO-string** — pero exige ser byte-exacto al formato natural del modelo (BUG-012); cualquier variante "optimizada" del header es riesgo de regresión silenciosa.
- Extender Opt2 con un SEGUNDO tramo estático (entre el cierre del value de `"name"` y la apertura de `"parameters": {`) es la próxima idea barata a probar — mismo mecanismo, no evaluado todavía.

## Pendientes (orden)

1. ⏳ Re-correr pytest + flake8 + mypy tras los cambios de esta sesión-tarde (`state.py`, `schema_validator.py`, `tests/test_state.py`) — NO verificado todavía.
2. ⏳ **Decisión de estrategia con datos nuevos**: Opt2 ya rindió -45% en la suite completa (15.1', versión anterior del header) y -48/-57% en la remedición aislada P2/P8 (con BUG-012 ya resuelto); KPI sigue incumplido. Cuello = cantidad de forwards, concentrados en `IN_STRING_VALUE` (ver pistas arriba). Opciones: 2do tramo de Opt2, hardware/modelo, aceptar KPI por prompt, re-diseñar bar M14. → **más tarde hoy: seguir buscando otras vías de reducir tiempos** (pedido explícito del usuario).
3. ✅ Fix BUG-011 a prueba de balas (`state.py`, 3 líneas) — **aplicado esta sesión-tarde**, pendiente re-verificar suite.
4. ⏳ Scoring M14: relajar expected (regex sin grupo / NUMBERS) vs exigir → usuario.
5. ⏳ Tasks 5.1–5.3 + DoD5 + 6.1–6.5 (`src/validator/` no existe; pipeline NO persiste output — a4b054d solo desacopló prints).
6. ⏳ **Push pendiente**: `a4b054d` + working tree completo (Opt2, BUG-011/012, metrics, tests).

## Protocolo de actualización

- INICIO de sesión: verificar el real (`git status -sb`, `git log --oneline -3`); actualizar si difiere. CIERRE: reflejar HEAD, tests, mediciones. Estado desactualizado = peor que ninguno.
