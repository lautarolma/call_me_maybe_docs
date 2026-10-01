# PLAN_EJECUCION_V2 — call_me_maybe

> **Fecha**: 2026-09-26
> **Reemplaza a**: `PLAN_IMPLEMENTACION.md` como documento de *ejecución*. La Parte A
> de aquel documento (arquitectura, A1–A13) **sigue vigente como referencia de diseño**
> y NO se duplica acá — se cita por sección.
> **Motivo de la v2**: el plan original estaba ordenado por *módulo* ("hacelo todo de
> los loaders, después todo el decoder"). El código real ya no sigue esa forma: el decoder
> está completo e incluye optimizaciones que el plan original no contemplaba (Opt2,
> oráculo N1, pre-índice de vocabulario). Lo que falta ya no es "construir módulos" sino
> **cerrar la capa de salida**. Este plan reordena el trabajo alrededor de eso.

---

## 0. Mapa de reutilización (qué del original se conserva)

| Sección original | Veredicto | Dónde vive ahora |
|---|---|---|
| A1 Contexto y estado | 🟡 **Parcial** | Desactualizado: dice que `src/` falta. Estado real en `ESTADO_ACTUAL.md` |
| A2 Requisitos (M1–M16, S1–S5, B1–B9) | 🟢 **Vigente** | Se mantiene íntegra. Es el contrato del subject |
| A3 Restricciones críticas | 🟢 **Vigente** | Se mantiene íntegra |
| A4 Arquitectura objetivo | 🟢 **Vigente** | El árbol de `src/` es idéntico al planificado |
| A5 Flujo de datos | 🟡 **Parcial** | El flujo cambió: header inyectado + oráculo (ver §2.1) |
| A6 State machine del decoder | 🟢 **Vigente** | `src/decoder/state.py` — 15 fases, 16 `DecoderPhase` |
| A6.4 algoritmo de allowed-tokens | 🟢 **Vigente** | `token_filter.py` — la lógica de 3 fases se respetó |
| A7 Trie | 🟢 **Vigente** | `src/decoder/trie.py` (83 líneas) |
| A8 Manejo de errores | 🟡 **Parcial** | Tabla de errores existe; cobertura real incompleta (ver E3) |
| A9 Diseño de performance | 🔴 **Superado** | El plan quedó atrás; ahora hay optimizaciones que no contemplaba (ver §2.2) |
| A10 Estrategia de testing | 🟢 **Vigente** | 195 tests existen |
| A11 Riesgos | 🟢 **Vigente** | El riesgo #1 se materializó (ver §1) |
| A13 Ambigüedades resueltas | 🟡 **1 decisión es ahora RIESGO** | La fila del nombre del archivo hay que revisitar (ver E1) |
| Parte C: Phase 1–4 (tasks) | ✅ **Hechas** | Se conservan como registro histórico |
| Parte C: Phase 5–6 (tasks) | ⬜ **Pendientes** | Se reordenan en Etapas 2–5 de este plan |
| Parte B: Anexo BONUS | ⬜ **No iniciado** | Va al Anexo de este plan |

**Lo que se descarta del original**: la premisa de que el trabajo pendiente es
"construir el decoder". Eso ya ocurrió. El plan v1, si se ejecutara hoy, haría trabajo
duplicado.

---

## 1. Por qué el plan v1 ya no sirve (y qué falló)

El plan v1 tiene Phase 4 (Generation Loop) como última fase ejecutada. Su
**Definition of Done** pedía 4 cosas. El estado real es **1 de 4 en verde**:

| Criterio DoD Phase 4 | Estado | Evidencia |
|---|---|---|
| 1 prompt → JSON válido con fn correcta | ✅ | `results_43.json` ok_fn 11/11 |
| 11 prompts → todos producen output | ⚠️ **en memoria, nunca a disco** | `src/pipeline.py:104` — *"Task 5.2 persistirá esta lista"* |
| Timing <5' | ❌ 7.6' local | 1.24' en hardware de cluster (pendiente de confirmar forwards) |
| Accuracy ≥90% | ❌ 82% (9/11) | P9 y P10 |
| `make lint` pasa | ✅ | |

**La causa raíz de que 4.3 no cierre no es el decoder — es que Phase 5 nunca se
empezó.** El decoder funciona; lo que no existe es el *output*. Por eso la Etapa 1
de este plan es bloqueante y va primera.

---

## 2. Realidad del código que el plan v1 no refleja

Esta sección es la que justifica la reordenación. Tres optimizaciones y una deuda
que el plan original no contemplaba.

### 2.1 El decoder ya no es un bucle lineal de tokens

El plan v1 (A5, Task 4.1) describe: *encode → loop { get_logits → filter → argmax → decode → update }*.
Eso ya no es lo que pasa. El flujo real tiene **dos shortcuts** encima:

| Shortcut | Qué hace | Ganancia |
|---|---|---|
| **Opt2 — header estático** (`STATIC_HEADER`, `constrained_generator.py:82`) | El prefijo `'\n\n{\n  "name": "'` se inyecta con `encode()` **sin ningún forward** | 45% |
| **Oráculo por estado N1** (`_TRAMPS` T1–T6, `:305-332`) | Cuando la state machine está en una fase estructural, `_next_static_text()` devuelve el texto completo del tramo sin consultar al modelo | 50% (Opt2 → 7.6') |

Consecuencia: las fases estructurales (`IN_OBJECT`, `PARAMS_OBJECT`, `VALUE_END`) quedaron en
**0 forwards**. Todo el costo vive en `IN_STRING_VALUE` (82% del tiempo, 114/133 fwd).
**Cualquier optimización futura debe apuntar ahí** — el resto ya es gratis.

### 2.2 Vocabulario pre-indexado

`src/loader/vocab_loader.py` construye `Vocab.tokens_starting_with`: un índice de
primer carácter decodificado → set de ids. Convierte un escaneo de ~151k tokens en
O(1). El plan v1 (A6.2) solo비는 "bucketizar no sirve" para `IN_STRING_VALUE` — y sigue
sirviendo para las fases de keys, donde es el mecanismo real.

### 2.3 Deuda técnica registrada

| Deuda | Ubicación | Impacto |
|---|---|---|
| `M5 skip-if-single` = código muerto (3ª medición, `skips_if_single=0`) | `token_filter.py` | Ruido en métricas; confunde la lectura del desglose por fase |
| `DecoderPhase.VALUE_START` declarado y nunca usado | `state.py:76` | Dead code en la máquina de estados |
| Docstrings sin estandarizar | todo `src/` | Inciso 6.5.1 diseñado, nunca aplicado |

### 2.4 Bugs resueltos que el plan v1 no preveía

`BUG-011` (`state.py._step_string` rechaza `\` en `current_key=="name" and depth==0`),
`BUG-012` (el header estático debe ser byte-exacto, **incluidas 2 newlines iniciales**),
`BUG-013` (el flag `has_seen_params_object` es *sticky* y no puede gatear T6 con tokens
fusionados). Los tres están resueltos y verificados. Sirven de aviso: **el oráculo
N1 es correcto pero frágil ante cambios en segmentación BPE** — cualquier cambio en el
formato del header exige re-correr el probe de `fn_empty`.

---

## 3. El camino de ejecución

Ordenado por **bloqueos reales**, no por módulo. Cada etapa declara su entrada, sus
steps, y su criterio de salida verificable.

---

### ETAPA 0 — Verificar hardware de cluster
**Estado**: 🟡 partial · **Bloquea**: nada · **Depende de**: nada

- [ ] **0.1** Confirmar forwards del cluster: `uv run python -c "import json; d=json.load(open('data/output/metrics_run.json')); print(d.get('total_forwards'), d.get('total_ms'))"`
      → si `133`, queda confirmado **0.56 s/fwd** (5.2x vs local) y el KPI de 5' es
      alcanzable sin GPU. Si NO es 133, hay una diferencia de comportamiento entre
      máquinas que hay que debuggear antes de nada más.
- [ ] **0.2** Documentar en `ESTADO_ACTUAL.md` con el resultado.
- [ ] **0.3** Decidir si la evaluación final se corre en cluster o en la máquina local.
      Esto define qué KPI es el que importa.

**DoD**: forwards confirmados y registrados.

---

### ETAPA 1 — Capa de salida ⛔ **BLOQUEANTE**
**Estado**: ⬜ no empezada · **Bloquea**: TODO lo demás · **Depende de**: nada

> **Por qué primera**: hoy el programa **imprime** resultados y los descarta. El subject
> V.4 exige un archivo. Sin esto el score es 0, sin importar el accuracy o el timing.

- [ ] **1.1 — Decidir el nombre del archivo** (3 líneas de investigación, altísimo impacto)
      El subject es internamente inconsistente:
      - línea 350 (ejemplo CLI) → `data/output/function_calls.json`
      - línea 582 (V.4 Output File Format) → `data/output/function_calling_results.json`
      - línea 662 (V.6 Testing) → `Check that output/function_calling_results.json is created`

      El plan v1 (A13) eligió `function_calls.json` contra 2 de 3 menciones.
      **Decisión recomendada: escribir AMBOS** (el default del CLI queda como está; se
      escribe además el alias). Es la única opción que sobreviva a las dos lecturas del
      enunciado y cuesta 3 líneas. Updatear el default de `--output` a
      `function_calling_results.json` es la alternativa si se prefiere una sola.
- [ ] **1.2 — `src/validator/output_validator.py`** (Task 5.1 del plan v1)
      `validate_output(raw_output: str, functions: list[FunctionDef]) -> FunctionCall | str`.
      Usa `FunctionCall` (ya existe en `src/models/output.py`) + `FunctionDef`.
      Reusa el schema validator del decoder donde se pueda — no reimplementar validación.
- [ ] **1.3 — `src/validator/__init__.py`** con export público.
- [ ] **1.4 — Tests del validator**: output válido, output con key extra, tipo incorrecto,
      key faltante, JSON malformado, garbage. Sin modelo.
- [ ] **1.5 — Persistir en `src/pipeline.py`** (Task 5.2)
      Acumular `FunctionCall` (o dicts) en vez de `list[str]`, y escribir con
      `Path.write_text(json.dumps(..., indent=2))` al final. Crear el directorio con
      `mkdir(parents=True, exist_ok=True)`. Escribir **ambos nombres** si se adopta 1.1.
- [ ] **1.6 — Propagar errores sin tumbar el pipeline** (cruza con 6.1 del plan v1)
      Un prompt que falla validación → warning + continuar, no `raise`. El subject
      (línea 321) exige que el programa "must never crash unexpectedly".
- [ ] **1.7 — Verificación de formato** (Task 5.3): test que lee el archivo generado y
      valida 11 entries con exactamente las keys `name` + `parameters`.

**DoD**:
- [ ] `uv run python -m src` genera el archivo con 11 entries
- [ ] El archivo abre como JSON válido, 100% parseable
- [ ] Cada entry tiene exactamente `name` (str) y `parameters` (object)
- [ ] Un prompt inválido produce warning, no crash
- [ ] `make lint` + `make test` en verde

---

### ETAPA 2 — Accuracy M14 (≥90%)
**Estado**: ⟡ investigándose · **Bloquea**: DoD final · **Depende de**: Etapa 1

- [ ] **2.1** Ejecutar el forense sobre P9/P10 (prompt quirúrgico ya generado).
      Nota previa: **P10 falla por DOS razones**, no una. Además del regex con grupo
      capturador, hay `replacement='****'` vs `'*'` — que **no** es relaxable.
- [ ] **2.2** Clasificar cada error: `ARTEFACTO_DE_SCORER` vs `ERROR_REAL`.
- [ ] **2.3** Aplicar el fix. Orden de preferencia:
      1. Si es artefacto del scorer → documentar y no tocar código.
      2. Si es ambigüedad del prompt template (`src/prompt/prompt_builder.py`) → ahí sí
         es un fix nuestro, legítimo, y no es hardcoding.
      3. Prohibido: heurísticas o post-proceso que "arregle" la salida (subject línea 310).
- [ ] **2.4** Re-medir accuracy con `task43_accuracy.py`. Objetivo: ≥10/11 (90.9%).

**DoD**: accuracy ≥90% medida, con la causa raíz de P9 y P10 documentada.

> **Dato de contexto**: P9 y P10 son también los 2 prompts más lentos (36.7% del tiempo
> entre ambos). La causa común es la misma: valores string con caracteres atípicos
> (dígitos, apóstrofos) que fragmentan en muchos tokens BPE. Si el fix de accuracy
> tocara la segmentación, movería también el timing.

---

### ETAPA 3 — Robustez y error handling
**Estado**: ⟡ parcial · **Depende de**: Etapa 1

- [ ] **3.1** Auditar cobertura de la tabla de errores de A8 contra la realidad:
      `FileNotFoundError` con mensaje claro, `JSONDecodeError`, duplicados en functions,
      lista de prompts vacía, schema sin match, `allowed set vacío`.
- [ ] **3.2** Exit codes: `0` éxito · `!=0` falla crítica. Verificar que un path
      irrecuperable (falta `functions_definition.json`) sale con `!=0` y mensaje útil.
- [ ] **3.3** Logging consistente: warnings a stderr, info a stdout (S3).
- [ ] **3.4** Tests de los paths de error (S2, S4).

**DoD**: cada fila de la tabla A8 tiene un test que la ejercita, o una justificación de por qué no aplica.

---

### ETAPA 4 — Documentación
**Estado**: ⬜ no empezada · **Depende de**: Etapas 1–3

- [ ] **4.1** `README.md` (Task 6.2): descripción · quick start (`make install && make run`)
      · arquitectura · formato de output · requisitos (incl. la nota de que el build de
      torch es CPU-only y por qué).
- [ ] **4.2** Estandarizar docstrings según el **Inciso 6.5.1** (Task 6.5 del plan v1,
      ya diseñado en la línea 644). Empezar por los públicos: `pipeline.py`,
      `constrained_generator.py`, `state.py`, `token_filter.py`.

**DoD**: un dev nuevo corre el proyecto leyendo solo el README.

---

### ETAPA 5 — Quality gate
**Estado**: 🟡 parcial · **Depende de**: todo lo anterior

- [ ] **5.1** Coverage real (`>70%` según DoD del plan v1). Medir antes de asumir.
- [ ] **5.2** Limpiar deuda: `M5 skip-if-single` muerto y `DecoderPhase.VALUE_START`
      sin usar (ver §2.3). Decidir: eliminar o documentar por qué se quedan.
- [ ] **5.3** `make lint` y `make lint-strict` limpios.
- [ ] **5.4** `make test` en verde y sin warnings.
- [ ] **5.5** Verificación final contra A2: recorrer M1–M16 punto por punto y marcar
      cuáles están cubiertos con evidencia.

**DoD**: `make lint` + `make test` verdes, coverage >70%, A2 verificado punto por punto.

---

## Anexo — Bonus (no bloquea el MVP)

Se conserva de la Parte B del plan v1. **No empezar antes de cerrar la Etapa 5.**

| # | Bonus | Nota |
|---|---|---|
| B1 | `lint-strict` target | Ya existe en el Makefile — solo verificar que corre |
| B5 | Optimizaciones de performance | Solo si el KPI sigue sin cumplirse en hardware modesto |
| B6 | Tests comprehensivos | Parcialmente hecho (195 tests) |
| B7 | Visualización de generación | Alto valor didáctico, bajo valor de score |
| B9 | API pública `encode`/`decode` | |

---

## 4. Mapa de dependencias

```
ETAPA 0 (cluster)  ──────────────────────────────┐  (informativa, no bloquea)
                                                  │
ETAPA 1 (salida) ⛔ ──► ETAPA 2 (accuracy) ───────┤
       │                     │                    │
       └──► ETAPA 3 ─────────┴──► ETAPA 4 ──► ETAPA 5
```

**Camino crítico**: Etapa 1. Es la única que bloquea a todas las demás y la única que
hoy vale 0 puntos → 100.

---

## 5. Reglas para reformular este plan en el futuro

1. **El plan se reordena cuando cambia el bloqueante**, no cuando cambia el módulo.
2. Cada etapa declara su *bloquea* y su *depende de*. Si no puede, no es una etapa.
3. Toda tarea que quede [ ] debe decir **cómo se verifica**, no **qué se hace**.
4. Los desvíos del código se documentan en §2, no se borra del plan — el original
   ya enseñó que un plan que no registra sus desvíos se vuelve engañoso.
