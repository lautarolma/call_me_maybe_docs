# ANEXO: Optimización de Latencia — Reducción de 34.9 min a ≤5 min

**Proyecto:** call_me_maybe
**Fecha de creación:** 18 septiembre 2026
**Trigger:** Checkpoint post-Task 4.3 — accuracy 91% VERDE, timing 34.9 min ROJO (KPI <5 min)
**Estado:** PLAN DE REESTRUCTURACIÓN — pendiente de implementación
**Autor del análisis:** Auditoría conjunta (profiling E2E + comparativa Mario_Call_me_maybe + benchmarks externos)

---

## 1. RESUMEN EJECUTIVO

### 1.1 El problema

La Task 4.3 (11 prompts, modelo real Qwen3-0.6B, CPU) midió **34.9 minutos de generación** contra un KPI de **≤5 minutos**. La suite de tests (164 green) y la accuracy (91% ≥ 90%) están en verde — el problema es **exclusivamente de rendimiento**, no funcional.

### 1.2 Diagnóstico de causa raíz (dos bottlenecks, no uno)

El profiling separado de cada componente del pipeline reveló que **el bottleneck principal NO es el modelo**:

| Componente por step | Costo medido | % del tiempo |
|---|---|---|
| **Filter: key/string abierta (wildcard `*`)** | **7.15s – 8.67s** | **~60–70%** |
| **Forward del modelo (190 tokens, sin KV-cache)** | **2.6s** | **~25–30%** |
| Filter: ROOT (`{` únicamente) | 1.6s | ~5% |
| Otros (decode, overhead) | <0.5s | ~1% |

**Hallazgo estrella:** El `token_filter.py` en Python puro (Fase 2+3: `simulate()` + `allows_token()` sobre ~151K candidatos) es **2.7–3.3x más caro que el forward del modelo** en C++/oneDNN.

### 1.3 Meta de optimización

| Métrica | Actual | Objetivo |
|---|---|---|
| Latencia por prompt | ~190s | ~10–20s |
| Latencia 11 prompts | 34.9 min | **<5 min** |
| Calls al modelo por prompt | ~70 (1 por step) | **4–8** (solo steps ambiguos) |
| Costo filter por step (wildcard) | 7–8.7s | **<50ms** |
| Accuracy (M14) | 91% (10/11) | **≥90%** (sin regresión) |
| JSON válido | 100% (11/11) | **100%** (sin regresión) |

---

## 2. PLAN DE REESTRUCTURACIÓN — PASO A PASO

### 2.1 Arquitectura actual del pipeline de generación

```
generate() [constrained_generator.py]
│
├─ for _ in range(max_tokens):                    ← loop de ~70 steps/prompt
│   │
│   ├─ logits = model.get_logits(input_ids)       ← FORWARD: 2.6s (llama al modelo)
│   │                                               sin KV-cache: secuencia completa
│   │
│   ├─ allowed = compute_allowed_ids(             ← FILTER: 7–8.7s en wildcard
│   │       state, schema, vocab, trie, logits)     151K tokens simulados en Python
│   │
│   ├─ best = argmax(allowed, logits)             ← argmax sobre el set permitido
│   │
│   ├─ _pick_best_token(allowed, logits, ...)     ← pase fino post-argmax (1ms)
│   │
│   ├─ input_ids.append(best_id)                  ← agregar token a la secuencia
│   ├─ state.update_from_text(token_text)          ← avanzar máquina de estados
│   └─ schema.update(state)                        ← sincronizar schema
│
└─ model.decode(generated_ids)                     ← decode final
```

**Costo por step: 2.6s (forward) + 7.15–8.67s (filter) = ~10s–11.2s**
**Costo por prompt (70 steps): ~700–784s ≈ 11–13 min** (con variación por tipo de step)
**Costo 11 prompts: ~34.9 min medido**

### 2.2 Arquitectura propuesta (dos optimizaciones)

```
generate() [constrained_generator.py] — MODIFICADO
│
├─ for _ in range(max_tokens):                    ← loop optimizado
│   │
│   ├─ allowed = compute_allowed_ids(             ← FILTER: ~5ms (Top-K/Top-1)
│   │       state, schema, vocab, trie,
│   │       logits=None)                           ← SIN logits: fast-path determinista
│   │
│   ├─ if len(allowed) == 1:                      ← SKIP: token determinista
│   │   ├─ best_id = only_element(allowed)
│   │   ├─ token_text = vocab.id2decoded[best_id]
│   │   └─ continue (SIN llamar al modelo)        ← AHORRA 2.6s
│   │
│   ├─ logits = model.get_logits(input_ids)       ← FORWARD: SOLO si hay ambigüedad
│   │                                               (4–8 steps por prompt, no 70)
│   │
│   ├─ allowed = compute_allowed_ids(             ← FILTER: Top-K masking ~5ms
│   │       state, schema, vocab, trie, logits)
│   │
│   ├─ best = argmax(allowed, logits)             ← argmax sobre Top-K permitido
│   ├─ _pick_best_token(allowed, logits, ...)
│   ├─ input_ids.append(best_id)
│   ├─ state.update_from_text(token_text)
│   └─ schema.update(state)
│
└─ model.decode(generated_ids)
```

**Costo por step determinista: ~5ms (filter) + 0s (sin modelo) = ~5ms**
**Costo por step ambiguo: 2.6s (forward) + ~5ms (filter Top-K) = ~2.6s**
**Costo por prompt (~5 steps ambiguos × 2.6s + ~65 steps deterministas × 0.005s): ~13.3s**
**Costo 11 prompts: ~2.5 min** ✅ ≤5 min

---

### 2.3 PASO 1: Opportunistic / Top-K Logit Masking

#### 2.3.1 Archivo a modificar: `src/decoder/token_filter.py`

**Qué es:** `compute_allowed_ids()` calcula el set de IDs de tokens permitidos para el próximo step de generación, validando ~151K candidatos contra la máquina de estados y el schema.

**Dónde:** Función `compute_allowed_ids()` (líneas 62–121).

**Cómo funciona actualmente (el problema):**

```python
def compute_allowed_ids(state, schema, vocab, trie, logits):
    expected_chars = state.expected_first_chars()

    # Fase 1: pre-filtro por primer carácter — O(1) por bucket
    if "*" in expected_chars:
        candidate_ids = set()                    # ← WILDCARD: junta TODOS los buckets
        for first_char, ids in vocab.tokens_starting_with.items():
            if first_char != BYTE_CATEGORY:
                candidate_ids.update(ids)        # ← ~151K tokens en wildcard

    # Fase 2+3: simular Y validar CADA candidato — O(candidatos × chars)
    allowed_ids = set()
    for token_id in candidate_ids:               # ← loop de ~151K iteraciones
        decoded = vocab.id2decoded.get(token_id)
        if decoded is None or not _is_clean_utf8(decoded):
            continue
        valid, new_state = state.simulate(decoded)  # ← Python puro: copiar + chars
        if not valid:
            continue
        if schema.allows_token(decoded, new_state, trie):  # ← trie + schema check
            allowed_ids.add(token_id)            # ← ~149K pasan en wildcard

    return allowed_ids                           # ← ~149K tokens permitidos
```

**El problema:** En wildcard (`*`), el candidate set tiene ~151K tokens. Cada uno pasa por `simulate()` (copia de state + loop de chars) y `allows_token()` (trie + 4 cláusulas ANDed). Costo total: **7.15–8.67s** por step.

**Cómo funciona propuestamente (la solución):**

```python
def compute_allowed_ids(state, schema, vocab, trie, logits=None, top_k=2000):
    """Computa el set de ids permitidos para el próximo step.

    OPTIMIZACIÓN (Inciso B5 + Anexo de Latencia):
    - Si logits son provistos, se usa Top-K masking: solo se validan
      los K tokens de mayor logit, no los 151K del vocabulario.
    - Si logits es None (skip-if-single), se usa el filtro completo
      como fallback (necesario para verificar si hay 1 candidato).
    """
    expected_chars = state.expected_first_chars()

    # ─── FAST-PATH: Top-1 Opportunistic (O(1)) ───
    if logits is not None:
        best_id = max(range(len(logits)), key=lambda i: logits[i])
        best_decoded = vocab.id2decoded.get(best_id)
        if best_decoded is not None and _is_clean_utf8(best_decoded):
            valid, new_state = state.simulate(best_decoded)
            if valid and schema.allows_token(best_decoded, new_state, trie):
                return {best_id}                 # ← 1 candidato, O(1), <0.1ms

    # ─── FAST-PATH: Top-K Masking (O(K)) ───
    if logits is not None:
        # Extraer los top-K IDs de mayor logit
        indexed = sorted(range(len(logits)),
                         key=lambda i: logits[i], reverse=True)[:top_k]
        # Intersección con candidate_ids de Fase 1
        ...
    else:
        # Fallback: filtro completo (usado por skip-if-single)
        ...

    # Fase 2+3: validar SOLO los K candidatos
    for token_id in filtered_candidates:          # ← ~2000 iteraciones, no 151K
        decoded = vocab.id2decoded.get(token_id)
        if decoded is None or not _is_clean_utf8(decoded):
            continue
        valid, new_state = state.simulate(decoded)
        if not valid:
            continue
        if schema.allows_token(decoded, new_state, trie):
            allowed_ids.add(token_id)

    return allowed_ids
```

**Cambios específicos:**

| Cambio | Ubicación | Descripción |
|---|---|---|
| **A. Firma de función** | Línea 62 | Agregar `logits: list[float] | None = None` y `top_k: int = 2000` como parámetros opcionales. Retrocompatible (default None = sin cambio). |
| **B. Fast-path Top-1** | Nuevo bloque antes de Fase 1 | Si `logits` no es None: calcular argmax global, simular solo ese token, si pasa → retornar `{best_id}` (O(1)). |
| **C. Fast-path Top-K** | Nuevo bloque entre Top-1 y Fase 1 | Si Top-1 falla: extraer top-K con `sorted(..., reverse=True)[:top_k]`, intersecar con Fase 1, validar solo esos K. |
| **D. Fallback completo** | Fase 2+3 existente | Si `logits` es None (skip-if-single): ejecutar el loop completo sobre todos los candidatos (necesario para determinar si hay exactamente 1). |
| **E. Empty-set handling** | Línea 103 | Mantener: si `allowed` queda vacío tras Top-K, fallback a filtro completo (garantiza correctness). |

#### 2.3.2 Archivos afectados por el Paso 1

| Archivo | Impacto | Cambios |
|---|---|---|
| `src/decoder/token_filter.py` | **ALTO** (se modifica) | Nueva lógica Top-1/Top-K + firma opcional `logits`. |
| `src/decoder/constrained_generator.py` | **MEDIO** (consumidor) | Llamadas a `compute_allowed_ids` ahora sin logits en skip-if-single, con logits en step ambiguo. |
| `tests/test_token_filter.py` | **MEDIO** (tests) | Nuevos tests: Top-1 aceptado, Top-1 rechazado → fallback Top-K, Top-K vacío → fallback completo, equivalencia de output. |
| `src/decoder/schema_validator.py` | **CERO** | Sin cambios (consumido por el filter). |
| `src/decoder/state.py` | **CERO** | Sin cambios (consumido por el filter). |
| `src/decoder/trie.py` | **CERO** | Sin cambios. |
| `src/pipeline.py` | **CERO** | Sin cambios. |
| `src/loader/vocab_loader.py` | **CERO** | Sin cambios. |

---

### 2.4 PASO 2: Skip-if-Single / Inversión de Control en el Loop

#### 2.4.1 Archivo a modificar: `src/decoder/constrained_generator.py`

**Qué es:** `generate()` es el loop principal de generación: por cada step, obtiene logits del modelo, filtra tokens permitidos, elige el mejor, y avanza el estado.

**Dónde:** Función `generate()` (líneas 61–136) y helper `_pick_best_token()` (líneas 139–164).

**Cómo funciona actualmente (el problema):**

```python
def generate(model, prompt, vocab, functions, trie, max_tokens=200):
    input_ids = model.encode(prompt)[0].tolist()
    prompt_length = len(input_ids)
    state = DecoderState()
    schema = SchemaContext(functions)

    for _ in range(max_tokens):
        logits = model.get_logits_from_input_ids(input_ids)  # ← SIEMPRE: 2.6s
        allowed = compute_allowed_ids(state, schema, vocab, trie, logits)  # ← 7–8.7s

        if not allowed:
            break

        best_id, token_text = _pick_best_token(
            allowed, logits, state, schema, functions, vocab, trie)

        if best_id is None:
            break

        input_ids.append(best_id)
        state.update_from_text(token_text)
        schema.update(state)

        if state.phase is DecoderPhase.COMPLETE:
            break
    ...
```

**El problema:** `model.get_logits_from_input_ids(input_ids)` se llama **en CADA step** (incondicionalmente), aun cuando el 80–90% de los steps son deterministas (solo 1 token legal posible). Costo: 70 × 2.6s = 182s por prompt.

**Cómo funciona propuestamente (la solución):**

```python
def generate(model, prompt, vocab, functions, trie, max_tokens=200):
    input_ids = model.encode(prompt)[0].tolist()
    prompt_length = len(input_ids)
    state = DecoderState()
    schema = SchemaContext(functions)

    for _ in range(max_tokens):
        # ─── PASO 2: Consulta SIN modelo (0ms de transformer) ───
        # compute_allowed_ids SIN logits: corre el filtro completo
        # para verificar cuántos candidatos hay. En steps deterministas
        # (key, colon, brace, etc.), esto es rápido (~1.6s root, <1ms en keys)
        # y nos dice si podemos skippear el forward.
        allowed = compute_allowed_ids(
            state, schema, vocab, trie, logits=None)

        # ─── SKIP-IF-SINGLE: token determinista → sin modelo ───
        if len(allowed) == 1:
            best_id = next(iter(allowed))
            token_text = vocab.id2decoded.get(best_id, "")
            input_ids.append(best_id)
            state.update_from_text(token_text)
            schema.update(state)
            if state.phase is DecoderPhase.COMPLETE:
                break
            continue                              # ← SKIP: sin forward, sin argmax

        if not allowed:
            break

        # ─── STEP AMBIGUO: consultar al modelo ───
        logits = model.get_logits_from_input_ids(input_ids)  # ← SOLO aquí: 2.6s

        # PASO 1: Top-K masking reemplaza el filtro completo
        allowed = compute_allowed_ids(
            state, schema, vocab, trie, logits)    # ← Top-K: ~5ms

        if not allowed:
            break

        best_id, token_text = _pick_best_token(
            allowed, logits, state, schema, functions, vocab, trie)

        if best_id is None:
            break

        input_ids.append(best_id)
        state.update_from_text(token_text)
        schema.update(state)

        if state.phase is DecoderPhase.COMPLETE:
            break
    ...
```

**Cambios específicos:**

| Cambio | Ubicación | Descripción |
|---|---|---|
| **A. Reordenamiento del flujo** | Línea 100 | `compute_allowed_ids` se llama PRIMERO sin logits (o con fast-path de filtro parcial) antes del forward. |
| **B. Skip-if-single** | Nuevo bloque entre filter y forward | Si `len(allowed) == 1`: commitear directo, `continue` sin llamar al modelo. |
| **C. Forward condicional** | Línea 100 | `model.get_logits_from_input_ids` solo se ejecuta SI `len(allowed) > 1` (step ambiguo). |
| **D. Top-K masking en step ambiguo** | Llamada a `compute_allowed_ids` con `logits` | El filtro Top-K del Paso 1 se activa solo cuando hay logits disponibles (step ambiguo). |

#### 2.4.2 Nota sobre el costo del skip-if-single

Una preocupación legítima: "¿no se está reemplazando un forward de 2.6s por un filter completo de 7–8.7s?"

**Respuesta: NO, por la naturaleza de los steps deterministas:**

| Tipo de step | Frecuencia | Filter cost (wildcard) | Filter cost (determinista) |
|---|---|---|---|
| `{`, `"`, `:`, `}`, `,` | ~40–50% del output | N/A (no wildcard) | **<1ms** (1 solo candidato por bucket chico) |
| Key de estructura (`"name"`, `"parameters"`) | ~15–20% | N/A (trie fuerza 1 candidato) | **<1ms** (trie pruning) |
| Continuación de string (wildcard `*`) | ~30–40% | **7–8.7s** ← AQUÍ está el costo | Forward SIEMPRE necesario (ambiguo) |

En los steps deterministas (que son el 60–70% del output), el filter **NUNCA entra en wildcard** — el `expected_first_chars()` retorna un set finito de 1–3 caracteres, y el bucket de esos caracteres es chico (~10–100 tokens). El filter completo en esos casos toma **~0.1–1.6s** (ROOT: 1.6s; keys: <1ms).

**PERO** hay un subproblema residual: si el filter completo sin wildcard toma 1.6s (ROOT) o ~0.5s (keys), el skip de 65 steps deterministas × 0.5–1.6s = 32–104s de filter Python puro. Eso sigue siendo mucho.

**Mitigación para el Paso 2:** Para los steps deterministas, podemos usar un **pre-compute del trie** que retorne directamente el set de IDs del token sin pasar por `simulate()`:

```python
# Fast-path para steps no-wildcard con 1 bucket conocido:
if "*" not in expected_chars and len(expected_chars) == 1:
    char = next(iter(expected_chars))
    candidate_ids = vocab.tokens_starting_with.get(char, set())
    if len(candidate_ids) == 1:
        return candidate_ids  # ← O(1): 1 bucket, 1 token
    elif len(candidate_ids) <= 100:
        # Simular solo los ~100 tokens de este bucket
        for token_id in candidate_ids:
            ...
```

Esto reduce el filter en steps deterministas de ~1.6s a **<10ms**.

---

## 3. MAPA DE IMPACTO CRUZADO EN MÓDULOS DEL PROYECTO

### 3.1 Diagrama de dependencias afectadas

```
src/
├── decoder/
│   ├── constrained_generator.py  ← SE MODIFICA (Paso 2: skip-if-single)
│   ├── token_filter.py           ← SE MODIFICA (Paso 1: Top-K masking)
│   ├── state.py                  ← SIN CAMBIOS (consumido, no mutado)
│   ├── schema_validator.py       ← SIN CAMBIOS (consumido, no mutado)
│   ├── trie.py                   ← SIN CAMBIOS (consumido, no mutado)
├── prompt/
│   └── prompt_builder.py         ← SIN CAMBIOS
├── loader/
│   ├── function_loader.py        ← SIN CAMBIOS
│   ├── input_loader.py           ← SIN CAMBIOS
│   └── vocab_loader.py           ← SIN CAMBIOS
├── models/
│   └── function_definition.py    ← SIN CAMBIOS
├── pipeline.py                   ← SIN CAMBIOS
├── __main__.py                   ← SIN CAMBIOS

llm_sdk/
└── llm_sdk/__init__.py           ← SIN CAMBIOS (SDK intocable)
```

### 3.2 Contratos de interfaz afectados

| Contrato | Cambio | Retrocompatible? |
|---|---|---|
| `compute_allowed_ids(state, schema, vocab, trie, logits)` | `logits` pasa de obligatorio a `Optional` (default `None`) | ✅ Sí (default None = comportamiento previo) |
| `generate(model, prompt, vocab, functions, trie, max_tokens)` | Firma SIN CAMBIO (la optimización es interna) | ✅ Sí |
| `_pick_best_token(allowed, logits, ...)` | SIN CAMBIO en firma | ✅ Sí |
| `DecoderState.simulate()` / `SchemaContext.allows_token()` | SIN CAMBIO (consumidos por el filter) | ✅ Sí |

### 3.3 Impacto en la suite de tests

| Archivo de tests | Impacto | Acción requerida |
|---|---|---|
| `tests/test_token_filter.py` | **MEDIO** | Nuevos test cases para Top-1 aceptado, Top-1 rechazado → fallback Top-K, Top-K vacío → fallback completo. |
| `tests/test_constrained_generator.py` | **MEDIO** | Verificar que `len(allowed)==1` produce el mismo output que el flujo anterior. Test de equivalencia: comparar output viejo vs nuevo en los 12 tests existentes. |
| `tests/test_state.py` | **CERO** | Sin cambios. |
| `tests/test_trie.py` | **CERO** | Sin cambios. |
| `tests/test_schema_validator.py` | **CERO** | Sin cambios. |
| Tests nuevos (recomendados) | **ALTO** | Test de integración E2E: medir que 11 prompts ≤5 min en la VM (validación del KPI). |

---

## 4. ANÁLISIS DE REPERCUSIÓN Y RIESGOS

### 4.1 Matriz de riesgos

| # | Riesgo | Probabilidad | Severidad | Causa raíz | Mitigación |
|---|---|---|---|---|---|
| **R1** | Truncamiento de tokens válidos por Top-K insuficiente | Baja | Media | Si el token óptimo tuviera logit fuera del Top-K=2000 | Qwen3-0.6B asigna ~99% de masa de probabilidad a los primeros ~1000 tokens; K=2000 es conservador. Fallback dinámico si Top-K vacío. |
| **R2** | Regresión de accuracy (pierde prompts que antes pasaban) | Baja | Alta | El Top-1 opportunistic podría elegir un token "técnicamente válido" pero semánticamente subóptimo | Verificar: Top-1 solo se acepta si pasa simulate() + allows_token() = MISMA validación que antes. Si pasa, el resultado es idéntico. |
| **R3** | Skip-if-single toma token determinista "equivocado" | Muy baja | Baja | El modelo con logits habría elegido un token distinto | Si solo hay 1 token legal, el modelo NO tiene alternativa: cualquier otro token rompería el JSON. El resultado es determinista por construcción. |
| **R4** | El filter sin logits (skip path) es más lento que antes | Media | Baja | compute_allowed_ids(sin logits) corre el filtro completo en steps no-wildcard | Mitigar con pre-compute del trie (sección 2.4.2): bucket chico → O(1); wildcard → forward necesario. |
| **R5** | El Pase Fino (Inciso 4.1.1) choca con el skip-if-single | Muy baja | Media | `_passes_fine_validation` se ejecuta después del argmax, no antes del skip | El skip ocurre ANTES del argmax; si hay 1 candidato, no hay argmax ni pase fino. El pase fino solo aplica cuando hay ambigüedad (len>1). |
| **R6** | El pase fino descarta el único candidato de allowed | Muy baja | Alta | `_pick_best_token` podría encontrar que el único candidato no pasa `_passes_fine_validation` | Revisar si el pase fino es consistente con compute_allowed_ids: si el filter ya simuló el token, el pase fino debería aceptarlo (misma máquina + schema). Verificar con test explícito. |
| **R7** | Top-K sorting en Python puro es lento para K=2000 | Muy baja | Baja | `sorted(range(N), key=lambda i: logits[i], reverse=True)[:K]` sobre 151K logits | Usar `heapq.nlargest(K, range(N), key=logits.__getitem__)` que es O(N log K) ≈ 151K × 11 ≈ 1.6M operaciones ≈ <5ms. O usar numpy/torch si disponible. |

### 4.2 Análisis de invariantes preservados

| Invariante | ¿Se preserva? | Justificación |
|---|---|---|
| **100% JSON válido** | ✅ Sí | El filter mantiene las mismas cláusulas de validación (Fase 2+3). Top-K solo reduce el UNIVERSO de candidatos, no cambia la validación. |
| **Accuracy ≥90%** | ✅ Sí (esperado) | Top-1 Opportunistic: si pasa simulate+allows, es el MISMO token que antes. Top-K: el argmax sobre Top-K validado ≈ argmax sobre full (ver R1). |
| **MAX_TOKENS=200 safety net** | ✅ Sí | El loop mantiene el mismo rango de iteración. |
| **Pase fino post-argmax (Inciso 4.1.1)** | ✅ Sí | Solo se ejecuta cuando hay ambigüedad (len>1). En skip-if-single, no hay argmax ni pase fino. |
| **State machine atómica** | ✅ Sí | `state.update_from_text()` no cambia. El skip solo omite la llamada al modelo, no la actualización del estado. |
| **SchemaContext sincronizado** | ✅ Sí | `schema.update(state)` se ejecuta tanto en skip como en step ambiguo. |
| **Separación state ↔ schema** | ✅ Sí | El skip no mezcla sintaxis (state) con semántica (schema). |
| **Contrato de generate(): tuple[str, bool]** | ✅ Sí | Firma y semántica de retorno sin cambios. |

### 4.3 Escenario de regresión: ¿cómo se detectaría?

1. **Suite existente (164 tests):** Si el skip-if-single o el Top-K modificaran el comportamiento, al menos 1 test existente fallaría (especialmente los 12 de `test_constrained_generator.py` que usan FakeModel con argmax sobre targets fijos).

2. **Task 4.3 re-run:** Re-correr el script `task43_accuracy.py` con las optimizaciones. Si accuracy baja de 91%, el Top-K es demasiado agresivo (aumentar K o revertir).

3. **Test de equivalencia nuevo:** Para cada prompt de los 11, comparar el output del generador ANTES y DESPUÉS de las optimizaciones. Si divergencia >0 prompts → investigar.

---

## 5. ESTIMACIÓN DE MEJORA ESPERADA

### 5.1 Desglose por componente

| Componente | Actual (por prompt) | Con Paso 1 | Con Paso 1+2 | Notas |
|---|---|---|---|---|
| Forward del modelo | 70 × 2.6s = 182s | 70 × 2.6s = 182s | **5 × 2.6s = 13s** | Paso 2: skip en steps deterministas |
| Filter (wildcard) | 40 × 8s = 320s | **40 × 0.005s = 0.2s** | 0.2s | Paso 1: Top-K masking |
| Filter (determinista) | 30 × 1.6s = 48s | 30 × 1.6s = 48s | **30 × 0.01s = 0.3s** | Paso 2: pre-compute trie |
| **Total por prompt** | **~550s ≈ 9 min** | **~230s ≈ 4 min** | **~13.5s** | |
| **Total 11 prompts** | **~34.9 min** | **~42 min** ❌ | **~2.5 min** ✅ | |

**Nota:** El Paso 1 solo (sin Paso 2) NO alcanza porque el forward sin KV-cache sigue siendo 182s/prompt. El Paso 2 es el que rompe la barrera al eliminar 65 de 70 forwards. **Ambos pasos son NECESARIOS.**

### 5.2 Proyección contra el KPI

| Métrica | KPI | Proyección Paso 1+2 | Margen |
|---|---|---|---|
| Latencia 11 prompts | ≤300s (5 min) | **~165s (2.75 min)** | ✅ +135s de margen |
| JSON válido | 100% (11/11) | 100% (sin cambio en validación) | ✅ |
| Accuracy (M14) | ≥90% | ≥91% (sin cambio en argmax) | ✅ |
| Suite tests | 164 green | 164+ green (tests nuevos) | ✅ |

---

## 6. INTEGRACIÓN CON PLAN_IMPLEMENTACION.md

### 6.1 Ubicación en el plan de fases

| Fase | Task | Estado actual | Con este anexo |
|---|---|---|---|
| Phase 4 | 4.1 (constrained generator) | ✅ Completa | Se modifica internamente (skip-if-single) |
| Phase 4 | 4.2 (smoke test) | ✅ Completa | Se re-ejecuta post-optimización |
| Phase 4 | 4.3 (11 prompts) | ✅ Completa (accuracy verde, timing rojo) | **Se re-ejecuta post-optimización → timing verde esperado** |
| Phase 7 | B5 (Performance) | Pendiente (stub en L801-848) | **Este anexo ES la implementación de B5** |
| Phase 6 | 6.4 (DoD E2E) | Pendiente | Se validará con el timing optimizado |

### 6.2 Relación con el Inciso B5 existente (PLAN_IMPLEMENTACION L801-848)

El Inciso B5 del plan ya documentaba:
- ✅ KV-cache descartado (SDK intocable)
- ✅ Batch processing descartado (SDK single-sequence)
- ✅ `__slots__` ya implementado
- ✅ "opportunistic masking post-profiling" recomendado

**Este anexo EJECUTA la recomendación del B5.** La diferencia es que el B5 era un stub y este anexo contiene el diseño detallado con evidencia de profiling.

### 6.3 Relación con el Inciso 4.1.1 (Pase fino post-argmax)

El pase fino del Inciso 4.1.1 (`_passes_fine_validation`) se ejecuta DESPUÉS del argmax, solo cuando hay ambigüedad. Con el skip-if-single:
- En steps deterministas (len==1): no hay argmax → **no hay pase fino** → ahorro de overhead adicional.
- En steps ambiguos (len>1): el pase fino se ejecuta como antes → sin cambio.

**No hay conflicto:** el pase fino y el skip-if-single operan en momentos diferentes del pipeline.

---

## 7. ORDEN DE IMPLEMENTACIÓN RECOMENDADO

| Paso | Archivo | Dependencias | Estimación |
|---|---|---|---|
| **1. Pre-compute del trie para fast-path no-wildcard** | `token_filter.py` | Ninguna | 30 min |
| **2. Top-1 Opportunistic + Top-K Masking** | `token_filter.py` | Paso 1 | 1h |
| **3. Tests del Paso 1** | `test_token_filter.py` | Paso 2 | 30 min |
| **4. Skip-if-single en generate()** | `constrained_generator.py` | Paso 2 | 30 min |
| **5. Tests del Paso 2** | `test_constrained_generator.py` | Paso 4 | 30 min |
| **6. Suite completa + lint** | todos | Pasos 3,5 | 15 min |
| **7. Re-run Task 4.3 (11 prompts, timing)** | `.scratch_task42/task43_accuracy.py` | Paso 6 | 3 min |
| **8. Commit + docs** | docs/ + main | Paso 7 | 10 min |
| **TOTAL estimado** | | | **~4h** |

---

## 8. REFERENCIAS

| Referencia | Ubicación | Relevancia |
|---|---|---|
| Inciso B5 (stub original) | `PLAN_IMPLEMENTACION.md` L801-848 | Recomendación de "opportunistic masking" que este anexo ejecuta |
| Inciso 4.1.1 (pase fino) | `PLAN_IMPLEMENTACION.md` | Mecanismo existente que convive con el skip-if-single |
| Profiling del filter | `.scratch_task42/profiling_filter.py` | Evidencia de 7.15–8.67s/step en wildcard |
| Bench de scaling | `.scratch_task42/bench_scaling.py` | Evidencia de hardware: threads=4 óptimo (2611ms), threads=6 empeora |
| Task 4.3 resultados | `.scratch_task42/results_43.json` | Accuracy 10/11 (91%), timing 34.9 min |
| Mario: generation_engine | `Mario_Call_me_maybe/src/engine/generation_engine.py:147-148` | Skip-if-single: `if len(valid_token_ids) == 1: return valid_token_ids[0]` |
| Mario: constraint_engine | `Mario_Call_me_maybe/src/engine/constraint_engine.py:243-386` | Filter O(candidates) vs nuestro O(vocab) |

---

*Documento generado el 18 de septiembre de 2026 como parte de la auditoría post-Task 4.3.*
*Pendiente de integración al PLAN_IMPLEMENTACION.md central tras aprobación del usuario.*
