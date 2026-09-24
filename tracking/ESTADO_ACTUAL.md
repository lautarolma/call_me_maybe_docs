# ESTADO_ACTUAL — call_me_maybe (42 School)

> Archivo de estado DINÁMICO — importado con `@` desde `CLAUDE.md`. Se actualiza al inicio/fin de cada sesión. TODO lo que cambia entre sesiones va acá; CLAUDE.md se mantiene casi estático (prompt caching). Formato COMPACTO a propósito: detalle fino on-demand en `docs/design/`.

**Última actualización**: 2026-09-24, cierre noche (commits finales + verificación 170 green / lint limpio)

## HEAD · tests · working tree

- **HEAD**: `794a470` fix BUG-011 → `d6592d0` Opt2+hook → `a348931` metrics → `cf1f2d8` docs → `a4b054d` pipeline. **LOCAL, ahead 6 de origin/main (SIN push)**. Submodule docs: `80bf441` (estado+contexto+anexo+plan).
- **Suite: 170 tests GREEN — VERIFICADO (24/09 noche)** · **flake8 0** · **mypy Success (21 archivos)** ✅.
- **Working tree LIMPIO** (todo commiteado; quedan solo locales git-ignored: CLAUDE.md, data/output/).
- **Opt2** (`d6592d0`): header estático inyectado con `encode()` sin forward + hook de métricas. **BUG-011** (`794a470`): fix real en `state.py._step_string` (rechaza `\` en `current_key=="name" and depth==0`); guard viejo en `schema_validator.py` REMOVIDO (quedaba muerto); tests: `test_escape_rejected_in_name_value` (reemplaza al viejo) + clase `TestFineValidationNameEscapeRejected` (+2, parcialmente redundante pero aporta cobertura de fine-validation).
- **BUG-012** (con Opt2, en `d6592d0`): `STATIC_HEADER` debe ser byte-exacto al formato natural del modelo — incluidas las 2 newlines iniciales antes de `{` (mismo warning de la política compacta abortada `7bf38ed`, ver CONTEXTO_REFACTOR §2.2). Cortarlas rompía "Greet shrek" (tokenizaba name como "f" suelto, cero candidatos). Restauradas → ambos casos completan.
- **Stash**: `stash@{0}` refactor-metrics descartado · `stash@{1}` anexo-reverted. NO tocar.
- **Scratch FUERA del repo** (correr SIEMPRE con `cwd = repo`): `~/scratch/call_me_maybe_task42/` (task43_accuracy.py, bench_p8_vs_p2.py, vocab_study.py). Backup muerto `~/scratch/PENDING_DELETE__call_me_maybe_backup_mario_validation/` (borrar cuando se descarte).
- `data/output/` (git-ignored): `metrics_run.json` + `metrics_run.log`.

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

## Estudio del vocabulario (24/09) — `~/scratch/call_me_maybe_task42/vocab_study.{py,_results.json}`

- `string_safe` = 96.87% de 151,643 → bucketizar IN_STRING_VALUE NO sirve (wildcard ≈ gratis).
- BPE fragmenta dígitos/puntuación: peor 2.17 chars/token ("Hello 34 I'm 233 years old" = 12 tokens); global 3.33; palabras 5–6.
- 395 tokens con backslash (incl. `\n` tid 1699, `\t`, `\\`, `\"`) · 1,344 con comilla (0.9%) · bucket " " = 53k (35%).
- Forwards de strings ≈ tokens BPE + ~40% titubeo → irreductible con SDK actual (roll K=4 inviable sin logits multi-step).

## ⚠ Pendientes de la cancha (24/09)

1. **Bar M14 al 82%: los 2 fallos son SEMÁNTICOS, no del decoder.** JSON 100% válido en los 11. P9: `replacement='NUMBER'` vs "...with NUMBERS". P10: regex `([aeiouAEIOU])` (con grupo) vs expected sin paréntesis. → decidir: relajar expected o exigir.
2. **BUG-011 — RESUELTO y commiteado** (`794a470`): un token BPE que ES el escape completo (ej. `\n` fusionado, tid 1699) resuelve ESCAPE_IN_STRING→IN_STRING_VALUE DENTRO de `simulate()` → el `new_state` que ve el schema nunca queda en ESCAPE_IN_STRING → un guard ahí no lo atrapa (repro real: "Greet shrek" → `'{\n  "name": "f'`, success=False, 200 forwards). Fix real en `state.py`: rechaza `\` AL LEERLO si `current_key=="name" and depth==0`. Verificado: 170 green.

## Pistas verificadas para seguir optimizando (contexto próxima sesión)

- **El forward es compute-bound, no filter-bound**: costo casi uniforme (~4.4–5.9s) sin importar el candidate set (ROOT chico ≈ IN_STRING_VALUE con casi todo el vocab) — confirmado por profiling por fase + estudio de vocabulario. Acotar candidatos (trie/Opt1 en `IN_KEY`) NO abarata cada forward; solo evita forwards SI logra singleton (no evaluado a nivel token BPE).
- **Forwards de `IN_STRING_VALUE` ≈ tokens BPE + titubeo** — prompts con dígitos/puntuación (P8) fragmentan peor. Irreductible sin SDK (sin batch/KV-cache/speculative).
- **Opt2 es la palanca real para forwards NO-string** — pero byte-exacto (BUG-012); variante "optimizada" = riesgo de regresión silenciosa.
- **Próxima idea barata**: extender Opt2 con un 2º tramo estático (entre el cierre del value de `"name"` y la apertura de `"parameters": {`) — mismo mecanismo, NO evaluado.

## Pendientes (orden)

1. ⏳ **Decisión de estrategia con datos nuevos** + seguir buscando vías de reducir tiempos (pedido explícito del usuario): 2º tramo Opt2 (pista arriba) · hardware/modelo · aceptar KPI por prompt · re-diseñar bar M14 → requiere usuario.
2. ⏳ **Reconciliar la suite completa con BUG-012 resuelto** (re-correr suite 11 prompts con header byte-exacto → tiempos deberían bajar más que 15.1').
3. ⏳ Scoring M14: relajar expected (regex sin grupo / NUMBERS) vs exigir → usuario.
4. ⏳ Tasks 5.1–5.3 + DoD5 + 6.1–6.5 (`src/validator/` no existe; pipeline NO persiste output — a4b054d solo desacopló prints).
5. ⏳ **Push pendiente**: 6 commits locales (`a4b054d`..`794a470`) + submodule docs `80bf441`.

## Protocolo de actualización

- INICIO de sesión: verificar el real (`git status -sb`, `git log --oneline -3`); actualizar si difiere. CIERRE: reflejar HEAD, tests, mediciones. Estado desactualizado = peor que ninguno.
