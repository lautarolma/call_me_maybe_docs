# CONTEXTO_REFACTOR — call_me_maybe (42 School)

> **Archivo de referencia ON-DEMAND**: NO se carga automáticamente. Lo importa el `CLAUDE.md` del proyecto y la tarea de refactor de latencia (la instrucción inicial del CLAUDE.md dice leerlo ANTES de tocar código). Fuente de verdad de la spec: `docs/design/PLAN_IMPLEMENTACION.md`. Subject transcrito en §7 — no hace falta decodificar el PDF.

---

## 1. OBJETIVO INMEDIATO (la razón de ser de este refactor)

> Bajar la latencia end-to-end hasta cumplir el KPI del subject (**<5 min para los 11 prompts**, hoy 34.9 min) refactorizando **`src/decoder/token_filter.py`** (+ el uso que hace **`src/decoder/constrained_generator.py`**) con **DOS optimizaciones complementarias**, usando SOLO estructuras ya existentes en `src/` (nada de herramientas nuevas).

### 1.1 Problema técnico (con evidencia, no estimación)

1. **El pre-filtro Fase 1 es ciego en wildcard**: cuando `expected_first_chars()` devuelve `*` (KEY_START/IN_KEY e IN_STRING_VALUE — `state.py L183-192`), `compute_allowed_ids` toma **TODOS los buckets del pre-índice** (`token_filter.py L112-116`) → candidate set ≈ **151K tokens**. La spec A6.4 del plan YA exigía "el key parcial es prefix de un key conocido" y "el parcial es prefix de un nombre válido" — pero eso se valida recién en Fase 3 (schema/trie), DESPUÉS del loop, no para construir el candidate set.
   - **Origen = hueco de DISEÑO del plan, NO bug de implementación**: `PLAN_IMPLEMENTACION.md` A6.4 (Fase 1) asume que `expected_first_chars()` siempre trae chars concretos y NUNCA especifica el caso `*` en la construcción del candidate set; la validación de prefijo queda relegada a Fase 3. La implementación (`token_filter.py L112-116`) es la interpretación literal de ese diseño incompleto — el código es correcto según el plan, pero el plan no alcanza el KPI por diseño. Opt1 (prefijos validados) corrige el hueco en el origen (Fase 1), no en el síntoma (Fase 3).
2. **El forward es carísimo** (~2.6s/step) y se llama en cada step ambiguo. M5 ya lo saltea cuando hay 1 candidato, pero en wildcard `len>1` casi siempre → forward inevitable.
3. **El contexto importa**: al llegar a un wildcard con solo el prompt como contexto (sin el JSON ya generado), los logits están dispersos y M2 necesita recorrer más tiers. Con el header/prefijos ya emitidos, el contexto completo concentra los logits → M2 pega en el primer tier.

### 1.2 Mediciones reales (Task 4.3, 18/09, modelo Qwen3-0.6B, threads=4)

- Accuracy función: 11/11 (100%) · Accuracy full (M14): 10/11 (**91% ≥ 90%** ✅).
- Timing: **34.9 min para 11 prompts** — KPI del subject (<5 min) **INCUMPLIDO** ❌.
- Forward real: **~2.6s/step** (SDK sin KV-cache; ~70 steps/prompt → ~182s/prompt solo en forwards).
- Filter wildcard (ANTES de M1/M2): 7.15–8.67s/step (60–70% de la latencia, ahora mitigado por M2).
- El plan estimaba `get_logits_from_input_ids` en ~150-200ms → la realidad es **13-17x peor**; la estimación global del plan (~110s) quedó **~19x por debajo** de los 34.9 min reales.

### 1.3 M1-M5 ya implementados en HEAD (Anexo de Optimización de Latencia, `docs/design/ANEXO_OPTIMIZACION_LATENCIA.md`)

- M1: Top-1 opportunistic — si el top-1 por logits pasa el filter → O(1), sin recorrer candidatos.
- M2: Top-K masking escalonado (TIER_SIZES = [1,5,10,20,...,2000]) — valida por tandas hasta encontrar un token válido.
- M5: skip-if-single — si `compute_allowed_ids(sin logits)` devuelve exactamente 1 candidato y no hay `*` en `expected_first_chars()`, se saltea el forward (~2.6s) e inyecta el ID directo.

---

## 2. PLAN DE REFACTOR

### 2.1 Opt1 — WILDCARDS POR PREFIJOS VALIDADOS (filtro estructural, SIN modelo)

**Qué NO es**: pasarle el vocabulario completo al forward. **Qué ES**: construir `candidate_ids` de Fase 1 desde el **prefijo acumulado** en el estado, no desde `*`:

- **Keys** (depth 0 y 1): usar `state.current_key` parcial + el conjunto de keys permitidas por depth (`{"name","parameters"}` en depth 0; los param names de la función resuelta en depth 1 — los conoce `SchemaContext`) → `next_chars = {ch : current_key+ch es prefijo de una key permitida}` → candidate_ids = unión de buckets de esos chars. Resultado: decenas de candidatos (a veces exactamente 1 → M5 saltea el forward).
- **Value de "name"** (depth 0): usar `state.name_buffer` parcial → `next_chars = valid_next_chars(trie_de_funciones, name_buffer)` (`trie.py` YA lo expone) → misma lógica.
- **Mecanismo ya existente para reutilizar**: `trie.valid_next_chars()` + `trie.find_node()` (para keys de parameters hay que construir un trie de keys por depth, o derivarlo del schema — igual de barato, MISMO patrón que `build_trie`).
- **Cuidado**: los strings libres (valores string SIN schema) NO tienen restricción de prefijo — el wildcard residual queda para ellos, y se resuelve con M2 + contexto (Opt2). Documentar esa distinción en el docstring.

### 2.2 Opt2 — HEADER ESTÁTICO CON `encode()` (inyección de IDs deterministas, patrón Mario)

**Qué ES**: el output JSON arranca SIEMPRE con estructura fija (`{"name": "`... hasta el primer valor variable, incluyendo el whitespace de formato natural que Qwen produce: `{\n  "name": "`). Esa parte estática se tokeniza UNA vez con `model.encode(header)` y los token IDs resultantes se inyectan **directo a `input_ids` SIN forward** — cada uno es el único candidato posible ya validado por construcción:

- El state machine los consume en el step-by-step igual que siempre (`state.update_from_text`), pero **sin consultar el modelo**: mismo efecto que M5 pero **precomputado** (M5 descubre el candidato único corriendo el filter; Opt2 lo sabe ANTES, con costo cero de filter).
- Beneficio doble: (a) ~15-25 forwards eliminados por prompt en la estructura; (b) cuando llega el primer wildcard real (el value de "name"), el modelo ya tiene el JSON anterior como contexto → logits concentrados → M2 resuelve en el tier 1.
- **Implementación sugerida**: en `generate()`, antes del loop: `header_ids = model.encode(header_template)[0].tolist()`; durante los steps iniciales (mientras el estado esté en la parte determinista del header) consumir `header_ids[pos]` sin llamar al filter ni al modelo. Requiere un pequeño oráculo: "¿el siguiente char/token del header sigue siendo válido en el estado actual?" (un `expected_first_chars` que matchee, o consumir por chars vía state machine).
- **OJO formato**: el header debe capturar el formato natural del modelo (`{\n  "name": "` con newlines/indent), NO un JSON minificado — la política compacta YA falló (registro de decisión en ANEXO: commit abortado `7bf38ed`).

### 2.3 Las DOS van juntas (esto es lo que había que entender)

- Opt1 (prefijos validados) **encoge el candidate set** en los wildcards → filter barato y más skip-if-single.
- Opt2 (header con encode) **elimina forwards** en la estructura estática → contexto previo completo → M2 efectivo en el wildcard residual.
- En el proyecto de Mario ambas convivían: el state machine generaba token por token, pero en cada estado recibía el tokenID ya validado como **único candidato** (los del header/estructura), evitaba el forward, inyectaba el ID directo (generando contexto para el próximo token) y al llegar a un wildcard llamaba al forward con TODO el JSON previo como contexto → muchos menos candidatos que pasarle el vocabulario total. Tu estructura actual hace lo del forward con contexto casi vacío → por eso explota el candidate set.

### 2.4 Restricciones del refactor (NO negociables)

- **Solo estructuras ya existentes en `src/`**: state machine, trie, schema_validator, pre-índice `Vocab.tokens_starting_with`, `Small_LLM_Model.encode` (el subject V.3.1 lo expone como método público del SDK — usarlo está permitido).
- **No tocar `llm_sdk`** (sin KV-cache, sin batch — ya descartados en B5/ANEXO: "SDK intocable").
- **No romper**: 164 tests, `make lint` (flake8 + mypy), accuracy 91%.
- **Sin complejidad ajena**: si la solución requiere una herramienta/estructura que no existe en `src/`, es señal de MAL diseño — parar y proponer alternativa con tradeoffs.
- **Criterio de éxito medible**: re-correr Task 4.3 (`.scratch_task42/task43_accuracy.py`, `HF_HUB_OFFLINE=1`, `torch.set_num_threads(4)`) → timing < 5 min manteniendo accuracy ≥ 90% y JSON 100% válido.

### 2.5 FASE 2 (25/09): oráculo por estado — IMPLEMENTADO (Nivel 1)

- **Qué**: generalizar el 2º tramo de Opt2 (entre el cierre de `name` y `"parameters": {`) a un oráculo por estado completo: tabla de tramos keyed por estado (VALUE_END/IN_OBJECT/PARAMS_OBJECT × depth × gates), con **E** (whitespace ya emitido) y **K** (próxima key canónica pendiente). `_next_static_text(state, schema, emitted)` reemplaza al trigger lineal.
- **Estado (25/09 noche)**: ✅ implementado en el working tree (`constrained_generator.py` + tests, SIN commitear). T1–T6 del Nivel 1 como funciones puras en `_TRAMPS`; `emitted` acumulado por iteración en `generate()`; `_inject_static_header` extendido con acumulador opcional. Suite 195 green + flake8 + mypy.
- **Por qué**: el trigger lineal `current_key=="name"` rompió (BUG-013 — matchea el param interno `"name"` tras cerrar `parameters`); el gate robusto es **N** (`depth==0 ∧ current_key=="name" ∧ keys_enclosed==∅`) — reemplaza a `¬has_seen_params_object()` que es sticky y no cubre tokens fusionados (D8, ANEXO).
- **Gates**: T1/T2 (apertura de `parameters`) exigen N=1; T3/T4 keys dentro de parameters exigen función resuelta y key pendiente (`required_keys_remaining()`); T5/T6 cierre exigen keys completas (N=0 ∧ ρ=0 en T6 — sin P, ver D8).
- **No tocar**: `state.py` ni `schema_validator.py` (el oráculo vive en `generate()`/`constrained_generator.py`; `emitted` se acumula en el loop, no en el estado). ✅ respetado.
- **fn_empty (0 params)**: probe real confirmó cierre INLINE (`"parameters": {}`) **y** `SUCCESS=True` post-fix (25.1s) — el `}` del ROOT lo inyecta T6; el caso entra por N=0 (tramo gratis ord=∅ → el modelo emite el `{}` fusionado).
- **B′ (autocompletar fn_name por trie)**: validado con probe de unicidad, DIFERIDO hasta métricas del Modelo A (suite completa).
- Registro completo de decisiones (lo abordado/descartado y las refutaciones): `docs/design/ANEXO_OPTIMIZACION_LATENCIA.md` → "Registro de decisiones — FASE ORÁCULO (25/09)" (D1–D8).

---

## 3. ARQUITECTURA ACTUAL (mapa rápido — orientación sin leer todo)

```
src/
├── __main__.py, cli.py, pipeline.py      # entry points + orquestación (Phase 1)
├── loader/                               # input_loader, function_loader, vocab_loader
│   └── vocab_loader.py                   # Vocab: id2token, id2decoded, tokens_starting_with (pre-índice por 1er char)
├── models/                               # pydantic: FunctionDef, FunctionCall, prompt models
├── prompt/prompt_builder.py              # template de prompt (enumera funciones)
└── decoder/                              # ★ CORAZÓN del proyecto ★
    ├── state.py                          # DecoderState (dataclass slots): state machine JSON + expected_first_chars()
    ├── trie.py                           # build_trie/find_node/valid_next_chars/is_complete_name (nombres de función)
    ├── schema_validator.py               # SchemaContext.allows_token(): 4 cláusulas (name→trie, param key, value type, params close)
    ├── token_filter.py                   # compute_allowed_ids(): Fase 1 pre-filtro → Fase 2 simulate → Fase 3 schema
    └── constrained_generator.py          # generate(): loop principal, M1/M2/M5, pase fino post-argmax (Inciso 4.1.1)
```

**Flujo por token**: `generate()` → `model.get_logits_from_input_ids(input_ids)` (forward 2.6s) → `compute_allowed_ids(state, schema, vocab, trie, logits)` → `_pick_best_token` (argmax + pase fino) → `input_ids.append(best_id)` → `state.update_from_text(token_text)` → `schema.update(state)` → COMPLETE?

**Puntos de anclaje para el refactor**:
- Fase 1 wildcard: `token_filter.py L112-116` (skips pre-filter en `*`).
- Skip-if-single M5: `constrained_generator.py L100-119` (chequea `"*" not in state.expected_first_chars()` — Opt1 hará que más wildcards tengan 1 candidato y este check pueda saltar).
- Header estático: se inserta en `generate()` antes del loop (Opt2).
- Prefix keys: `state.current_key` (L100), `name_buffer` (L113), trie `valid_next_chars` (`trie.py L67`), keys por depth en `SchemaContext`.

---

## 4. SUBJECT EN TEXTO PLANO (requisitos REALES — transcrito de `docs/sources/en.subject.pdf`, sin decodificar el PDF)

### General rules (IV.1)
- Written in **Python 3.10 or later**.
- Must adhere to the **flake8 coding standard**.
- Handle exceptions gracefully (try-except, context managers); crashes during review = non-functional.
- Manage all resources properly (files, connections) → prefer context managers.
- **Type hints** for function params, return types, variables (typing module). Use **mypy** for static checking; all functions must pass mypy without errors.
- **Docstrings** in functions and classes following PEP 257 (Google or NumPy style): purpose, params, returns.
- HEAVY requirement: **all classes must use pydantic for validation** (IV.3.1).
- You can use the **numpy and json** packages ONLY.
- **FORBIDDEN**: dspy or any similar package, pytorch, huggingface package, transformers, outlines, etc.
- Models: **Qwen/Qwen3-0.6B (default)**; other models allowed as long as it works with Qwen3-0.6B.
- **The function to call must be chosen by the LLM**, not heuristics or any other medieval magic.
- **Forbidden to use any private methods/attributes from the llm_sdk package**.
- Create a venv and install numpy + pydantic using **uv**; copy `llm_sdk` next to `src`.
- Reviewer and moulinette run only `uv sync`.
- Never crash unexpectedly; always clear error messages.

### Makefile (IV.2) — mandatory rules
- `install`: install deps (pip/uv/pipx).
- `run`: execute main script.
- `debug`: run with Python built-in debugger (pdb).
- `clean`: remove temp files/caches (`__pycache__`, `.mypy_cache`).
- `lint`: run `flake8 .` and `mypy . --warn-return-any --warn-unused-ignores --ignore-missing-imports --disallow-untyped-defs --check-untyped-defs`.
- `lint-strict` (optional): `mypy . --strict`.

### Usage (IV.3.2)
`uv run python -m src [--functions_definition <file>] [--input <file>] [--output <file>]`
- Default input dir `data/input/`, output `data/output/`.

### Mandatory part (V)
- Build a **function calling tool**: translates natural-language prompts → structured function calls.
- MUST use **constrained decoding** to guarantee 100% valid JSON output, even with a small 0.6B model.
- **Schema-driven**: decoder restricts token selection to satisfy the schema from `functions_definition.json` (number → int/float, etc.). Every generated token maintains structural AND semantic validity.
- **NOT allowed**: relying on the model spontaneously producing correct JSON from a prompt ("prompting and hoping"). Must use the vocabulary JSON file to map tokens↔strings to decide valid tokens per step.
- Two input files (data/input/):
  - `function_calling_tests.json`: array of `{"prompt": "..."}`.
  - `functions_definition.json`: array of functions `{name, description, parameters:{name:{type}}, returns:{type}}`. Types: number/string/boolean (examples: fn_add_numbers, fn_greet, fn_reverse_string). Input files MAY change during review — do not hardcode.
  - Proper JSON error handling for inputs: invalid JSON or missing files.

### llm_sdk (V.3.1) — public methods (the ONLY allowed interface)
- `get_logits_from_input_ids(input_ids: List[int]) -> List[float]`
- `get_path_to_vocab_file() -> str`
- `encode(text: str) -> Tensor`
- `decode(token_ids: List[int]) -> str` (optional)

### Output file (V.4) — `data/output/function_calling_results.json`
- One JSON object per prompt, with EXACTLY these keys: `prompt` (string), `name` (string, function to call), `parameters` (object, all required args with correct types).
- Valid JSON (no trailing commas/comments); keys/types match schema exactly; no extra keys/prose; all required args present; arg types match definition.

### Performance and reliability (V.5) — KPIs
1. **Near-perfect accuracy: 90%+** correct function selection and argument extraction.
2. **100% valid JSON**: every output parseable and schema-compliant.
3. **Reasonable speed: ALL test prompts in under 5 minutes** on standard hardware.
4. Robust error handling: malformed inputs, missing files, edge cases.

### Testing (V.6)
Edge cases: empty strings, large numbers, special characters, wrong types, ambiguous prompts, functions with multiple parameters.

### README requirements (VI) — mandatory contents, in English
- **First line, italicized**: "This project has been created as part of the 42 curriculum by <login>."
- Description, Instructions (compilation/installation/execution), Resources (+ how AI was used, which tasks).
- PLUS: **Algorithm explanation** (constrained decoding approach in detail), **Design decisions**, **Performance analysis** (accuracy, speed, reliability), **Challenges faced**, **Testing strategy**, **Example usage**.

### Bonus (VII) — NOT required to pass, only after MVP
Multi-model support; recoding the tokenizer (avoid encode/decode, use get_logits_from_input_ids + get_path_to_vocab_file); advanced error recovery; performance optimizations (caching, batching); comprehensive test suite; visualization; complex nested function arguments; public tokenizer encode/decode; demo of encode/decode integration with constrained decoding. Bonus must be implemented and working, not just described.

### Submission (VIII)
Repo must contain: `src/`, `pyproject.toml` + `uv.lock`, `llm_sdk/` (copied), `data/input/` (test files), `README.md`. **Do NOT include the output/ directory** (generated during review). During evaluation you may be asked to make a small code modification on the spot — be prepared.

---

## 5. REFERENCIAS CLAVE

- `docs/design/PLAN_IMPLEMENTACION.md` — spec completa + tasks (Parte A arquitectura, Parte B bonus).
- `docs/design/ANEXO_OPTIMIZACION_LATENCIA.md` — mediciones, causa raíz, M1-M5, registro de decisión (política compacta abortada `7bf38ed`).
- `docs/design/CRONOGRAMA_TRABAJO.md` / `docs/tracking/CRONOGRAMA_GENERAL.md` — plan vs real (21/09 vencido; 03/10 general).
- `docs/tracking/PROGRESS_TRACKER.md` — bitácora de avance (última: 18/09, Task 4.3).
- `docs/notes/TeoricNotes.md` — teoría interna (cómo funciona cada módulo por dentro).
- `docs/tracking/SESSION_START_PROTOCOL.md` — protocolo de apertura de jornada.
- `.scratch_task42/` — scripts de prueba reales (smoke_42.py, task43_accuracy.py, bench_scaling.py, profiling_filter.py).