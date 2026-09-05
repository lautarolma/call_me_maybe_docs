# 🧭 Recorrido de ejecución — call_me_maybe (repaso Phase 1 + Phase 2)

> Clase/revisión file-by-file siguiendo el flujo REAL del programa.
> Método: **por demanda** — el usuario guía el ritmo, hace preguntas y ordena avanzar.
> Inicio: 2026-09-04 (Día 10, tras completar Phase 2)

---

## 📍 Posición actual del recorrido

- **Archivo en análisis**: `src/models/output.py` (parada en curso)
- **Progreso**:
  - [x] 1. `src/__main__.py`
  - [x] 2. `src/cli.py` (dudas resueltas: argparse -h/help, exit codes 0/1/2, Path vs str → ver TeoricNotes.md)
  - [x] 3. `src/pipeline.py`
  - [ ] 4. `src/models/output.py`
  - [ ] 5. `src/models/function_definition.py`
  - [ ] 6. `src/loader/input_loader.py`
  - [ ] 7. `src/loader/function_loader.py`
  - [ ] 8. `src/loader/vocab_loader.py`
  - [ ] 9. `src/prompt/prompt_builder.py` ← Phase 2 (hoy)
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
*(pendiente)*

### 6. `src/loader/input_loader.py`
*(pendiente)*

### 7. `src/loader/function_loader.py`
*(pendiente)*

### 8. `src/loader/vocab_loader.py`
*(pendiente)*

### 9. `src/prompt/prompt_builder.py`
*(pendiente)*

---

## ⚠️ Gotchas y decisiones registradas (acumulan durante el recorrido)

- *(vacío por ahora — se llena en cada parada)*

---

## 🔗 Referencias vivas (no repetir acá, son el source of truth)

- **Plan de fases**: `docs/design/PLAN_IMPLEMENTACION.md` (Tasks 1.1-2.3 marcadas)
- **Cronograma ajustado**: `docs/design/CRONOGRAMA_TRABAJO.md` (bloque "Ajuste por desfase", D10-D21)
- **Apuntes teóricos**: `docs/notes/TeoricNotes.md`
- **Errores/procedimientos**: `docs/tracking/BITACORA_BUGS.md`