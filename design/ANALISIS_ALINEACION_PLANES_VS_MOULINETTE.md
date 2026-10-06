# ANÁLISIS — Alineación PLAN_IMPLEMENTACION (v1) / PLAN_EJECUCION_V2 vs subject + moulinette real

> **Fecha**: 2026-09-27 (sesión de análisis, sin tocar código).
> **Pedido del usuario**: individualizar los objetivos de ambos planes de ejecución,
> confirmar si están alineados con la planilla de evaluación y los tests de la
> moulinette, y evaluar la arquitectura planteada para lo pendiente desde la
> **Task 4.3 (inclusive)** del plan v1.
> **Método**: lectura directa de fuentes primarias — `docs/sources/en.subject.pdf`
> (vía `pdftotext -layout`), el código real de `moulinette/moulinette/moulinette/`
> (el evaluador, no un mock), y el código actual de `src/`. No se tocó código ni
> se commiteó nada — esto es un documento de análisis puro.

---

## 1. Objetivos individualizados

### Plan v1 — `docs/design/PLAN_IMPLEMENTACION.md` (18/08, ordenado por fase de módulo)

| Fase | Objetivo | Estado real (verificado 27/09) |
|---|---|---|
| 1. Foundation | loaders, CLI, modelos Pydantic, estructura del repo | ✅ completa |
| 2. Prompt Engineering | `prompt_builder.py` | ✅ completa |
| 3. Decoder Core | state machine, trie, schema validator, token filter (testeable sin modelo) | ✅ completa + superada (oráculo N1, Opt2 no estaban en el plan original) |
| 4. Generation Loop | `constrained_generator.py`; **Task 4.3**: correr los 11 prompts, medir timing + accuracy | ✅ corre; **Task 4.3 en rojo**: 82% full accuracy (bar 90%), timing al filo en la VM local |
| 5. Validation + Pipeline | 5.1 `output_validator.py`, 5.2 pipeline persiste output, 5.3 verificar formato | 🟡 **recién arrancada** (`e415a6c`, 26/09) — ver §3 |
| 6. Error Handling + Polish | manejo de errores completo, README, tests comprehensivos, lint, **6.5 auditoría de docstrings** | ⬜ no empezada |
| 7. Bonus (anexo) | B1-B9, no bloquea MVP | ⬜ |

### Plan v2 — `docs/design/PLAN_EJECUCION_V2.md` (26/09, ordenado por bloqueante — reemplaza al v1 desde Phase 5)

| Etapa | Objetivo | Depende de / Bloquea |
|---|---|---|
| 0 | Confirmar forwards en hardware de cluster | informativa |
| 1 ⛔ | **Capa de salida**: validator + persistencia + nombre de archivo — bloquea TODO lo demás (score 0 sin esto) | nada → bloquea 2-5 |
| 2 | Accuracy M14 ≥90% (forense P9/P10) | Etapa 1 |
| 3 | Robustez y error handling | Etapa 1 |
| 4 | Documentación (README, docstrings 6.5.1) | Etapas 1-3 |
| 5 | Quality gate (coverage, lint-strict, checklist A2 final) | todo lo anterior |

**Veredicto sobre la reordenación**: el v2 es correcto en su premisa. El v1 dejaba
Phase 4 como último hito ejecutado y asumía que "falta construir el decoder" —
cuando en realidad el decoder está completo/optimizado (oráculo N1, Opt2) y lo
que faltaba al 25/09 era el output. Es una reordenación válida de objetivos ya
existentes, no un cambio de objetivos.

---

## 2. Alineación con el subject + moulinette real — verificado punto por punto

Fuentes primarias usadas: `docs/sources/en.subject.pdf` (extraído con
`pdftotext -layout` a `/tmp/subject.txt`, no persistido) y
`moulinette/moulinette/moulinette/{__main__,functions_definition,generate_tests_and_corrections}.py`
(el evaluador real, presente en disco pero gitignored).

### ✅ Confirmado — nombre del archivo de salida

Subject V.4, literal (línea 582 del PDF extraído): *"Your program will produce
a single JSON file: data/output/function_calling_results.json"*. V.6 (testing,
línea 662) lo repite. La línea 350 (`function_calls.json`) es solo un ejemplo
de invocación de CLI, no la especificación del formato de salida.

**Conclusión**: la decisión tomada en `e415a6c` / `src/cli.py:40`
(`DEFAULT_OUTPUT = Path("data/output/function_calling_results.json")`) está
bien fundada **con o sin** la "planilla oficial" citada en el mensaje de ese
commit — esa planilla no la encontré como archivo en el repo (busqué con
`grep -rli "planilla"` sobre todo `docs/` y no hay match), así que no pude
verificarla directamente. No hace falta: el subject solo ya zanja el punto de
forma inequívoca.

### ✅ Confirmado — campo `prompt` con igualdad exacta

`moulinette/__main__.py:133` compara
`student_answer.get("prompt") != correction["prompt"]` **antes** de mirar
`name`/`parameters` — si no matchea, ese test es 0 sin más chequeo, sin
importar si la función/parámetros están bien.

Verifiqué además que `data/input/function_calling_tests.json` (11 prompts)
está en el **mismo orden** que el dict `exercises` de
`moulinette/functions_definition.py` (fn_add_numbers×2, fn_greet×2,
fn_reverse_string×2, fn_get_square_root×2,
fn_substitute_string_with_regex×3) — necesario porque el grader empareja con
`zip(student_answers, corrections)`, que es **posicional**.

**Conclusión**: agregar `prompt` (verbatim, sin las function definitions
inyectadas por `build_prompt`) era efectivamente el bloqueante más crítico de
los tres que cerró `e415a6c`. Confirmado correcto.

### ✅ Confirmado — nunca omitir entries (alineamiento posicional)

`moulinette` usa `zip()`: un hueco en medio de las entries desalinea TODO lo
que viene después contra las corrections. El diseño de
`build_results()` (`src/validator/output_validator.py:152`) — placeholder
`__unparseable__` en la posición exacta cuando un prompt no parsea, nunca
omitir la entry — es correcto para este contrato.

### 🔴 Hallazgo importante — el script de accuracy local NO mide lo mismo que moulinette

`moulinette` **no compara strings de parámetros**: ejecuta la función real y
compara el `return` value
(`__main__.py:161`, `student_output != correction["expected_output"]`, donde
`student_output = fn(**fn_params)` sobre la función real de
`functions_definition.py`).

En cambio `~/scratch/call_me_maybe_task42/task43_accuracy.py` (el script que
produjo el "82% full accuracy, P9/P10 fallan" que figuraba en el tracking de
esa fecha) hace comparación de VALOR/TIPO estricta contra
una lista fija de variantes aceptadas por parámetro:

```python
# task43_accuracy.py:95-99
def _match_value(got: object, expected: object) -> bool:
    return type(got) is type(expected) and got == expected
```

Consecuencia concreta, caso por caso:

- **P10** (vowels → asteriscos): la lista de regex aceptados en el script es
  `["[aeiouAEIOU]", "[aeiou]"]` (línea 80). Si el modelo emite
  `regex="([aeiouAEIOU])"` (con grupo de captura), el script lo marca
  ERROR porque no está en la lista. Pero con `re.sub(pattern, replacement, s)`
  y un `replacement` literal (`'*'`, sin backreferences `\1`), **un grupo de
  captura en el patrón no cambia el resultado de la sustitución** — moulinette,
  que ejecuta la función real, muy probablemente NO marcaría esto como error.
  El propio `PLAN_EJECUCION_V2.md` (§Etapa 2, nota previa) ya sospechaba algo
  así al decir que P10 "falla por DOS razones, no una" y que el `replacement`
  (`'****'` vs `'*'`) es la causa real no-relajable — pero el tracking vigente
  de esa fecha todavía atribuía el fallo, en parte,
  al grupo capturador como si fuera un fallo real ante moulinette.
- **P9** (dígitos → NUMBERS): `replacement="NUMBER"` (singular) vs esperado
  `"NUMBERS"` (plural) SÍ cambia el string resultante de `re.sub` → ahí el
  script local y moulinette deberían coincidir en marcarlo mal.
- Nota adicional sobre `_match_value`: compara `type(got) is type(expected)`,
  o sea que un `2` (int) vs `2.0` (float) para `fn_add_numbers`/
  `fn_get_square_root` cuenta como error en el script local aunque
  `2 == 2.0` y la aritmética real de Python (lo que moulinette de hecho
  ejecuta) no distinga entre ambos. Otra fuente posible de subestimación.

**Implicación práctica para la Etapa 2 del plan v2**: antes de decidir
"relajar vs exigir" el scoring (pendiente de esa fecha en el plan maestro),
conviene correr el output real del pipeline contra el evaluador de verdad:

```bash
uv run python -m moulinette grade_student_answers <path-al-output-real> --set public
```

(el código ya está en disco en `moulinette/`, gitignored, corre local) en vez
de — o además de — `task43_accuracy.py`. El 82% medido con el script scratch
podría estar **subestimando** el accuracy real que vería un evaluador.

### ⚠️ Riesgo no resuelto — "All classes must use pydantic for validation" (subject IV.3.1, línea 297 del PDF, texto literal)

Catálogo completo de clases sustantivas en `src/` (vía
`grep -rn "^class \|^@dataclass" src/`):

| Pydantic (`BaseModel`) | NO pydantic |
|---|---|
| `FunctionCall` (`models/output.py`), `FunctionDef`, `ParameterDef` (`models/function_definition.py`) | `DecoderState` (`decoder/state.py`), `Vocab` (`loader/vocab_loader.py`), `TrieNode` (`decoder/trie.py`), `PhaseMetrics`, `MetricsRun` (`utils/metrics.py`) — todas `@dataclass(slots=True)` · **`SchemaContext`** (`decoder/schema_validator.py`) — clase plana, ni siquiera `@dataclass` |

El plan v1 (Decisión 1, `PLAN_IMPLEMENTACION.md` §A12) decidió deliberadamente
sacar el decoder de Pydantic, con razón técnica documentada (costo de
instanciación Pydantic en el inner loop de generación, ~200μs/instancia ×
tokens × prompts). Es defendible en ingeniería, pero es una **contradicción
literal** con un "Additional Requirement" explícito del subject — no una
ambigüedad interpretable como la del nombre de archivo.

**Estado**: ninguno de los dos planes (v1 ni v2) lo trata como riesgo de
compliance — el v2 ni lo menciona en su Etapa 5 (quality gate / checklist A2
final). Queda **abierto, sin decisión tomada** — no propuse solución porque es
una decisión de arquitectura que corresponde marcar al usuario primero.

---

## 3. Arquitectura de lo pendiente desde Task 4.3 (inclusive) — evaluación

### Task 4.3 (all-prompts timing + accuracy)

Corrida y documentada, con la salvedad de medición de accuracy del punto
anterior (script local vs moulinette real). El timing depende del hardware —
ya está bien diagnosticado en `docs/notes/HARDWARE_VM.md` (VM con oversubscription de
vCPU, causa raíz identificada 27/09).

### Phase 5 / Etapa 1 (capa de salida) — parcialmente implementada en `e415a6c`

- **`src/pipeline.py::run` + `write_results`**: correcto — crea
  `data/output/` con `mkdir(parents=True, exist_ok=True)`, serializa con
  `model_dump()`, preserva el orden 1:1 con `raw_prompts` (no con el prompt ya
  inyectado con las function definitions), que es justo la distinción que
  exige moulinette para el match exacto de `prompt`.
- **`src/validator/output_validator.py::build_results`**: la pieza que
  realmente conecta con el output final del pipeline. Diseño sólido
  (placeholder posicional, nunca rompe el `zip()` de moulinette).
- **`src/validator/output_validator.py::validate_output`**: **incompleta**
  respecto a lo que pide el subject V.4.2 ("Keys y tipos deben matchear el
  schema exactamente", "no extra keys", "todos los required presentes",
  "tipos deben matchear"). Hoy solo chequea que `name` exista en `functions`
  — no valida tipos de `parameters`, ni keys faltantes/extra contra el schema
  de la función elegida. El propio docstring del módulo lo admite
  (`output_validator.py:126-131`: "queda fuera de este primer corte"). Es
  defendible como defense-in-depth de bajo riesgo (el decoder ya bloquea casi
  todo esto por construcción), pero es la Task 5.1/1.2 del plan **sin
  terminar** — y además `validate_output()` hoy **no lo invoca nadie**
  (confirmado también en el mensaje del propio commit `e415a6c`).
- **Sin tests propios**: no existe `tests/test_output_validator.py` ni ningún
  test que ejercite `parse_output` / `build_function_call` / `validate_output`
  / `build_results` directamente (verificado sobre los 10 archivos de
  `tests/`). Es la Task 1.4 de la Etapa 1 del plan v2, todavía pendiente.
- **Desalineado con el plan maestro de esa fecha**: la tabla de tareas
  (incluso tras la actualización del 27/09 leída en esta sesión) sigue
  marcando la fila 5.1/5.2 con ⬜, pese a que `e415a6c` ya las adelantó
  bastante. Vale la pena actualizarla para no perder el hilo entre sesiones.

### Phase 6 / Etapas 3-5 (error handling, README, tests, docstrings, quality gate)

Sin empezar en ambos planes — coinciden en esto, no hay desalineación que
señalar.

---

## 4. Acciones sugeridas (NO ejecutadas — pendientes de decisión del usuario)

1. Correr el output real del pipeline contra
   `uv run python -m moulinette grade_student_answers` para tener el accuracy
   real de M14, antes de decidir si P9/P10 se relajan o se atacan de raíz.
2. Decidir sobre el riesgo IV.3.1 (Pydantic en todas las clases): aceptar el
   trade-off documentado (dataclasses en el inner loop) y dejarlo anotado
   como decisión consciente, o migrar `SchemaContext` (y evaluar el resto) a
   Pydantic.
3. Completar `validate_output()` contra el schema completo (tipos, required,
   extra keys) y conectarlo al pipeline, más sus tests — cierre real de la
   Task 5.1 / Etapa 1.1-1.4 del plan v2.
4. Actualizar la tabla de tareas del plan maestro (filas 3-4) para reflejar el
   avance real de `e415a6c`.
