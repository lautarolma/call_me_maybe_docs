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

## BUG-004: abstención de `allows_token` ante estados límite — gaps de validación en las 4 cláusulas

- **Fecha de detección**: 2026-09-17 (análisis cláusula por cláusula tras el diagnóstico del operador)
- **Severidad**: ALTA (conceptual: el schema NO valida lo que promete en casos de tokens multi-fase; no rompía la suite porque los tests nunca ejercitaban esos tokens)
- **Estado**: ✅ RESUELTO (pase fino post-argmax — Inciso 4.1.1, Task 4.1)
- **Archivos afectados**:
  - `src/decoder/constrained_generator.py` (NUEVO — implementa el pase fino)
  - `src/decoder/schema_validator.py` (flag `_params_object_seen` + docstring de la residual 2)
  - `tests/test_constrained_generator.py` (NUEVO — 12 tests, 9 del pase fino)
  - `docs/design/PLAN_IMPLEMENTACION.md` (Inciso 4.1.1)
- **Síntoma**: las 4 cláusulas de `allows_token` juzgan estados LÍMITE (commiteado pre-token vs. simulado post-token) y NO ven estados intermedios dentro del token. Cuando un token cruza varias fases (entra a parameters + abre la primera key, o completa key+value+cierre), las cláusulas **abstienen** (`return True` "sin constraint") y el texto pasa de contrabando, quedando aceptado de por vida. Casos concretos:
  1. **Gap 2 residual 2** (descubierto en este análisis): `', "parameters": {"a'` — el token arranca en depth 0 → cláusula 2 (`self._depth != 1` → abstiene) y la key jamás se valida ni en tokens posteriores (el trigger por cambio no vuelve a disparar: `current_key` siguió siendo "a"). Una key INEXISTENTE (`"zz"`) entraba al schema de por vida.
  2. **Gap 3**: `', "b": "x",'` (key + value string + cierre para `"b": number`) — el token ni termina en fase de value ni arranca en COLON → la cláusula 3 esquiva ambos triggers y deja pasar un type WRONG.
  3. **Gap 4**: entrar Y salir de parameters en un token — la cláusula 4 solo ve depths commiteado/simulado; el 0→1→0 intermedio (y el ⊆ de required contra keys_enclosed de ESE punto) le pasa desapercibido.
  4. **Slip del duplicado exacto**: `'"a": 4'` con "a" ya emitida — la máquina resetea `current_key` a "" y la reconstruye idéntica → cláusula 2 no detecta cambio (limitación residual 1 documentada) → la key duplicada pasa.
  5. **Gap del plan** (documentado, no bug): COMPLETE sin haber pasado por PARAMS_OBJECT — `'{"name":"fn_empty"}'` se completaba sin el objeto parameters.
- **Análisis causa raíz**: `allows_token` es PURA por diseño (no muta el schema, lee snapshot commiteado + estado simulado). Ese contrato, correcto para el filter de Task 3.4, solo ve DOS puntos del recorrido por token. Cualquier semantic que dependa de un punto intermedio (cambio de key, cierre de objeto, entrada a parameters) queda fuera del alcance de una validación token-wise y solo puede resolverse re-simulando el token char por char. La abstención (`return True` en las líneas 235/302/336/338/380/382/385/405 del schema_validator) NO es un bug de lógica sino un límite DE INFORMACIÓN del approach por-token.
- **Resolución**: pase fino post-argmax en el generator (Inciso 4.1.1): tras elegir el mejor token en `_pick_best_token`, se re-simula char por char sobre una copia del estado con un `SchemaContext` FRESCO. En cada carácter: avanzar la máquina → `allows_token(char, trial)` con el schema aún sincronizado al estado PRE-char → recién después `update`. Ese orden reproduce el contrato de Fase 3 y expone los estados intermedios a las mismas 4 cláusulas: el reset de `current_key` se ve como cambio (cierra gaps 2/4/slip), el COLON intermedio expone el tipo (gap 3), el depth 1→0 dispara required (gap 4), y el flag `_params_object_seen` (sembrado desde el schema real, porque el historial previo no se puede re-derivar del token actual) exige parameters en COMPLETE (gap del plan). Costo: 1 re-simulación por step, solo del GANADOR.
- **Verificación**: 12 tests nuevos (9 de pase fino + 3 end-to-end con FakeModel) → 163 tests suite green, flake8 + mypy limpios.

### Lecciones aprendidas

1. **`return True` en una cláusula NO es "validar", es ABSTENERSE**: cada abstención es un hueco de validación, no una decisión. Al diseñar cláusulas por estados límite, auditar qué token multi-fase puede esquivar CADA trigger — el diagnóstico del operador ("las cláusulas juzgan la frontera, no el interior") era exacto.
2. **El contrato pre/post de `allows_token` no se puede "forzar" para ver estados intermedios sin violar la pureza**: la solución correcta es una segunda pasada char-por-char con el MISMO verbo (`allows_token`) y el MISMO orden (allows antes de update). El pase fino reusa la lógica existente en vez de duplicarla.
3. **El orden update→allows es un bug silencioso**: la primera implementación del pase fino actualizaba el schema ANTES de llamar `allows_token` → `self.* == new_state.*` siempre → las cláusulas de cambio/depth jamás gatillaban y TODOS los tests de bloqueo fallaban. El contrato es exactamente el inverso: allows (pre) → update (post).
4. **Los tokens de test deben partir de estados ALCANZABLES**: el caso gap 4 original arrancaba con `"` desde VALUE_END (sintaxis inválida: `_step_value_end` solo acepta ws/,/}) — el test fallaba por la razón equivocada. El token real equivalente arranca con la coma: `', "parameters": {...}'` (ver `expected_first_chars`).
5. **El `_params_object_seen` no se puede re-derivar**: si el token actual no contiene el `{` de parameters, el pase fino no puede saber si se abrió antes; el flag del schema REAL (historial acumulado) debe sembrarse en el schema fino.

---

## BUG-005: bug de dimensiones en el flujo de generación — `encode().tolist()` asumía tensor 1D y el SDK devuelve 2D

- **Fecha de detección**: 2026-09-18 (hipótesis del operador + verificación independiente con torch real)
- **Severidad**: ALTA (bloqueaba la generación con el modelo REAL: crash al primer step; invisible en CI porque el mock replicaba un tensor 1D)
- **Estado**: ✅ RESUELTO
- **Archivos afectados**:
  - `src/decoder/constrained_generator.py` (fix: `[0].tolist()` + `prompt_length` pre-loop)
  - `tests/test_constrained_generator.py` (mock _FakeTensor 2D + assert de contrato + test N-tokens)
  - `docs/design/PLAN_DIDACTICO.md` (L1720/L1756-57 reescritos)
  - `docs/design/PLAN_IMPLEMENTACION.md` (spec Task 4.1 L1374/L1396 + nota BUG-005)
- **Síntoma**: `model.encode(prompt).tolist()` con el SDK real produce `list[list[int]]` (`[[id1,...,idN]]`) porque `Small_LLM_Model.encode()` arma `torch.tensor([ids])` → tensor **2D [1, N]** (docstring del SDK L78: "return a 2-D input_ids tensor"). Ese resultado se pasaba a `get_logits_from_input_ids()` que espera `list[int]` plano y hace `torch.tensor([input_ids])` → tensor **3D [1,1,N]** → `out.logits[0, -1]` devuelve `[N, V]` → cada elemento es una FILA → `[float(x) for x in logits]` lanza `TypeError: float() argument must be a string or a real number, not 'list'`. Además: `prompt_length = len(model.encode(prompt).tolist())` contaba **filas** (=1), no tokens → el slice final arrastraba/omitía tokens del prompt; y `input_ids.append(best_id)` mezclaba un int en una lista de listas (`torch.tensor` → `TypeError: not a sequence`).
- **Análisis causa raíz**:
  1. Supuesto falso compartido: código Y docs (didáctico L1718 visual: `input_ids = [5765, 318, ...]` plano; plan L1374) asumían que `tolist()` aplana un tensor 1D. El SDK real es 2D.
  2. El único testigo de la forma real era el docstring del propio SDK (L78) — nunca se testearon las FORMAS del contrato, solo el comportamiento de alto nivel del mock.
  3. El mock `_FakeTensor.tolist()` devolvía `list[int]` plano → replicaba un tensor 1D inexistente → la suite aprobaba el bug (163 tests en verde con código roto).
  4. mypy no lo detectó: `torch.Tensor` sin stubs → `.tolist()` es `Any`; flake8 no chequea tipos.
  - Verificación ejecutada (sin tocar archivos): `/tmp/opencode/dim_bug_demo.py` con torch real replicando el código exacto del SDK — tolist 2D `[[...]]`, tensor 3D `[1,1,N]`, `logits[0,-1]` → `[N,V]` → TypeError confirmado; `len([[..]]) == 1` vs `4` tokens; `append` → `torch.tensor` TypeError.
- **Resolución**: `input_ids = model.encode(prompt)[0].tolist()` (aplana la única fila del tensor 2D → `list[int]`) + `prompt_length = len(input_ids)` guardado ANTES del loop (elimina el re-encode del prompt al final y usa la longitud REAL de tokens).
- **Verificación**: mock `_FakeTensor` ahora replica la forma 2D (`.tolist()` → `list[list[int]]`, `t[0].tolist()` → plano) + assert de contrato de formas dentro del `FakeModel.get_logits_from_input_ids` (si `generate()` dejara de aplanar, la suite entera tiñe de rojo) + test nuevo `test_n_token_prompt_does_not_leak_into_generated` (prompt de 3 ids inexistentes en VOCAB → si alguno llegara a generated, KeyError ruidoso). Suite: **164 tests en verde**, flake8 + mypy limpios.

### Lecciones aprendidas

1. **El mock debe replicar las FORMAS del contrato, no solo el comportamiento**: `_FakeTensor` devolvía lista plana porque "funcionaba" — pero el SDK es 2D. Un mock que no refleja la forma real del objeto que reemplaza puede aprobar bugs de dimensionalidad por años. Regla: testear el contrato de formas (shape/estructura), no solo valores.
2. **Tercer caso de divergencia doc ↔ código ↔ realidad**: BUG-002 (código peor que spec), BUG-003 (código mejor que doc), BUG-005 (código Y docs compartían un supuesto falso sobre el SDK). La doc enseña el mismo error — por eso la corrección se documentó en el pseudocódigo y en la spec, no solo en el código.
3. **`len()` sobre un `.tolist()` de tensor 2D cuenta FILAS**: `len([[a,b,c]]) == 1`. Cualquier métrica de longitud sobre tensores tiene que aplanar primero o indexar la fila.
4. **Guardar `prompt_length` antes del loop mata dos pájaros**: elimina el re-encode del prompt (costo 1 encode extra por generación) y ancla la longitud a los tokens reales, no a un slice posterior de la lista mutada.
5. **Torch sin stubs es un agujero de tipos para mypy**: el contrato dimensional hay que testearlo explícitamente (el assert de `isinstance(x, int)` en el fake lo hace), no confiar en el type checker.

---

<!-- Próximos bugs se agregan acá abajo con el mismo formato -->
