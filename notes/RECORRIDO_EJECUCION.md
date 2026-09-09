# 🧭 Recorrido de ejecución — call_me_maybe (repaso Phase 1 + Phase 2)

> Clase/revisión file-by-file siguiendo el flujo REAL del programa.
> Método: **por demanda** — el usuario guía el ritmo, hace preguntas y ordena avanzar.
> Inicio: 2026-09-04 (Día 10, tras completar Phase 2)

---

## 📍 Posición actual del recorrido

- **Parada 8 EN REVISIÓN**: `src/loader/vocab_loader.py` — presentada y documentada (BUG-003), pero **pendiente de re-explicación** (usuario desorientado con unicode/decodificación, 2026-09-09 noche). **Retomar acá.**
- **Próximo archivo en análisis (tras confirmar la 8)**: `src/prompt/prompt_builder.py` (parada 9, ← Phase 2)
- **Progreso**:
  - [x] 1. `src/__main__.py`
  - [x] 2. `src/cli.py` (dudas resueltas: argparse -h/help, exit codes 0/1/2, Path vs str → ver TeoricNotes.md)
  - [x] 3. `src/pipeline.py`
  - [x] 4. `src/models/output.py` (encuadre "ficha de registro" de pydantic)
  - [x] 5. `src/models/function_definition.py` (confirmada 2026-09-06)
  - [x] 6. `src/loader/input_loader.py` (confirmada 2026-09-07; short-circuit/orden de condiciones → ver TeoricNotes.md)
  - [x] 7. `src/loader/function_loader.py` (confirmada 2026-09-08; asimetría con input_loader → BUG-002, ver BITACORA_BUGS.md)
  - [ ] 8. `src/loader/vocab_loader.py`
  - [ ] 9. `src/prompt/prompt_builder.py` ← Phase 2
  - [ ] (opcional) Tests como referencia cruzada de cada capa

---

## 🗺️ Ruta de ejecución (el orden del mapa es el flujo real)

```
uv run python -m src
        │
        ▼
 1. src/__main__.py   → main(): parse_args() → run(args) → sys.exit(exit_code)
 2. src/cli.py        → parse_args(): argparse + defaults (data/input/*.json)
 3. src/pipeline.py   → run(): orquesta load → model → generate → validate → output
        │
        ▼   (los bloques que ya existen hoy)
 4. src/models/output.py            → FunctionCall, JSONValue (forma del resultado)
 5. src/models/function_definition.py → FunctionDef, ParameterDef, ParameterType
 6. src/loader/input_loader.py      → load_prompts (las 11 queries)
 7. src/loader/function_loader.py   → load_functions (las 5 funciones)
 8. src/loader/vocab_loader.py      → load_vocab + preindexación (151K tokens)
 9. src/prompt/prompt_builder.py    → build_prompt (3 partes) ← Phase 2
        │
        ▼   (lo que llega en Phase 3-4-5: decoder, generator, validator, pipeline completo)
```

---

## 📝 Notas por archivo (se completa ACÁ a medida que avanzamos)

> Cada archivo se analiza por demanda. Acá van: qué hace, cómo lo hace,
> decisiones de diseño, y las dudas que fuimos resolviendo.

### 1. `src/__main__.py`

**Qué hace**: punto de entrada del programa. Orquesta: parsea args → corre el pipeline → propaga el exit code al SO. NO tiene lógica de negocio (eso vive en cli.py/pipeline.py).

**Cómo lo hace**:
- `uv run python -m src`: uv activa el venv → python busca el paquete `src` en sys.path → ejecuta `__main__.py` como script (por eso `if __name__ == "__main__"` corre).
- `main() -> int`: `parse_args()` (cli.py) + `run(args)` (pipeline.py). Devuelve un int, NO llama sys.exit() adentro → main queda pura y testeable.
- Cierre: `sys.exit(main())` dentro de try/except que atrapa CUALQUIER excepción → mensaje limpio en stderr + exit 1 (sin traceback para el usuario final).

**Decisiones de diseño**:
1. Separación entrada/lógica → cli.py y pipeline.py son importables SIN ejecutar nada (testeables, sin efectos colaterales).
2. `main()` devuelve int → convención POSIX de exit codes (0 = éxito, ≠0 = fallo). El int se propaga al SO vía sys.exit.
3. Catch-all `except Exception` → UX limpia ANTES que traceback. Tradeoff aceptado: perdés el stack trace; en desarrollo conviene comentarlo temporalmente.
4. `print(f"ERROR: ...", file=sys.stderr)` → stdout es para DATOS (redirigible a archivo/pipes), stderr para DIAGNÓSTICO. `python -m src > salida.txt` nunca contamina los datos.

**Detalles**: `sys.argv[1:]` lo consume parse_args automáticamente cuando recibís `argv=None`; `from __future__ import annotations` da hints modernos en Python viejo.

---

### 2. `src/cli.py`

**Qué hace**: parsea los argumentos de línea de comandos con `argparse`. Declara 3 flags (`--functions_definition`, `--input`, `--output`), cada uno con su default `Path`.

**Cómo lo hace**:
- `parse_args(argv=None)`: si recibís `None`, argparse lee `sys.argv[1:]` por su cuenta. Devuelve un `argparse.Namespace` (objeto-bolsa con acceso por atributo: `args.input`).
- `ArgumentParser` es un registro declarativo: cada `add_argument` agrega nombre/tipo/default/help. `-h/--help` se genera AUTOMÁTICAMENTE.
- `type=Path` NO es solo tipado: es un CALLABLE que argparse ejecuta sobre el string crudo → `Path(valor)`. Si lanza excepción, argparse aborta solo (exit 2).
- Los defaults son `Path` **relativos al CWD**, no a la ubicación del archivo; se evalúan UNA vez en importación (Path es inmutable → compartirlo es seguro).

**Decisiones de diseño**:
1. Defaults en constantes de módulo → visibles/documentables, no enterrados en add_argument.
2. Cero validación manual de args: argparse ya aborta con `sys.exit(2)` ante flag desconocido/requerido faltante — el entry point nunca ve ese caso.
3. Sin flags requeridos: todo tiene default → `uv run python -m src` funciona sin argumentos (subject-friendly).
4. `--functions_definition` con guión bajo (no `-`) → se mapea al atributo `functions_definition` directo.

**Gotchas**: CWD-dependence de los defaults (correr desde otra carpeta rompe las rutas); argparse nunca ejecuta nuestro código para errores de CLI (exit 2 propio).

---

### 3. `src/pipeline.py`

**Qué hace**: ORQUESTADOR del pipeline (skeleton de Phase 1). Coordina a los especialistas en orden y propaga excepciones hacia `__main__.py` (que las convierte en exit 1). NO carga nada él mismo: delega.

**Cómo lo hace** — `run(args) -> int`, 4 pasos con print de progreso:
1. `load_functions(args.functions_definition)` → list[FunctionDef] (JSON → pydantic → duplicados check)
2. `load_prompts(args.input)` → list[str] (acepta array de strings o de {"prompt": ...}; rechaza vacías)
3. `Small_LLM_Model()` → model (Qwen3-0.6B default; device mps>cuda>cpu; dtype f16 en GPU / f32 en CPU; AutoTokenizer + AutoModelForCausalLM; .eval(); requires_grad=False)
4. `load_vocab(model)` → Vocab (lee vocab.json del modelo + pre-indexa tokens por PRIMER CARÁCTER DECODIFICADO → base del decoder restringido, O(1) vs escanear 150k+)

Remata con summary (len functions/prompts, vocab_size, buckets, output path) y `return 0`. El print final dice "Generation arrives in Phase 3" — el skeleton está pensado para crecer (Phase 5 lo extiende con Task 5.2).

**Decisiones de diseño**:
1. Orquestador fino: `run()` no sabe cargar JSON ni correr tensores; conoce SOLO el orden y delega. Testeable y cada falla se propaga como excepción.
2. Progreso en stdout con `[1/4] [2/4]...`: feedback progresivo para el usuario (y el evaluador ve que avanza).
3. `Small_LLM_Model()` sin args usa el default del SDK → cero configuración de modelo en el CLI.
4. Pre-indexación por primer carácter = decisión arquitectónica CLAVE para el límite de 5 min del subject (Fase 3-5).

**Gotchas/nota**: RAM 3.8GB → el paso 3 OOM-killea sin los flags MALLOC_ARENA_MAX=1 OMP_NUM_THREADS=1 MKL_NUM_THREADS=1 (exit 137). El tokenizer es el mismo vocab que usa el decoder restringido.

---

### 4. `src/models/output.py`

**Qué hace**: define `JSONValue` (unión de los 5 escalares JSON) y `FunctionCall` (name + parameters) — la FORMA de la SALIDA del sistema. Es la capa I/O: lo que el decoder restringido va a producir por cada prompt (Phase 4) y que se serializa a function_calls.json (Phase 5).

**Cómo lo hace**:
- `JSONValue = str | int | float | bool | None` — sintaxis PEP 604 (Python 3.10+), equivale a `Union[str, int, float, bool, None]`. Describe EXACTAMENTE los escalares que JSON puede representar. GOTCHA: en Python `bool` es subclase de `int` → un validador estricto (True vs 1) debe chequear `bool` ANTES que `int`.
- `FunctionCall(BaseModel)` — `name: str` SIN default (requerido: si el generador produce algo sin name, explota → fail fast); `parameters: dict[str, JSONValue]` con `default_factory=dict`.

**Decisiones de diseño**:
1. Modelar la salida con pydantic → gratis: validación al construir + serialización `.model_dump()` / `.model_dump_json()`.
2. `default_factory=dict` y NO `parameters: dict = {}` → los defaults mutables se evalúan UNA vez (definición de clase) y se COMPARTEN entre instancias: mutar el dict de una instancia mutaría el de TODAS (el clásico mutable default bug). `default_factory` LLAMA al callable en cada construcción → dict fresco por instancia.
3. Valor de parámetro limitado a `JSONValue` (escalares, sin arrays/objetos anidados) — el scope del MVP. Habilitar objetos/arrays = bonus B8 (nota en function_definition.py).

**Gotchas**: bool ⊂ int en Python (chequear bool primero); pydantic v2 usa `model_dump()` (el viejo `.dict()` está deprecado).

---

### 5. `src/models/function_definition.py`

**Qué hace**: modela la ENTRADA del sistema — las fichas con las que pydantic valida `functions_definition.json`. Define qué funciones existen y qué parámetros acepta cada una.

**Cómo lo hace** — 4 piezas:
1. `ParameterType = Literal["string", "number", "boolean", "null"]` — conjunto CERRADO de valores exactos (menú fijo). Distinto de `str`: si el JSON trae `"integer"` en vez de `"number"`, pydantic explota. Documenta + mypy estático + pydantic en runtime.
2. `ParameterDef(BaseModel)` — ficha del parámetro: `name: str = Field(default="")` (ANDAMIO temporal, ver abajo) + `type: ParameterType` (obligatorio).
3. `FunctionDef(BaseModel)` — ficha de la función: `name`, `description`, `parameters: dict[str, ParameterDef]` (validación recursiva, error con ruta exacta), `returns: dict[str, str]` (solo entrada, no valida salida).
4. `_sync_parameter_names` — el validador que sincroniza `name` desde la key del dict padre.

**El corazón: el validador** — PROLEMA: en `{"a": {"type":"number"}}` el nombre vive en la KEY del dict, no adentro. Pero el decoder (Phase 4/5) necesita cada `ParameterDef` saber su nombre sin contexto externo → hay que copiarlo adentro.

```python
@field_validator("parameters", mode="after")
@classmethod
def _sync_parameter_names(cls, params):
    for key, param in params.items():
        param.name = key
    return params
```

Los 3 conceptos:
- `@field_validator("parameters", ...)` — pydantic lo llama SOLO al construir un FunctionDef, nunca directo.
- `mode="after"` — corre DESPUÉS de validar, recibe el dato YA tipado (objetos reales) → por eso podemos mutar `param.name`. (`mode="before"` recibiría el JSON crudo).
- `@classmethod` — OBLIGATORIO en pydantic v2: el validador corre sobre la clase (cls), antes de que exista la instancia.

Secuencia (el "andamio"): 1. Pydantic construye `ParameterDef(name="")` placeholder → 2. El validator pisa `name="a"` → 3. `ParameterDef(name="a", type="number")`. El validador corre DENTRO de la construcción → ningún consumidor ve el estado des-sincronizado (fail fast + consistencia). No hay estado intermedio visible.

**Decisiones de diseño**:
1. `name` con `default=""` (andamio) porque el JSON no trae el nombre adentro — el validador lo pisa después.
2. `Literal` como menú fijo: valida runtime y documenta.
3. Validación recursiva de pydantic: primero cada parámetro, después la función entera.

**Gotchas**: `mode="after"` recibe datos ya tipados (permite mutar); `mode="before"` recibe crudo (pre-procesar). Todo esto es del M2 de NotebookLM (pydantic y modelos).

### 6. `src/loader/input_loader.py`
*(pendiente)*

### 7. `src/loader/function_loader.py`

**Qué hace**: carga `functions_definition.json` → lista de fichas validadas (`list[FunctionDef]`). Es el loader de la ENTRADA de funciones — define el set que el decoder restringido va a poder emitir (Phase 4).

**Cómo lo hace**:
- JSON file → `json.load` → lista de dicts crudos.
- **Validación de forma ANTES de contenido**: `isinstance(data, list) and len(data) > 0` → si no, `ValueError` con el path (fail fast). Cierra la asimetría con `input_loader` (que ya validaba no-vacío) — ver BUG-002 en BITACORA_BUGS.md.
- **Una sola pasada** construye Y valida:
  - `FunctionDef(**item)`: pydantic valida cada campo contra sus anotaciones (tipos, Literal, requeridos) → `pydantic.ValidationError` (subclase de ValueError) cumple el contrato sin código extra.
  - `seen: set[str]`: consultar/insertar un nombre es O(1) (hash table). Duplicado → `ValueError` inmediato, sin construir los modelos restantes.

**Decisiones de diseño**:
1. **Opción A (rechazo duro)** para lista vacía: `ValueError` claro en vez de dejar pasar `[]` en silencio. Un array vacío = 0 funciones = fallo asegurado contra el corrector ("different function sets"). Descartada la opción B (log + seguir): para datos de entrada, el silencio no es una opción.
2. **Fail-fast integrado**: detectar el duplicado DENTRO del loop (primera reincidencia) en vez de construir todo y contar después. El mensaje es singular (el primer repeat), no la lista ordenada de todos.
3. **O(n) con `set` en vez de O(n²) con `names.count()`**: la versión anterior recorría la lista entera por cada nombre. Para ~10 funciones daba igual; es una decisión de forma para que el patrón no se copie a futuros loaders con datasets grandes (ver TeoricNotes.md → Counter/set/fail-fast).
4. **Todo se traduce a `ValueError`**: el caller (pipeline) atrapa UNA excepción. Exception chaining `from exc` para JSONDecodeError/FileNotFoundError.

**Gotchas**: el mensaje de duplicados cambió de plural a singular (fail-fast) → el test `test_duplicate_names_raise` se actualizó al nuevo match. La lista vacía tiene test propio nuevo (`test_empty_list_raises`). El plan de implementación (Task 1.8) y el didáctico documentaban el patrón O(n²) sin validación — se actualizaron en el mismo cambio para no contradecir el código.

### 8. `src/loader/vocab_loader.py`

**Estado: 🔴 EN REVISIÓN — presentada 2026-09-09, PENDIENTE de re-explicación.** El usuario terminó la sesión desorientado con unicode/decodificación (disonancias entre dos análisis). Retomar con: byte-to-unicode table, `'Ġ'` = espacio byte-encodeado, por qué `model.decode()` y NO `encode+decode` de Python (roundtrip identidad), y por qué el índice usa el 1er carácter DECODIFICADO. Material: TeoricNotes.md → "Byte-level BPE", BUG-003 en BITACORA_BUGS.md.

**Qué hace** (sin testear, sin modificar — análisis 2026-09-09):
1. Abre el `vocab.json` del modelo via `model.get_path_to_vocab_file()` → `{token_text: token_id}` (151K+ tokens).
2. Construye `id2token` invirtiendo el dict.
3. Por cada token, decodifica con el tokenizer REAL (`model.decode([token_id])`) → corrige byte-to-unicode table.
4. Indexa el token por el PRIMER carácter del texto **decodificado** (`tokens_starting_with[first_char]`) → `dict[str, set[int]]`, O(1) de membership.
5. Si decodifica a nada o falla (especial `<|endoftext|>`, textos fallidos) → bucket `BYTE_CATEGORY = "<byte>"`.
6. Guarda además `id2decoded` (vista decodificada completa) — la spec original NO la tenía.
7. Un solo pase: O(V) tokens × O(D) decodificación = pago único upfront; toda consulta posterior es O(1) en slots RAM.

**Decisiones de diseño**:
1. `@dataclass(slots=True)` → sin `__dict__` por instancia → ~menos RAM con 151K+ tokens.
2. Índice agrupado por primer char DECODIFICADO: el decoder filtra candidatos por el primer carácter del texto REAL que viene del modelo, no del token crudo.
3. `setdefault` idiom para el bucket inicial sin `if/else` ruidoso.
4. Un solo `die` de carga: si llama `decode()` con `[token_id]` (lista), NO con `token_id` suelto — la API del SDK espera lista.

**Gotchas**: los tokens del vocab JSON guardan caracteres byte-mapeados (ej: `'Ġthe'`), no el texto real. `model.decode` los convierte a texto real (`' the'`). 🔴 HALLazGO BUG-003: la spec (Task 1.9 + didáctico) enseñaba `token_text.encode('utf-8').decode('utf-8')` — roundtrip identidad que NO deshace la tabla → índice inservible. Doc actualizada al approach real (ver BITACORA_BUGS.md → BUG-003).

**Tests**: `TestLoadVocab` con `FakeModel` (tests/test_loader.py L95-125) — NO necesita el modelo real. `test_builds_preindexed_structures` verifica tamaño 5, `token2id["{"]==0`, `id2token[2]=="Ġworld"`, `id2decoded[2]==" world"`, `2 in tokens_starting_with[" "]`, `4 in tokens_starting_with["<byte>"]`.

### 9. `src/prompt/prompt_builder.py`
*(pendiente)*

---

## ⚠️ Gotchas y decisiones registradas (acumulan durante el recorrido)

- **Asimetría de validación entre loaders hermanos** (BUG-002, 2026-09-08): `input_loader` rechazaba `[]` y `function_loader` no → se unificó con Opción A (rechazo duro) + duplicados fail-fast O(n) con `set`. La spec (PLAN_IMPLEMENTACION Task 1.8) documentaba el patrón viejo; se actualizó en el mismo cambio (doc ↔ código no se contradicen).
- **Doc ↔ código divergentes en vocab_loader** (BUG-003, 2026-09-09): la spec (Task 1.9 + didáctico) enseñaba `encode('utf-8').decode('utf-8')` — roundtrip identidad que NO deshace la byte-to-unicode table → índice por 'Ġ' inservible; el código real usa `model.decode([token_id])`. Doc actualizada al approach real. Además: circuló "sin test unitario" — FALSO (`TestLoadVocab` con FakeModel, tests/test_loader.py L95-125). Parada 8 queda EN REVISIÓN hasta re-explicar unicode/decode al usuario.

---

## 🔗 Referencias vivas (no repetir acá, son el source of truth)

- **Plan de fases**: `docs/design/PLAN_IMPLEMENTACION.md` (Tasks 1.1-2.3 marcadas)
- **Cronograma ajustado**: `docs/design/CRONOGRAMA_TRABAJO.md` (bloque "Ajuste por desfase", D10-D21)
- **Apuntes teóricos**: `docs/notes/TeoricNotes.md`
- **Errores/procedimientos**: `docs/tracking/BITACORA_BUGS.md`