# PENDIENTES DE PREPARACIÓN DE ENTREGA — subject VIII

> **Fecha de corte:** 06/10/2026 · **Alcance:** TODO lo que falta para entregar.
> **Fuente:** re-baseline completo contra el estado real (smoke 05/10, 272 tests).
> **Regla:** este doc es la lista de trabajo. Si algo está acá, NO está entregado.
> Estrategia y decisiones cerradas → `PRE_ENTREGA.md`.

---

## 🔴 BLOQUEANTES — cada uno vale 0 puntos por sí solo

### B1. `README.md` — ✅ EXISTE (commits `779484a` + `84e8412`)
Requisitos del subject + bucle de decode explicado paso a paso, en inglés,
primera línea en itálica con login. **Verificar en la auditoría** que cubre
todas las secciones obligatorias (Description · Instructions · Resources ·
cómo se usó la IA · Algorithm explanation · Design decisions · Performance
analysis · Challenges faced · Testing strategy · Example usage).

### B2. "All classes must use pydantic" — ✅ DECISIÓN DE DISEÑO, NO PENDIENTE
**Cerrado 06/10.** No es un bloqueante a resolver: es arquitectura ya decidida
y documentada (Decisión 1 de `PLAN_IMPLEMENTACION.md` §A12, `PRE_ENTREGA.md` §2).

- **Criterio**: pydantic en la **frontera de I/O** — `loader/` valida la
  entrada (`ParameterDef`, `FunctionDef`), `validator/` valida la salida
  (`FunctionCall`). **Todo lo que entra y sale del sistema pasa por pydantic.**
- **Por qué NO se integra paso a paso en el decoder**: `Vocab` (151.643
  tokens), `DecoderState` (mutado por carácter) y `SchemaContext` viven en el
  hot path. `@dataclass(slots=True)` cuesta ~30ns; pydantic ~200μs por
  instancia. Meterlo ahí arriesga el KPI <5' sin aportar validación — esos
  objetos no son I/O, son estado interno del algoritmo.
- **Qué sí es pydantic hoy**: `ParameterDef`, `FunctionDef`, `FunctionCall`
  — el 100% de lo que se lee de disco y se escribe a `function_calling_results.json`.
- **Qué NO y por qué** (documentar en README → sección Design decisions):
  `DecoderState`, `Vocab`, `TrieNode`, `PhaseMetrics`, `MetricsRun`,
  `SchemaContext` — inner loop / estado de máquina, no frontera.

### B3. Auditoría de frontera + recorrido de docstrings — EN CURSO (06/10)
Ver `PRE_ENTREGA.md` §3-4. Consiste en: (1) inventario de qué pertenece a
la entrega, (2) review superficial por módulo, (3) reescritura de docstrings
a norma ENGLISH Google-style con síntesis didáctica. Es el grueso del
trabajo de hoy.

---

## 🟡 MEDIOS — no bloquean, pero son puntos de la planilla

### M1. Task 5.1 — validación de `parameters` contra el SCHEMA — ✅ CERRADO 09/10 (cubierto por construcción)
**Decisión:** NO se revalida post-hoc. El schema se enforcea en decode-time por
construcción (`SchemaContext`: keys no-extra/no-duplicadas, required, tipos,
forma integer) — subject V.1/V.3; y `FunctionCall` (pydantic) valida la
estructura del output (IV.3.1). `validate_output()` era redundante y **no lo
invocaba nadie** (código muerto): eliminado junto con sus 2 tests. Ver
`ANALISIS_ALINEACION` §2.

### M2. Tasks 5.2/5.3 y 6.x del plan maestro
- 5.2/5.3: resuelto de facto (`a4b054d` + `79499ab`) — falta cerrar la fila.
- 6.x pipeline non-interactive + métricas + reports: parcial
  (`report_prompt_metrics` + `decode_metrics.json` existen; falta unificar
  el doble camino `report()`/`write_json()`).

### M3. Bonus B7 — visualización de generación (DIFERIDA 06/10)
Aplazada hasta avanzar más con la entrega. **Foundation ya existe**:
`MetricsRun` + `DecoderPhase` + `report_prompt_metrics()` + tablas de score
de los smokes. **Gap**: capa step-by-step (token a token, verde/rojo/
amarillo) gateada con `isatty()` + `NO_COLOR`. Diseño: `PLAN_DIDACTICO.md`
M13 B7. Nace en inglés.

### M4. Bonus B7 de la prima (prima, no reclamada) — solo si sobra tiempo
Ver tabla de bonuses abajo.

---

## 🟡 PENDIENTES DE MEDICIÓN (no bloquean la entrega)

### P1. Smoke del set privado — ✅ CORRIDO 05/10 · 11/11 + 11/11
`run_private_smoke.sh both` con corretero real. Evidencia:
`data/output/private_smoke/` (public 11/11 + private 11/11, 272 tests,
flake8 0, mypy 23 archivos). Las reglas A/B/C se ven reparando en vivo en
`suite_private.log`. **Cerrado.**

### P2. Latencia local — fuera de KPI por hardware, ✅ resuelto en CAMPUS
Medido 05/10: pública ~325 s (KPI 300 s), privada ~407 s. Regresión atribuida
a entorno (energía/host), no a código — mismo trabajo, mejor CPU-s/fwd,
menos cores efectivos. **El KPI se jugó en campus y PASÓ (ver P3).**

### P3. Latencia en campus — ✅ RESUELTO 09/10
Caja real de corrección (i5-8500, 6 threads, 7,63 GiB, `torch 2.13.0+cpu`,
sin CUDA, cache HF frío): **wall 163 s (2'43")** · generación 124 s (2'04") ·
**137 forwards** → **KPI PASS** (límite 300 s, margen 137 s). Pase GPU
`SKIPPED` (sin CUDA; entregable CPU-only). Referencia histórica local:
292,72 s (27/09, 4 vCPU cap 80, agente cerrado). Cluster: 74,3 s.

### P4. Test de humo en equipo con GPU — pendiente
Guardar golden de CPU, correr en GPU, diffear. Procedimiento documentado
en `docs/notes/HARDWARE_VM.md`. Cierra para siempre la variable float16.

### P5. Memoria estable (RSS) — pendiente
Peak medido: 4,65 GiB (vocab index). Samplear durante corrida completa si
sobra tiempo: `watch -n5 'ps -o rss= -C python | sort -n | tail -1'`.

### P6. Backups de probes — parcial
`grade_real.py` + set privado recuperados en `~/scratch/`. El resto de
probes de `/tmp` se perdió. Lowest value.

---

## ✅ RESUELTOS — no repetir

| Ítem | Cómo / Cuándo |
|---|---|
| Smoke público + privado E2E | 05/10 → 11/11 + 11/11 |
| **Latencia KPI en campus** (163 s, PASS) | 09/10 → caja i5-8500, CPU-only |
| Reglas A/B/C (truncado, comillas, conteo) | `bd07ad1` + `3087fc5` |
| P9 reparado sin tocar logits | regla C, `_collapse_repeated_run` |
| Coerción `number`→float / `"integer"` privado | `aed4c14` |
| Eco de stdout post-validación | `echo_view()` (pendiente de commit 06/10) |
| `README.md` | `779484a` + `84e8412` |
| `data/correction/` ignorado | `.gitignore` 28/09 |
| `.opencode/` fuera del índice | 28/09 |
| `uv.lock` versionado | 28/09 |
| Warning prompt sin match | `find_unsupported_prompts` + 19 tests |
| `docs/` fuera del repo principal | 06/10 (submodule → local standalone) |
| `CLAUDE.md` eliminado | 06/10 (contenido migrado a `docs/`) |
| Conteo de forwards por prompt | `79499ab` → `decode_metrics.json` |
| BUG-011 / BUG-012 / BUG-014 / BUG-015 | ver `BITACORA_BUGS.md` |

---

## ❌ DECIDIDO NO HACER (con el porqué — no reabrir sin dato nuevo)

| Descartado | Por qué |
|---|---|
| **Pydantic en el inner loop del decoder** | Decisión de diseño (ver B2): ~200μs/instancia en hot path arriesga KPI <5'. Los objetos del decoder no son I/O. |
| **Prompt engineering para P9** | El LLM no lo resuelve (13,77 logits abajo). La regla C lo repara post-hoc → 11/11. Tocar prompt arriesga los otros 10 tests. |
| **Orquestación de `src/decoder/`** | El pre-índice no reduce forwards; coste compute-bound uniforme. |
| **Optimización B′ (batching)** | Complejidad sobre hardware donde la suite ya cumple. |
| **Nivel 2 del oráculo** | Descartado por el usuario. |
| **Migrar la `moulinette` al repo** | Ya fuera del track. Es código de la cátedra. |
| **Hardcodear P9** | Trampa. Además no cierra el set privado. |
| **`numpy` en el código** | El subject lo autoriza; higiene menor: se usa o se saca de deps. |
| **Traducir a inglés por `sed` todo el backlog de comentarios** | Superseded: la reescritura a norma ENGLISH Google-style es parte del recorrido módulo a módulo (hoy), no un task aparte. |

---

## 🎁 BONUS PRIMA — qué se puede reclamar (0-5)

| Bonus | Estado |
|---|---|
| Conjunto de pruebas completo | ✅ **272 tests**, lint limpio |
| Optimizaciones de rendimiento | ✅ pre-índice, header estático, oráculo, trie, skip-if-single (−45%) |
| Visualización de la generación (B7) | ⚠️ **diferida** — foundation de métricas existe; falta step-by-step con color (M3) |
| Mecanismos de recuperación de errores | ⚠️ pase fino post-argmax + reglas A/B/C post-hoc, parcial |
| Demo encode/decode + constrained decoding | ⚠️ se hace en el README, gratis ← mejora B1 |
| Recodificación del tokenizer | ❌ no se reclama (el subject manda usar el SDK) |
| Anidados complejos | ❌ fuera de scope declarado |

---

## 🔍 CONTRADICCIÓN DOCUMENTADA (para el README → Design decisions)

La planilla dice: *"los parámetros de tipo 'número' aceptan enteros **o** floats"*.
Pero el código de la cátedra hace `fn(**params)` y las funciones assertan
`isinstance(a, float)`. **El texto es más permisivo que el código.**

Nuestro fix (emitir `2.0`) satisface las dos cosas. Documentarlo en el README
para que un revisor entienda la coerción.

---

## 📌 RESTRICCIONES QUE NO SE ROMPEN

- NO librerías fuera del subject. NO commits sin OK. NO tocar los stashes.
- `docs/` es local y **no va en la submission** (gitignored desde 06/10).
- `data/output/` es git-ignored: métricas y entregable viven ahí.
- Nunca commitear `data/correction/` (contiene las respuestas de la cátedra).
- Nunca medir latencia con `opencode` vivo (contamina 2,6x).
