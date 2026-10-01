# ESTADO_ACTUAL — call_me_maybe (42 School)

> Archivo de estado DINÁMICO — importado con `@` desde `CLAUDE.md`. Se actualiza al inicio/fin de cada sesión. Formato COMPACTO a propósito: **este archivo es para arrancar sesión y decidir, no para historia.** Detalle fino → `docs/design/`, bugs → `docs/tracking/BITACORA_BUGS.md`, y las series históricas de medición están en el **git history** (no se repiten acá).

**Última actualización**: 2026-09-27 noche — los 2 KPIs medidos: **latencia <5' CUMPLIDA (4'52")** · **accuracy real 6/11 (54,5%)**, con el fix ya verificado que la sube a **10/11 (90,9%)**.

---

## 🔴 1. ESTADO DEL DÍA — leer esto primero

| Criterio del subject | Bar | Medido | Estado |
|---|---|---|---|
| Suite de 11 prompts | < 5 min | **4'52,72"** | ✅ CUMPLE (4,7% de margen) |
| Accuracy full (M14) | ≥ 90% | **6/11 = 54,5%** | 🔴 FALLA · con el fix **10/11** ✅ |
| Accuracy fn (nombre) | — | 11/11 (100%) | ✅ |
| Entregable | archivo existe | ✅ 11 entries en `data/output/function_calling_results.json` | ✅ |
| lint (`make lint`) | == moulinette | flake8 0 · mypy 23 files | ✅ |
| tests | verde | **206 passed** | ✅ |

**El único bloqueante de la bar M14 es la coerción `int`→`float`** (§4). Está verificada end-to-end, son ~5 líneas, y **falta tu OK para tocar código**.

### 1.1 Latencia — la carrera de hardware se CERRÓ

Corrida **limpia** (agente cerrado, cargador conectado, 4 vCPU 1:1 + `cpuexecutioncap 80`):

| | 25/09 (6 vCPU) | 27/09 12'49 (**contaminada**) | **27/09 noche (LIMPIA)** |
|---|---|---|---|
| wall | 389,1 s | 769,3 s | **292,72 s (4'52,72")** |
| CPU / cores efectivos | — / 4,0 | 257% / 2,57 | **336% / 3,364** |
| CPU-s por forward | ~9,9 | 14,87 | **7,40** |
| RSS máx | — | 4,65 GiB | **4,65 GiB** |

- **La regresión de 2,63x de las 12'49 fue 100% el agente vivo**: 1,31x (más cores) × 2,01x (el doble de eficiencia por núcleo) = 2,63x ✓. **Ninguna medición de latencia sirve si el agente está vivo** — comparar siempre con `opencode` cerrado.
- 🔑 **El código corre al 99,2% del techo de hardware de la VM** (3,364 de los 3,39 cores medidos con curva de saturación, `steal=0.00`). **No queda margen en el guest: cualquier mejora futura de latencia tiene que ser CÓDIGO, no config.**
- 🔑 **El output es BIT-IDENTICAL al de las 12'49 bajo 257% vs 336% de CPU** → **el decode es reproducible, no es lotería de threading**. Cierra la pregunta abierta del 25/09 y descarta a BUG-012 y al float16 como causas de los fallos.
- ⚠ **Único hueco de medición abierto: el 2x de CPU-s/forward NO tiene causa.** Con MENOS cores ocupados la eficiencia por núcleo fue MEJOR → no es contención. `steal=0.00`. Hipótesis: SMT del host o power/thermal. No bloquea la entrega.

### 1.2 Accuracy — el 82% era fiction

**Medido con el corretero real: 6/11 (54,5%)**. Los 5 fallos:

| Test | Síntoma | Causa raíz | ¿Arreglable? |
|---|---|---|---|
| P0, P1 (`2+3`, `265+345`) | `invalid parameters` | emite `2` (int) donde va `2.0` (float) | ✅ **un solo bug, 6 valores** |
| P6, P7 (sqrt 16, 144) | `invalid parameters` | ídem | ✅ ídem |
| P9 (vowels→asterisks) | `wrong output` | `replacement: "****"` en vez de `"*"` | ❌ techo del modelo |

- 🔑 **La causa NO es el decoder: es el schema.** `extract_functions_infos.TYPE_MAP` mapea `int → "integer"` y `float → "number"`, así que `"number"` en el JSON proviene **exclusivamente** de un float de Python, y la moulinette asserta `isinstance(a, float)`. **La coerción keyeada en el tipo declarado es la traducción fiel del schema, no un parche.**
- ⚠ **Un blanket "todo número → float" ROMPE el set privado**: `fn_is_even(n: int)` asserta `isinstance(n, int)`. **`fn_calculate_compound_interest` tiene `principal`/`rate` = `number` y `years` = `integer` en la MISMA función** → el tipo declarado es la única clave que desambigua.
- **`task43_accuracy.py` (scratch) NO es oráculo de accuracy: sobreestimaba 5,5 puntos.** Sólo `grade_real.py` sirve.
- **`P9` ya no falla** por `NUMBER`/`NUMBERS` (trae `NUMBERS` correcto) y **`P10` nunca falló** por el grupo capturador (con replacement literal, `re.sub("([aeiou])","*",s)` da lo mismo que sin paréntesis). Las dos notas viejas al respecto eran **FALSAS**.

### 1.3 P9 — diagnóstico con medición, no intuición

Un forward sobre el prefijo exacto que la state machine ya había inyectado:

```
'****'  logit 19,130   ← lo que emitimos
'*'     logit  5,359   ← lo esperado     → delta 13,77, rank 6.302
```

El **top-15 global entero** es de asteriscos e inglés (`aster` 18,24 · `stars` 16,04): el modelo **nunca decidió emitir un carácter**, sigue en la palabra inglesa "asterisks".

- 🔑 **REENCUADRE: el modelo es un COPIADOR LITERAL, no un abstractor.** Los **6 tests cuyo argumento está VERBATIM en el prompt pasan los 6** (shrek, john, hello, world, NUMBERS, dog). **El único string que NO está literal en el prompt es P9, y es el único fallo de string.** P9 no es un bug de asteriscos: es el techo del modelo.
- **Ningún filtrado en el decoder lo arregla** (el candidato está, 13,77 logits abajo). Sólo el prompt lo mueve, y `build_function_list` (`src/prompt/prompt_builder.py:61`) hoy imprime sólo `name (type)`, sin semántica de parámetro.
- 🔴 **Tunear el prompt hasta que P9 pase es ENVENENADO**: es un test público y el peer review lo ve. **Recomendación: cobrar el 10/11 y documentarlo como limitación conocida con esta medición.**

---

## 2. HEAD · tests · working tree

- **HEAD**: `e415a6c` — **`main` está 1 commit AHEAD de `origin/main`, SIN PUSH**.
- 🔴 **Working tree SIN commitear, listo para commit con OK — fix del SET PRIVADO (4 archivos, +211/-6)**:

  | Archivo | Qué hace | Qué pasa si NO está |
  |---|---|---|
  | `src/models/function_definition.py` | `ParameterType` acepta `"integer"` | 🔴 **El programa NO ARRANCA con el set privado**: `ValidationError: literal_error, input_value='integer'`. **0 en la mitad de la evaluación.** (verificado) |
  | `src.decoder/schema_validator.py` | `_declared_type_accepts()` — compatibilidad en vez de igualdad | 🟡 Cada prompt con un parámetro declarado `integer` da **output truncado** (`allowed` vacío → `break` en `constrained_generator.py:442`) → `build_results` emite el placeholder `__unparseable__` → **ese test en 0, los otros 10 bien**. (leído del código, no medido) |

  **Son dos cambios encadenados, no dos bugs.** Y son **complementarios de la coerción float/int, no alternativos**: el `"integer"` es para que el decoder NO rechace el valor; la coerción es para que el TIPO de Python que salga sea el que la moulinette asserta. `fn_calculate_compound_interest` necesita los dos en la misma función.
  **La regla que implementa el 2º cambia exactamente 1 celda de 20** de la matriz kind×declared: `(number, integer)` False→True. Todo lo demás idéntico (verificado).

- **Suite: 206 tests GREEN** · **flake8 0** · **mypy 23 archivos** ✅
- Commits clave: `d6592d0` Opt2 (header estático) · `794a470` BUG-011 · `0c59092` oráculo Nivel 1 (BUG-013) · `e415a6c` entregable + 3 bloqueantes de peer review.
- **Stash**: `stash@{0}` refactor-metrics descartado · `stash@{1}` anexo-reverted. **NO tocar.**
- `data/output/` es **git-ignored** (destino de métricas y del entregable, no se versiona).

---

## 3. ARQUITECTURA DE LATENCIA — lo que hay que saber para optimizarla

- 🔑 **El tiempo escala lineal con TOKENS DE SALIDA, no con el candidate set.** El SDK **no tiene KV-cache**: cada forward re-alimenta la secuencia completa. Confirmado: P1 (34,3 s) vs P0 (12,4 s) con el mismo schema (solo `265`/`345` son multi-token); P9 y P10 cuestan 40,3 y 42,2 s con schema casi idéntico. **Reducir candidatos NO abarata el forward**; sólo evita forwards si logra singleton.
- **Forwards de strings ≈ tokens BPE + comilla final, EXACTO** — no hay margen en strings libres.
- **133 forwards** para los 11 prompts. Fases: `IN_STRING_VALUE` **82% del tiempo**, `IN_NUMBER_VALUE` 13 fwd, el resto de estructura **0 fwd** (todo por oráculo).
- **Per-prompt (s)**: 20,1 · 34,3 · 12,4 · 10,1 · 10,4 · 10,4 · 18,5 · 21,9 · **58,5** · 40,3 · 42,2. P8 = 21% del total (el más largo, y PASA).
- **Overhead de arranque = 13,64 s (4,7% del wall)**: pesos ya cacheados (0,58 s) + índice de vocab de 151.643 + trie. Pre-índice y header estático lo dejaron en el plagó. **Ahí no hay más que ganar.**
- **Techo de la VM = 3,39 de 4 cores (85%)**, `steal=0`. Con `cpuexecutioncap 80` el techo real es 3,2. **El ambiente de fondo se come 1,29 cores (32%) con la VM "idle"** (`opencode`, `gnome-shell`, `tracker-miner`).
- **Pillow de CLUSTER (mismo código, mismo build `+cpu`): 0,56 s/fwd, 133 forwards, 1'14".** Local quedó en **2,10 s/fwd → 3,75x de brecha, y ya NO es config: es hardware de host** (i7-7700HQ de notebook vs nodo de cluster).
- **Vocab (151.643 tokens)**: 96,87% son `string_safe` → bucketizar `IN_STRING_VALUE` NO sirve. El BPE fragmenta dígitos y puntuación (peor 2,17 chars/token).
- **Palancas que quedan, ninguna barata**: B′ (autocompletar `fn_name` por trie) y Nivel 2 del oráculo. Diferidas por decisión del usuario. El KPI ya se cumple → esto es hambre, no supervivencia.

**Hardware de la VM (leer antes de medir)**: host i7-7700HQ = **4 núcleos físicos / 8 hilos** · VM Ubuntu 22.04 con **4 vCPU 1:1** + cap 80 · `threads=4` (default = nproc, no hay `set_num_threads` en el código) · **SIN GPU** (`VMware SVGA II` emulada, torch `2.13.0+cpu`, `cuda.is_available()=False`) · RAM: host 24 GB − VM 15 GiB (**NO era 64G**, anotación vieja falsa) · ⚠ **`--cpuexecutioncap` se aplica con `controlvm`, NO con `modifyvm`** (que exige VM apagada) y es **PER vCPU**.

**El decoder es device-agnostic**: `llm_sdk` resuelve device (mps > cuda > cpu) y dtype solo, y `get_logits_from_input_ids` devuelve `list[float]` → **`src/decoder/` es CPU puro**. En un equipo con GPU corre sin tocar `src/`: `pip install torch accelerate` y listo. El formato de salida es invariante al device (header y tail se **inyectan** vía `encode()` sin `forward()`; la estructura la imponen `compute_allowed_ids` + `SchemaContext`). **Test de humo pendiente**: guardar el golden de CPU y diffear en GPU — si da idéntico, el tema queda cerrado para siempre. ⚠ **Costo latente en GPU**: `[float(x) for x in out.logits[0,-1].tolist()]` es un no-op semántico que cuesta **26,4 ms/step** (1% en CPU, **17–53% en GPU**) más el `sorted()` de `token_filter`.

---

## 4. 🔀 VÍAS DE SOLUCIÓN — decidir (bloqueante de la entrega)

### Bloqueante 1 · coerción `int`→`float` (6/11 → 10/11 = 90,9%, cumple M14)

Regla común a las tres: **keyeada en el tipo declarado del schema, NUNCA blanket**. Si `param.type == "number"` y el valor parseado es `int` → `float(valor)`. Si es `"integer"` → se deja el `int`.

| Vía | Dónde | Tamaño | Riesgo | Veredicto |
|---|---|---|---|---|
| **A** ⭐ | `src/validator/output_validator.py:152` `build_results()` — frontera de serialización, después de `parse_output` | ~5 líneas | **cero**: `16.0 == 16` exacto, determinista, no puede cambiar el valor | **Recomendada** — testeable sin modelo, legible para el revisor |
| **B** ❌ | En el decoder, bloqueando el cierre del número hasta `dígito . dígito` | media | **ALTO**: obliga al modelo a elegir el dígito post-punto sin razón → puede emitir `2.5` donde debía emitir `2.0`, **cambiando el valor semántico** | **Descartada** |
| **C** ⭐ | `src/decoder/constrained_generator.py` — splice de `model.encode(str(float(v)))` en los `input_ids` al completar el número | hook en el hot loop + re-simulación con `copy(state)` | cero en el valor, peor legibilidad | Alternativa válida si se prioriza consistencia |

- 🔑 **C es INSERCIAL, no destructivo** (verificado): los ids del float son **superconjunto** de los del int — `265`→`[17,21,20]` vs `265.0`→`[17,21,20,13,15]`. Se insertan `[13,15]` (`".0"`). **R2: 0 de 151.643 tokens contienen dígito+coma → el splice es atómicamente seguro.**
- **C no gana accuracy** (mismo texto, mismo 10/11). Gana **consistencia**: los `input_ids` que condicionan al modelo son los que se emiten — importa porque no hay KV-cache. **A gana legibilidad. El trade-off es ese, nada más.**
- **Tests para A o C**: (i) `"number"`+int→float · (ii) `"number"`+float→float idempotente · (iii) `"integer"`+int→int **intacto (el caso del set privado)** · (iv) `string`/`boolean`/`null` intactos · (v) E2E con `grade_real.py` → 10/11.

### Bloqueante 2 · P9 (el único fallo restante — NO relaxing, NO hardcodear)

| Vía | Veredicto |
|---|---|
| Filtrar candidatos en el decoder | ❌ **imposible**: `*` está presente, 13,77 logits abajo |
| Prompt enrichment genérico (descripción por parámetro + ejemplo) | ⚠ legítimo pero **sin garantía** — no puede inventar "asterisks" → `*` |
| Mentionar asteriscos en el prompt | 🔴 **overfitting a un test público que el peer review ve** — NO |
| **Documentarlo como limitación conocida con la medición** | ⭐ **Recomendado** |

**Si A → 10/11 = 90,9% → M14 CUMPLIDA y P9 deja de bloquear la entrega.**

---

## 5. ✅ Pendientes — PARA RETOMAR (27/09 noche)

1. 🔴 **Implementar la coerción** (A ~5 líneas o C) + los 5 tests + correr `grade_real.py` para confirmar 10/11. **Único bloqueante de M14. Requiere OK del usuario para tocar código.**
2. 🔴 **Commitear el fix del set privado** (§2) — ya verificado en verde. **Sin el 1º de los dos el programa NO ARRANCA con el set privado.**
3. 🔴 **`git push`** — `main` 1 commit adelante (`e415a6c`).
4. 🔴 **Decidir compliance IV.3.1** ("All classes must use pydantic"): **6 de 9 clases de `src/` no son Pydantic** (5 dataclasses + `SchemaContext` plana). Sigue ABIERTO y **ningún plan lo trata como riesgo**. Leer `docs/design/ANALISIS_ALINEACION_PLANES_VS_MOULINETTE.md` ENTERO antes de decidir Etapa 2 o 5 del plan v2.
5. 🟡 **Smoke test del SET PRIVADO end-to-end** — **nunca se midió accuracy contra el set privado, que es la mitad de la evaluación.** Ya hay `private_smoke_input.json` / `private_smoke_output.json` en `/tmp/cmm/`.
6. 🟡 **Registrar el conteo de forwards en el output** — es la unidad de medida de toda §3 y hoy no existe.
7. 🟡 **Investigar el 2x de CPU-s/forward** (SMT del host vs power/thermal). No bloquea.
8. 🟡 **Tasks 5.1–5.3 + DoD5 + 6.1–6.5**: `src/validator/` existe pero **no valida que los `parameters` encajen con el schema de la función elegida** (TODO explícito en `validate_output`, L126-131); falta 5.3 (formato de salida) y las de métricas/reports no-LLM.
9. 🟡 **B′ (autocompletar `fn_name` por trie)** y **Nivel 2 del oráculo** — la única vía de latencia que queda.
10. 🟡 **Test de humo en un equipo con GPU** (golden de CPU diffeado).
11. ⚪ **Backups de probes en `/tmp/` (se pierden al reiniciar)**: `grade_real.py` (**el corretero real sin `fire` — recrear primero**), `probe_ceiling.py` (curva de saturación), `probe_p9.py` (los logits), `probe_integer_fix.py` (la matriz 20 celdas). Los logs que hay que preservar están en `/tmp/cmm/`: `suite_run.txt`, `grade_after.txt`, `answer_6of11_original.json` (6/11), **`answer_float.json` (el 10/11 verificado)**, `functions_definition_private.json`.
12. ⚪ **Scratch fuera del repo** (correr siempre con `cwd = repo`): `~/scratch/call_me_maybe_task42/` — `task43_accuracy.py` (⚠ **ya no es oráculo de accuracy**), `bench_p8_vs_p2.py`, `vocab_study.py`. Borrar `~/scratch/PENDING_DELETE__call_me_maybe_backup_mario_validation/` cuando se descarte.
13. ⚪ **Migrar el `moulinette/` del repo fuera de la entrega** (es dependencia de la cátedra, no código nuestro) y **bump del puntero de `docs/`** (submodule privado, commit aparte — `docs/` NO va en la submission).
14. ⏳ **Higiene de la entrega — prosa y referencias colgantes** (pedido del usuario 28/09). **SECUENCIADO: va DESPUÉS de cerrar M14 y la medición de latencia.** Objetivo: que un revisor encuentre la lógica en menos de 2 líneas de lectura.
    ⚠️ **Las MÉTRICAS NO SE BORRAN — son el bonus B7** (ver §5.14bis). Este pendiente es sólo *poda de prosa*, nunca *eliminación de instrumentación*.

    **Medición del problema (28/09)**: `src/` tiene 3474 líneas; **1691 son prosa** (1109 de docstrings + 582 de comentarios = **48,7%**). Código real: 1333.

    | Qué | Veredicto | Evidencia |
    |---|---|---|
    | **Referencias a `docs/` desde `src/`** | 🔴 **defecto duro** | `function_loader.py:66` → `BITACORA_BUGS.md` · `constrained_generator.py:63` → `CONTEXTO_REFACTOR.md`. `docs/` es un repo **privado** (`github.com:lautarolma/call_me_maybe_docs`) y **no va en la submission**: punteros que el revisor no puede abrir. Hay que **inlinear el porqué** en el docstring |
    | **Fechas de diario** en comentarios | 🟡 ruido puro | `(2026-09-18)`, `(2026-09-23/24)`, `dec. 25/09`, `probe 25/09` — ~10 ocurrencias. Al revisor le importa el BUG, no cuándo se encontró |
    | **Refs al plan externo** | 🟡 misma clase colgante | `PLAN_DIDACTICO L1615/L1696`, `Inciso 4.1.1`, `Anexo §2.2` (4+2+2 ocurrencias). El revisor no tiene esos documentos |
    | **Deliberación auto-referencial** | 🟡 | Frases que narran la conversación en vez del diseño ("POR QUÉ ES UN HELPER Y NO UN if EN EL LOOP", "DESVÍO del plan (ver docstring del módulo)"). Recortar a la decisión + el porqué |
    | **Traducir la prosa a inglés** | 🔴 **NO** | Los comentarios en español son el estilo de la casa y la pedagogía del 42. Son 1691 líneas: reescribirlas cuesta muchísimo y **no suma un punto de la planilla**. El problema es el *volumen* y el *qué dice*, no el idioma. Ojo: el **README sí va en inglés** (exigido por el subject, cap. VII L676) |

    **Orden de ataque sugerido**: (1) los 2 punteros colgantes — son 2 líneas y son los únicos que rompen algo; (2) fechas + refs al plan, que es `sed`-eable; (3) la poda de volumen, archivo por archivo, empezando por `schema_validator.py` (57,7%) y `constrained_generator.py` (43,7%).

14bis. 🎁 **BONUS B7 — Visualización de la generación, CON estilo y color** (corregido por el usuario 28/09: *"son parte del bonus de representación, las vamos a usar para enseñar los resultados de la ejecución"*). **NO eliminar la instrumentación — es lo que se demuestra.**

    **Verificado contra el enunciado oficial** (`docs/sources/en.subject.pdf`, cap. VII *Bonus Part*):
    - L689: *"Visualization of the generation process"* — bonus oficial.
    - L693: *"Bonus features must be implemented and **working** — not just described in the README.md."*
    - 🔑 L694: *"You may be asked to **demonstrate them during evaluation**."* → no es decoración: **la evaluación puede pedir una demo en vivo.** Por eso el color tiene que ser el vehículo de la explicación, no adorno.
    - Diseño ya definido en `PLAN_DIDACTICO.md` M13 B7: flag `--visualize` step-by-step, **verde = allowed · rojo = blocked · amarillo = estado actual**.

    | Pieza | Qué hay | Qué falta |
    |---|---|---|
    | `MetricsRun` (`utils/metrics.py`) | forwards, skips, elapsed por `DecoderPhase` | — |
    | `report_prompt_metrics()` (28/09) | tabla por prompt + `decode_metrics.json` | estilo/color |
    | Doble camino de reporte | `report()`/`write_json()` (sólo los usan sus tests, y `report()` fija `warm_up_discarded: True` **falso en el pipeline**) | **unificar** en UN camino, con estilo — no borrar: son la fuente de datos de B7 |
    | `--visualize` step-by-step | ❌ no existe | el verde/rojo/amarillo de B7 |

    ⚠️ **Regla de ingeniería, NO hardcodear ANSI**: gatear con `sys.stdout.isatty()` y respetar `NO_COLOR` (https://no-color.org). Motivo concreto: el output de la corrida se **pega en el chat y se guarda en logs** — los códigos de escape ensucian la evidencia y rompen el diff de métricas. El color sólo cuando hay terminal de verdad; la corrida normal queda limpia.


---

## 6. Protocolo de actualización

- **INICIO**: verificar el real (`git status -sb`, `git log --oneline -3`, `make test`, `make lint`) y actualizar si difiere. **Nunca tocar código con rojo.**
- **CIERRE**: reflejar HEAD, tests, mediciones. **Estado desactualizado = peor que ninguno.**
- **Regla de higiene**: este archivo se **recorta**. Si un dato es histórico y ya no cambió una decisión, va al **git history** o a `docs/design/`, no acá. Series de medición viejas, benchmarks superados y notas de diseño ya digeridas **no se reproducen** — se acotan a la conclusión vigente.
- **Nunca medir latencia con el agente vivo** (§1.1) — contamina por 2,6x.
