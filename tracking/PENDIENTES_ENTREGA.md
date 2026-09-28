# PENDIENTES DE PREPARACIÓN DE ENTREGA — subject VIII

> **Fecha de corte:** 28/09/2026 · **Alcance:** TODO lo que falta para entregar.
> **Fuente:** §5 de `ESTADO_ACTUAL.md` (pendientes vigentes) + auditoría del 28/09
> contra `moulinette/__main__.py`, los generadores de la cátedra y el corretero real.
> **Regla:** este doc es la lista de trabajo. Si algo está acá, NO está entregado.

---

## 🔴 BLOQUEANTES — cada uno vale 0 puntos por sí solo

Orden de ataque sugerido: son cortos y de riesgo decreciente.

### B1. `README.md` — INEXISTENTE (obligatorio, subject VI pág. 16-17)
No hay README en el repo, local ni versionado. La planilla de peer review tiene
**3 ítems** que dependen de él, y el subject lo pide con secciones obligatorias
y **en inglés**:

- Primera línea, **en itálica**:
  `*This project has been created as part of the 42 curriculum by <login1>, ...*`
  → ⏳ **Falta confirmar los logins de 42.**
- Description · Instructions · Resources
- Resources debe incluir **cómo se usó la IA**: para qué tareas y en qué partes del proyecto.
- Algorithm explanation · Design decisions · Performance analysis
- Challenges faced · Testing strategy · Example usage

**Todo el material existe** (docs de diseño, ESTADO_ACTUAL, CONTEXTO_REFACTOR).
Es trabajo mecánico de redacción. Es lo que más puntúa.

### B2. "All classes must use pydantic" — 6 de 10 clases no lo son
El subject IV.3.1 lo dice textual; la planilla lo tiene como condición.
- Sin pydantic: `SchemaContext`, `DecoderState`, `TrieNode`, `Vocab`,
  `PhaseMetrics`, `MetricsRun` (+ `DecoderPhase`, que es `Enum`).
- Con pydantic: `ParameterDef`, `FunctionDef`, `FunctionCall`.

⚠️ **No convertir por reflejo:** `Vocab` (151.643 tokens) y `DecoderState`
(mutado por carácter del output) están en el hot path. Meter validación
pydantic ahí puede costar el KPI de <5'. **Medir antes de decidir.**
Propuesta: convertir las baratas (`TrieNode`, `PhaseMetrics`, `MetricsRun`) y
para las del hot path decidir con medición + documentar la excepción si aplica.
(Ex-Material pendiente §5.4 → **escalado a bloqueante** por la auditoría.)

### B3. `.opencode/` versionado en el repo — RESUELTO 28/09
`git rm -r --cached .opencode` + `.gitignore`. Queda en disco, fuera del track.
Era config de agente de IA dentro de la entrega: ruido para un tercero y
palanca de la bandera de trampa.

### B4. `data/correction/` sin ignorar — RESUELTO 28/09
La cátedra corre `prepare_exercises` **en el repo del alumno** y escribe ahí
las respuestas de los 11 tests. Sin ignorar, un `git add -A` post-defensa
versiona las soluciones. Agregado a `.gitignore`.

### B5. `uv.lock` no versionado — RESUELTO 28/09
El subject dice textual: *"the reviewer, as well as the moulinette, will just
run `uv sync`"*. Sin lock resuelve lo último publicado (hoy `transformers 5.15.0`).
Lock versionado; `uv sync --frozen` verificado y sin rutas absolutas (portable).

---

## 🟡 MEDIOS — no bloquean, pero son puntos de la planilla

### M1. Falta el warning de "prompt sin match" — **RESUELTO 28/09** (ver abajo)
`find_unsupported_prompts` en `src/validator/output_validator.py`, cableado en
`src/pipeline.py`, descarga a stderr. Si NINGÚN valor de parámetro aparece
literalmente en el prompt, avisa que el request probablemente no matchea ninguna
función. No elige función (eso es del LLM) ni toca el JSON. 19 tests nuevos
en `tests/test_output_validator.py`, que además cubren `build_results` —que
estaba **sin un solo test** pese a garantizar la alineación posicional del `zip()`.

### M2. Tasks 5.1–5.3 y 6.x del plan maestro
- 5.1 validador de output que contraste la llamada contra el **schema** de la
  función elegida (hoy se valida contra las definiciones, no contra el schema).
- 5.2/5.3 — **resuelto de facto** por `a4b054d` + `79499ab`: el output ya
  persiste con el formato del subject. Falta cerrar la tarea en el plan.
- 6.x pipeline non-interactive + métricas + reports: parcial.

### M3. Higiene de prosa — limpiar español de código y docstrings
El usuario lo pidió explícitamente. Los comentarios están en español por
decisión propia (y el subject no lo prohíbe), pero **strings visibles al
usuario final y docstrings públicos** deberían ir en inglés.

### M4. Bonus B7 (Prima) — no reclamado
Gastar effort solo si sobrara tiempo. La lista real de bonuses está abajo.

---

## 🟡 PENDIENTES DE MEDICIÓN (no bloquean la entrega)

### P1. Correr el smoke del set privado — **ARMADO, FALTA CORRER**
```
bash ~/scratch/call_me_maybe_task42/private/run_private_smoke.sh both
```
⚠️ Con `opencode` CERRADO. El set privado reconstruido es **byte-idéntico** al
que generan los generadores de la cátedra (verificado por diff), así que esto
es el input real del evaluador. Riesgo conocido sin medir:
- `fn_is_even(n: integer)` — el schema privado usa `"integer"`, no `"number"`
- `fn_calculate_compound_interest(number, number, integer)` — dos floats + un int
- backslashes: `C:\Users\john\config.ini`
- comillas dobles dentro de template strings
Ambos fixes ya están commiteados (`aed4c14`) pero **sin probar contra el set real**.

### P2. Investigar el 2x CPU-s/forward
Nunca explicado. Dato nuevo del 28/09: corriendo **con el agente vivo**, 3
prompts dieron **7,83 s/forward** vs **2,29 s/forward** con el agente cerrado
(3,4x). Es contaminación de CPU, no un problema del decoder. **No tocar el
código por esto.** Documentar y seguir.

### P3. Latencia en máquina del campus
Pendiente de medición propia del usuario. Con 4 vCPU (sin cap) a ~2,1 s/fwd la
suite da ~4,8 min. El cap 80% deja **3,2 de 4 cores** (medido por burn test):
para medir local, usar cap 100% y no confundir.

### P4. Test de humo para un equipo con GPU
Imposible acá: `torch 2.13.0+cpu`, `cuda.is_available()=False`, la VM no
pasa GPU. El decoder ya es device-agnostic (`llm_sdk` resuelve device y dtype;
`src/` es CPU puro). Procedimiento documentado: guardar el golden de CPU,
correr en GPU, diffear. Si da idéntico → cerrado para siempre.

### P5. Memoria estable
Pendiente: samplear RSS durante la corrida completa. Hoy el peak medido es
**4,65 GiB** (índice de vocabulario en RAM), estable en corridas cortas.

### P6. Backups de los probes de /tmp — PARCIAL
`grade_real.py` y el set privado se recuperaron en `~/scratch/`. Los demás
probes de /tmp se perdieron. lowest value; sólo recuperarlos si hacen falta.

---

## ✅ RESUELTOS EN ESTA SESIÓN (28/09) — para no repetirlos

| Ítem | Cómo |
|---|---|
| `data/correction/` ignorado | `.gitignore` + prueba: git no lo ve |
| `.opencode/` fuera del índice | `git rm -r --cached` |
| `uv.lock` versionado | `uv sync --frozen` OK, portable |
| Warning de prompt sin match | `find_unsupported_prompts` + 19 tests |
| Coerción `number` → float | `aed4c14` (6 tests públicos passes) |
| `"integer"` del schema privado | `aed4c14` |
| Conteo de forwards por prompt | `79499ab` → `decode_metrics.json` |
| Output persistido | `a4b054d` |
| Line endings CRLF→LF | `ef89a5c` |
| BUG-011 (loop de escapes en `name`) | `794a470` |
| BUG-012 (empate técnico en coma) | `794a470` |

---

## ❌ DECIDIDO NO HACER (con el porqué — no reabrir sin dato nuevo)

| Descartado | Por qué |
|---|---|
| **Prompt engineering** para P9 | El LLM no tiene la capacidad: no reconoce el `****` como reemplazo válido de una vocal (4 templates, siempre 0 ejemplos). P9 es **límite del modelo**, no bug. El KPI de accuracy (≥90%) ya se cumple con 10/11. Tocar el prompt arriesga los otros 10 tests + latencia. |
| **Orquestación de `src/decoder/`** (índice de args, trie global) | Diseñada y descartada: el pre-índice no reduce forwards. El coste es **compute-bound** (sin KV-cache), uniforme por paso. |
| **Optimización B′** (batching) | Descartada: agrega complejidad sobre hardware donde la suite ya cumple. |
| **Nivel 2 del oráculo** | Descartado por el usuario. |
| **Migrar la `moulinette` al repo** | Ya está fuera del track (`.gitignore` + `.flake8` + `pyproject`). Es código de la cátedra, no nuestro. |
| **Hardcodear P9** | Trampa. Además no cierra el set privado (backslashes). |
| **`numpy`** en el código | El subject lo autoriza explícitamente. Higiene menor: o se usa o se saca de las deps. |

---

## 🎁 BONUS PRIMA — qué se puede reclamar (0-5)

| Bonus | Estado |
|---|---|
| Conjunto de pruebas completo | ✅ **240 tests**, lint limpio |
| Optimizaciones de rendimiento | ✅ pre-índice, header estático, oráculo por estado, trie, skip-if-single (−45%) |
| Visualización de la generación | ⚠️ hay métricas; falta el step-by-step con color |
| Mecanismos de recuperación de errores | ⚠️ pase fino post-argmax, parcial |
| Demo encode/decode + constrained decoding | ⚠️ **se hace en el README, gratis** ← mejora el B1 |
| Recodificación del tokenizador | ❌ usamos `encode`/`decode` del SDK, que es lo que el subject MANDA. El bonus es para reimplementarlo. No se reclama. |
| Anidados complejos (object/array) | ❌ fuera del scope declarado |

---

## 🔍 CONTRADICCIÓN DOCUMENTADA (para el README → B1)

La planilla dice: *"los parámetros de tipo 'número' aceptan enteros **o** floats"*.
Pero el código de la cátedra hace `fn(**params)` y las funciones assertan
`isinstance(a, float)`. **El texto es más permisivo que el código.**

Nuestro fix (emitir `2.0`) satisface las dos cosas: es float válido para un
"number" **y** pasa el assert. Si nos hubiéramos guiado por el texto, habríamos
perdido 6 tests. **Vale documentarlo en el README** para que un revisor entienda
por qué hacemos la coerción en vez de dejarla.

---

## 📋 CHECKLIST DE LA NOCHE (para cuando el usuario mida)

1. `opencode` cerrado, VM en 4 vCPU, **cap 100%** para medir limpio.
2. `bash ~/scratch/call_me_maybe_task42/private/run_private_smoke.sh both`
3. Mirar: score privado 11/11, escapes de backslash, `decode_metrics.json`.
4. En otra terminal, samplear RSS: `watch -n5 'ps -o rss= -C python | sort -n | tail -1'`
5. Anotar en `ESTADO_ACTUAL.md` y actualizar este doc.

---

## 📌 RESTRICCIONES QUE NO SE ROMPEN

- NO librerías fuera del subject. NO commits sin OK. NO tocar los stashes.
- `docs/` es submodule privado, **no va en la submission**: commit aparte + bump.
- `data/output/` es git-ignored: las métricas JSON viven ahí, no se versionan.
- Nunca commitear `data/correction/` (contiene las respuestas de la cátedra).
