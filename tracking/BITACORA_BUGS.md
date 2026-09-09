# 🐛 BITÁCORA DE BUGS — call_me_maybe

> Registro cronológico de bugs encontrados, su causa raíz y resolución.
> **Regla**: cada entrada documenta QUÉ pasó, POR QUÉ pasó, y QUÉ aprendimos.

---

## BUG-001: `ParameterDef` definida pero nunca vinculada a `FunctionDef`

- **Fecha de detección**: 2026-08-23
- **Severidad**: MEDIA (no rompía el pipeline, pero dejaba una capa completa de validación sin efecto)
- **Estado**: ✅ RESUELTO
- **Archivos afectados**:
  - `src/models/function_definition.py` (corregido)
  - `tests/test_models.py` (actualizado)

### Síntoma

La clase `ParameterDef` existía en `function_definition.py`, pero `FunctionDef` no la usaba:

```python
# ANTES — ParameterDef era código muerto
class ParameterDef(BaseModel):
    name: str = Field(description="Parameter name")
    type: str = Field(description="Parameter type: 'string', 'number', 'boolean', 'null'")

class FunctionDef(BaseModel):
    parameters: dict[str, dict[str, str]] = Field(...)  # ← dict crudo, SIN validación
```

Consecuencia práctica: cualquier valor en el campo `type` de un parámetro pasaba
la validación silenciosamente:

```json
{"a": {"type": "chuchu"}}   // ✅ pasaba (NO debería)
{"a": {"type": 123}}        // ✅ pasaba (NO debería)
```

### Análisis de causa raíz

**Dato clave verificado**: el patrón incompleto está en la SPEC MISMA, no solo
en el código. `PLAN_IMPLEMENTACION.md` (Task 1.6) define exactamente ese par de
clases — `ParameterDef` por un lado y `FunctionDef.parameters:
dict[str, dict[str, str]]` por otro. El ejecutor copió la spec fielmente.

#### Atribución de autoría

| Documento | Subagente | Motor | Rol |
|-----------|-----------|-------|-----|
| `PLAN_IMPLEMENTACION.md` | lama-design | **big-pickle** | Diseño arquitectónico |
| `CRONOGRAMA_TRABAJO.md` | lama-tasks | deepseek-v4-flash-free | Planificación temporal |
| `PLAN_DIDACTICO.md` | lama-onboard | mimo-v2.5-free | Material didáctico |
| Código `src/` | Sin atribución registrada | Desconocido* | Ejecución del plan |

\* No hay git history ni metadatos que identifiquen qué motor escribió el código.
Lo único verificable es que el código coincide línea-por-línea con Task 1.6 del
plan de diseño, incluyendo el defecto.

**Nota de atribución** (fuente: criterio y memoria del operador del proyecto,
2026-08-23): el diseñador fue el subagente **lama-design** del pipeline lama,
configurado para tirar del motor big-pickle en tareas de diseño. La asignación
de motor por rol es una decisión de configuración de los agentes tomada al
armar el setup, no un dato registrable en los artefactos.

#### Precisión sobre la responsabilidad

- El error se atribuye al **pipeline agente+motor** (lama-design sobre
  big-pickle), no al modelo "a secas". El subagente produjo una spec con
  estructura muerta; eso es un fallo del output del agente en esa tarea.
- **Es un solo incidente**: NO hay evidencia suficiente para concluir que
  big-pickle tiende sistemáticamente a generar clases sin vincular en diseños.
  Lo documentado acá es UN caso, útil como señal a monitorear, no como
  veredicto.
- El hallazgo **sistémico y corregible** es otro: el flujo
  diseño → ejecución → entrega **no tenía ningún paso de verificación** que
  detectara entidades definidas pero no referenciadas (dead code estructural).
  Ese hueco de proceso es la causa raíz operativa; cualquier motor puede
  colarse por él.

**Conclusión**: la desconexión se originó en la fase de DISEÑO (subagente
lama-design / big-pickle), que definió la estructura correcta (`ParameterDef`)
pero nunca especificó su vinculación con `FunctionDef`. La implementación
replicó fielmente ese hueco porque el proceso no contemplaba detectarlo.

### Resolución

1. Se cambió `FunctionDef.parameters` a `dict[str, ParameterDef]`.
2. Se agregó `ParameterType = Literal["string", "number", "boolean", "null"]`
   para validar valores permitidos, no solo tipos.
3. `ParameterDef.name` ahora tiene default `""` y se sincroniza con la key del
   dict vía `field_validator(mode="after")` — el JSON de entrada no incluye el
   nombre dentro del objeto del parámetro (viene como key).
4. Tests nuevos: tipo inválido rechazado, tipo no-string rechazado, los 4 tipos
   documentados aceptados, sincronización de nombres.

```python
# DESPUÉS
ParameterType = Literal["string", "number", "boolean", "null"]

class ParameterDef(BaseModel):
    name: str = Field(default="", description="Parameter name")
    type: ParameterType = Field(...)

class FunctionDef(BaseModel):
    parameters: dict[str, ParameterDef] = Field(...)

    @field_validator("parameters", mode="after")
    @classmethod
    def _sync_parameter_names(cls, params): ...
```

### Verificación

- [x] `uv run pytest tests/test_models.py tests/test_loader.py -v` → 23 passed
- [x] `flake8` + `mypy` limpios
- [x] Carga end-to-end de las 5 funciones reales de `data/input/functions_definition.json`

### Lecciones aprendidas

1. **Una clase definida pero no referenciada es un smell**: si existe en el
   modelo, debe estar conectada al flujo de datos o eliminarse.
2. **El diseño también puede tener bugs**: revisar las specs antes de asumir
   que el ejecutor fue el culpable. Acá el plan contenía el defecto.
3. **Validar VALORES, no solo tipos**: `dict[str, dict[str, str]]` garantizaba
   strings, pero aceptaba cualquier string. `Literal` cierra el set de valores.
4. Cuando un modelo Pydantic recibe datos cuya identidad viene de la KEY del
   dict (y no del value), documentar explícitamente cómo se sincroniza.
5. **El pipeline necesita verificación de coherencia spec↔código**: ningún paso
   del flujo lama detectó dead code estructural. Acción preventiva sugerida:
   agregar al verify (sdd-verify / revisión post-implementación) un chequeo de
   "clases/funciones exportadas sin referencias".
6. **Atribuir con precisión**: el responsable es el pipeline agente+motor, y un
   incidente aislado no es una tendencia. Documentar como señal monitoreable,
   no como veredicto sobre el motor.

---

## Procedimiento 2026-09-02 — Config de linters (Makefile / pyproject.toml)

**Nota enunciativa**: el diseño original (lama-design) definió la config de
linters de forma mínima y no alineada al subject: el target `lint` del Makefile
solo incluía `mypy src/ --ignore-missing-imports` (1 de los 5 flags
obligatorios) y no existía `lint-strict`. Además, `ignore_missing_imports`
vivía duplicado entre `pyproject.toml` y el Makefile.

**Cambios efectuados** (mínimos, sin tocar lógica de negocio):
- `Makefile`: `lint` ahora ejecuta `flake8 .` y `mypy .` con los 5 flags
  obligatorios del subject (`--warn-return-any --warn-unused-ignores
  --ignore-missing-imports --disallow-untyped-defs --check-untyped-defs`);
  se agregó el target `lint-strict` (`mypy . --strict`); se eliminó el
  `--max-line-length=120` redundante (ya vive en `.flake8`).
- `pyproject.toml`: se agregó `exclude = "(llm_sdk/|tests/)"` en `[tool.mypy]`
  — necesario para que `mypy .` no chequeé el SDK provisto ni los tests (no
  evaluados por el subject), análogo al `extend-exclude` de `.flake8`.
- `tests/test_loader.py`: se removió un `# type: ignore[arg-type]` unused
  (mypy lo resolvía como `Any` por el override de `llm_sdk`, por lo que era
  código muerto).

**Coherencia de diseño**: flake8 no lee `pyproject.toml` (solo `.flake8` /
`setup.cfg` / `tox.ini`), mientras que mypy sí — por eso la asimetría de
config (`.flake8` + `[tool.mypy]` en `pyproject.toml`) es la correcta y no una
inconsistencia.

**Verificación**: `make lint` y `make lint-strict` pasan limpios (11 archivos,
solo `src/`).

---

## BUG-002: `function_loader` sin validación de lista vacía y duplicados O(n²)

- **Fecha de detección**: 2026-09-08 (recorrido de código, parada 7)
- **Severidad**: MEDIA (no rompía el pipeline existente, pero dejaba un hueco de validación frente al corrector y un algoritmo cuadrático innecesario)
- **Estado**: ✅ RESUELTO
- **Archivos afectados**:
  - `src/loader/function_loader.py` (corregido)
  - `tests/test_loader.py` (test de duplicados actualizado + `test_empty_list_raises` nuevo)
  - `docs/design/PLAN_IMPLEMENTACION.md` (Task 1.8 actualizada)
  - `docs/design/PLAN_DIDACTICO.md` (sección "Cargador de funciones" actualizada)

### Síntoma

`function_loader.py` tenía DOS defectos de diseño frente a su loader hermano `input_loader.py`:

1. **Asimetría de validación de forma**: `load_prompts` rechazaba listas vacías
   (`if not isinstance(data, list) or len(data) == 0`), pero `load_functions` solo
   validaba `isinstance(data, list)`. Un `functions_definition.json` con `[]` pasaba
   **silenciosamente** → el pipeline seguía con **0 funciones**. El corrector compara
   el set de funciones declaradas contra el esperado ("different function sets") →
   un array vacío era falla garantizada sin ningún error de nuestro lado.

2. **Detección de duplicados O(n²) y sin fail-fast real**:

```python
# ANTES — construye todo, recorre la lista por CADA nombre
functions = [FunctionDef(**item) for item in data]
names = [fn.name for fn in functions]
duplicates = sorted({name for name in names if names.count(name) > 1})
if duplicates:
    raise ValueError(f"Duplicate function names: {duplicates}")
```

`names.count(name)` recorre la lista COMPLETA por cada nombre → O(n²). Además, el
error se detectaba DESPUÉS de construir todos los modelos (trabajo desperdiciado) y
el mensaje listaba todos los duplicados ordenados en vez de cortar en el primero.

### Análisis de causa raíz

**El defecto está en la SPEC, no solo en el código**: `PLAN_IMPLEMENTACION.md`
Task 1.8 especificaba exactamente ese patrón (O(n²) con `count()`, sin check de
vacío), mientras que la Task 1.7 (`input_loader`, su hermana) SÍ especificaba
`Expected non-empty list`. Dos specs hermanas con criterios de validación distintos
→ el ejecutor replicó cada spec fielmente y la asimetría quedó documentada en el
propio plan. `PLAN_DIDACTICO.md` replicó lo mismo: la sección de prompts muestra la
validación de vacío y la de funciones no la muestra.

#### Atribución de autoría

| Documento | Subagente | Rol |
|-----------|-----------|-----|
| `PLAN_IMPLEMENTACION.md` Task 1.8 | lama-design | Diseñó el patrón con la asimetría |
| `PLAN_DIDACTICO.md` sección cargadores | lama-onboard | Replicó la asimetría del plan |

**Conclusión**: defecto de DISEÑO (spec asimétrica entre Tasks 1.7 y 1.8), no de
ejecución. El recorrido de código (parada 7) cumplió su rol: al comparar capas
hermanas, detectó la incoherencia.

### Resolución

Decisión del operador (2026-09-08): **Opción A — rechazo duro + validación
integrada fail-fast**, una sola pasada, O(n):

```python
# DESPUÉS — valida forma + construye + detecta duplicados en UNA pasada
if not isinstance(data, list) or len(data) == 0:
    raise ValueError(f"Expected a non-empty JSON array of function definitions in {path}")

seen: set[str] = set()
functions: list[FunctionDef] = []
for item in data:
    fn = FunctionDef(**item)
    if fn.name in seen:
        raise ValueError(f"Duplicate function name: {fn.name}")
    seen.add(fn.name)
    functions.append(fn)
return functions
```

Cambios concretos:
1. **Lista vacía → `ValueError` claro** con el path (misma semántica que `input_loader`).
2. **Duplicados dentro del loop**: `set` (hash table) → consultar/insertar es O(1), total O(n).
3. **Fail-fast real**: corta en la PRIMERA reincidencia, sin construir los modelos restantes.
4. **Mensaje singular** (primer duplicado encontrado) en vez de la lista ordenada de todos.
5. El contrato no cambia: todo sigue siendo `ValueError` para el caller (`pipeline`/`__main__`).

### Verificación

- [x] `uv run pytest tests/ -v` → **37 passed** (36 previos + `test_empty_list_raises` nuevo)
- [x] Test de duplicados actualizado al nuevo mensaje (singular)
- [x] `flake8 .` limpio
- [x] `mypy .` con los 5 flags del subject → limpio
- [x] E2E: carga real de las 5 funciones + rechazo real de `[]` con el mensaje correcto

### Lecciones aprendidas

1. **Loaders hermanos deben validar lo mismo**: si `input_loader` rechaza vacío y
   `function_loader` no, el pipeline acepta JSON sin sentido (0 funciones) y el
   corrector lo cuenta como set inválido. La validación de forma (no vacío) es
   CONTRATO del loader, no lujo.
2. **`list.count()` dentro de un loop es O(n²)**: recorre la lista entera por cada
   nombre. Para decenas de funciones es irrelevante; para miles, cuadrático. El
   `set` da O(1) por operación → O(n) total. (→ TeoricNotes.md, sección Counter.)
3. **Fail-fast real = cortar durante la construcción**: detectar el duplicado
   MIENTRAS construís evita trabajo desperdiciado; detectarlo al final construye
   todo para nada.
4. **Las specs hermanas se copian entre sí**: cuando una Task define un patrón y su
   hermana otro distinto, el código resultante replica la inconsistencia. Al
   detectar un defecto, revisar la spec hermana — el bug suele estar documentado.
5. **Actualizar la doc que muestra el código viejo**: `PLAN_IMPLEMENTACION.md` y
   `PLAN_DIDACTICO.md` mostraban el patrón obsoleto; se actualizaron en el mismo
   cambio para no dejar doc que contradiga la decisión (pedido explícito del operador).

---

## BUG-003: `vocab_loader` — la spec enseñaba un approach que NO decodifica (doc ↔ código divergentes)

- **Fecha de detección**: 2026-09-09 (recorrido de código, parada 8 — análisis cruzado de dos sesiones)
- **Severidad**: MEDIA (no rompía runtime: el código real es CORRECTO; la DOC era la que enseñaba un approach que no funciona → riesgo de copia futura y de modelado mental erróneo)
- **Estado**: ✅ RESUELTO (doc actualizada)
- **Archivos afectados**:
  - `docs/design/PLAN_IMPLEMENTACION.md` (Task 1.9 reescrita con el approach real)
  - `docs/design/PLAN_DIDACTICO.md` (sección "Cargador de vocabulario": código + explicación corregidas)
  - `docs/notes/TeoricNotes.md` (sección nueva "Byte-level BPE y roundtrip identidad")
  - `docs/notes/RECORRIDO_EJECUCION.md` (parada 8 documentada, EN REVISIÓN)
  - `docs/tracking/PROGRESS_TRACKER.md` (nota de sesión 09/09)
- **Síntoma**: la spec (Task 1.9 línea ~1121 y PLAN_DIDACTICO línea ~725) proponía decodificar con `token_text.encode('utf-8').decode('utf-8')`; el código real usa `model.decode([token_id])`. Dos análisis de la misma parada llegaron a conclusiones que parecían contradictorias.
- **Análisis causa raíz**:
  1. El vocab.json guarda tokens maquillados por la **byte-to-unicode table** (cada byte 0-255 → carácter "visible"; el espacio 0x20 se guarda como `Ġ`). El texto real solo se obtiene con la tabla inversa, que aplica el TOKENIZER.
  2. `token_text.encode('utf-8').decode('utf-8')` en Python es un **roundtrip identidad** para strings unicode válidos: `'Ġthe'` sale `'Ġthe'` → el índice quedaría agrupado por caracteres-mapeados → inservible para el decoder que busca por texto real (`' the'` → `tokens_starting_with[" "]`).
  3. Las excepciones que atrapaba la spec (`UnicodeDecodeError`, `IndexError`) eran casi código muerto: `encode('utf-8')` no falla para strings unicode válidos.
  4. La explicación del didáctico ("bytes UTF-8 incompletos") confundía el modelo mental: el vocab ya viene en texto mapeado, no con bytes sueltos.
  5. El ejecutor implementó el approach correcto sin actualizar la doc → divergencia doc ↔ código (espejo inverso de BUG-002: allá el código era PEOR que la spec; acá el código es MEJOR).
- **Desmentido de afirmación errónea**: circuló que "vocab_loader no tiene test unitario / necesita el modelo real (~2.4GB)". **FALSO**: `TestLoadVocab` con `FakeModel` existe en `tests/test_loader.py` (L95-125) y pasa (`test_builds_preindexed_structures`, 5 tokens simulados). Verificar claims de sesiones contra el código antes de repetirlos.
- **Resolución**: doc actualizada al approach real — `model.decode([token_id])`, `id2decoded`, `BYTE_CATEGORY = "<byte>"`, `@dataclass(slots=True)`, índice por primer carácter DECODIFICADO.
- **Verificación**: solo docs modificados (código intacto) → 37 tests siguen en verde, flake8 + mypy limpios.

### Lecciones aprendidas

1. **Doc ↔ código divergen en dos direcciones**: BUG-002 (código peor que spec, faltaba validación) y BUG-003 (código MEJOR que spec, la doc enseñaba decodificación falsa). Al analizar una implementación, SIEMPRE contrastarla con su spec — el análisis "por dentro" del código no alcanza.
2. **La decodificación de BPE se hace con el TOKENIZER, no con Python**: `model.decode` aplica la tabla inversa byte-to-unicode; `str.encode().decode()` es un roundtrip identidad que no decodifica nada.
3. **El `Ġ` no es una letra rara, es un espacio disfrazado**: el vocab muestra tokens maquillados; `'Ġthe'` decodificado es `' the'`.
4. **Verificar afirmaciones de sesiones de IA contra el código**: "no tiene test unitario" era verificablemente falso en 30 segundos (`tests/test_loader.py` L95-125). Un claim categórico y falso merece corrección explícita en la doc.

---

<!-- Próximos bugs se agregan acá abajo con el mismo formato -->
