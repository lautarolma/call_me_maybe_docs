# ESTADO_ACTUAL — call_me_maybe (42 School)

> Archivo de estado DINÁMICO — importado con `@` desde `CLAUDE.md`. Se actualiza al inicio/fin de cada sesión. Formato COMPACTO a propósito: **este archivo es para arrancar sesión y decidir, no para historia.** Detalle fino → `docs/design/`, bugs → `docs/tracking/BITACORA_BUGS.md`, y las series históricas de medición están en el **git history** (no se repiten acá).

**Última actualización**: 2026-10-04 — **§7 nueva: teoría de las reglas post-hoc (A verificada y commiteada · B y C medidas, NO implementadas)**. **Pendiente de leer y de corroborar en suite.** §1–§6 son el snapshot del 27/09 (HEAD `e415a6c`); el estado medido hoy está en **§7.0**.
**Snapshot 27/09**: los 2 KPIs medidos — **latencia <5' CUMPLIDA (4'52")** · **accuracy real 6/11 (54,5%)**, con el fix ya verificado que la sube a **10/11 (90,9%)**.

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
- 🔴 **Tunear el prompt hasta que P9 pase es ENVENENADO**: es un test público y el peer review lo ve.

#### ✅ CORRECCIÓN 2026-10-04 — la recomendación de arriba quedó ANULADA

Esta subsection decía *"cobrar el 10/11 y documentarlo como limitación conocida"*. **Era una conclusión sobre una medición mal hecha.** El error: se comparó el output **crudo** del 28-sep (cuando `src/validator/` no existía) contra el output **ya reparado** del smoke de hoy — y se leyó esa diferencia como "el decoder mejoró". La única diferencia entre los dos archivos era que **hoy existe la regla C**. Nada del decoder cambió.

**Verificado en crudo** (llamando a `generate()` directo, sin pasar por el validador): el decoder **sigue emitiendo** `'replacement': '****'`. El techo del modelo es real y sigue donde estaba.

**Lo que cambia es la conclusión**: no hace falta cobrar el 10/11. La regla C (`_collapse_repeated_run`) repara P9 **post-hoc** sin tocar un solo logit → **11/11 público y 11/11 privado**. Ver §7 y §8.

⚠️ **Lección de método, para que no se repita**: nunca aplicar una regla post-hoc sobre un archivo de salida, porque el pipeline escribe el output **ya reparado** → doble aplicación → falso "0 reparaciones". Para ver la salida cruda hay que llamar `generate()` directo o bypasear el validador.

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

> **✅ RESUELTO 2026-10-04** — por la vía que la tabla de abajo no contemplaba: **reparación post-hoc** (regla C). M14 no queda en 10/11 sino en **11/11**. El diagnóstico de este bloque sigue siendo correcto (el decoder no va a emitir `*` por prompting: está 13,77 logits abajo); lo que estaba mal era la conclusión "cobrar el 10/11".

| Vía | Veredicto |
|---|---|
| Filtrar candidatos en el decoder | ❌ **imposible**: `*` está presente, 13,77 logits abajo |
| Prompt enrichment genérico (descripción por parámetro + ejemplo) | ⚠ legítimo pero **sin garantía** — no puede inventar "asterisks" → `*` |
| Mentionar asteriscos en el prompt | 🔴 **overfitting a un test público que el peer review ve** — NO |
| ~~Documentarlo como limitación conocida con la medición~~ | ❌ **anulada** — era conclusión de una medición mal hecha (§1.3) |
| **Reparación post-hoc `_collapse_repeated_run`** | ⭐ **la que funciona** — 11/11, sin tocar logits ni prompt |

**11/11 = 100% en ambos sets. P9 ya no bloquea la entrega.**

---

## 5. ✅ Pendientes — PARA RETOMAR (27/09 noche)

1. ✅ **CERRADO 2026-10-04.** La coerción se implementó como las **3 reglas post-hoc** (A `_snap_to_query_span`, B `_restore_internal_quotes`, C `_collapse_repeated_run`) en `src/validator/output_validator.py`. Resultado real: **11/11 público y 11/11 privado**, no 10/11. Ver §7 y §8.
2. 🔴 **Commitear el fix del set privado** (§2) — ya verificado en verde. **Sin el 1º de los dos el programa NO ARRANCA con el set privado.**
3. 🔴 **`git push`** — `main` 1 commit adelante (`e415a6c`).
4. 🔴 **Decidir compliance IV.3.1** ("All classes must use pydantic"): **6 de 9 clases de `src/` no son Pydantic** (5 dataclasses + `SchemaContext` plana). Sigue ABIERTO y **ningún plan lo trata como riesgo**. Leer `docs/design/ANALISIS_ALINEACION_PLANES_VS_MOULINETTE.md` ENTERO antes de decidir Etapa 2 o 5 del plan v2.
5. ✅ **CERRADO 2026-10-04.** Smoke end-to-end del set privado **y** del público, con generación real: **11/11 y 11/11** (`~/scratch/call_me_maybe_task42/private/run_private_smoke.sh both`). La mitad privada de la evaluación dejó de ser terra ignota.
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


15. 🟡 **Comparar la salida CRUDA a 1 vs 4 hilos** (~40 min). Los 281 márgenes de §8 son **todos de 1 hilo** (`OMP_NUM_THREADS=1`); el smoke que dio 11/11 corrió con **4 hilos** porque el script no pinea la variable. El orden de reducción en float depende del número de hilos → **nunca se comparó crudo contra crudo**. Lo único que hay es evidencia *débil*: el smoke de 4 hilos también dio 11/11. Para cerrarlo: la misma corrida con `generate()` directo, sin validador, en ambas configuraciones, y diff de los 22 outputs.

---

## 6. Protocolo de actualización

- **INICIO**: verificar el real (`git status -sb`, `git log --oneline -3`, `make test`, `make lint`) y actualizar si difiere. **Nunca tocar código con rojo.**
- **CIERRE**: reflejar HEAD, tests, mediciones. **Estado desactualizado = peor que ninguno.**
- **Regla de higiene**: este archivo se **recorta**. Si un dato es histórico y ya no cambió una decisión, va al **git history** o a `docs/design/`, no acá. Series de medición viejas, benchmarks superados y notas de diseño ya digeridas **no se reproducen** — se acotan a la conclusión vigente.
- **Nunca medir latencia con el agente vivo** (§1.1) — contamina por 2,6x.

---

## 7. 🧪 LAS 3 REGLAS POST-HOC · implementadas y corroboradas

> **Estado de esta sección**: escrita el 2026-10-04, **corroborada el mismo día**. La regla **A** ya estaba commiteada; **B y C quedaron implementadas con OK del usuario** (`3087fc5`). Todo lo de §7.1–§7.8 está **corroborado sobre una generación real completa**: el smoke end-to-end de público y privado dio **11/11 y 11/11**. La corroboración que §7.9 pedía está hecha — el detalle está en §7.9.

### 7.0 Estado medido (2026-10-04, post-corrobación)

- **HEAD `3087fc5`** — `main` **2 commits adelante de `origin/main`, SIN PUSH** (`bd07ad1` + `3087fc5`).
- **267 tests GREEN** (era 247) · flake8 0 · mypy 23 archivos.
- **Smoke end-to-end, generación real: 11/11 público (100%) y 11/11 privado (100%)**. Wall clock 1808 s / 1423 s, inflado ×2,6 por agente vivo (§1.1).
- 🔑 **Las reglas NO son facultativas en la práctica**: desactivando *sólo* la entrada rota verificada — una por set — el score cae a **10/11 (90,9%)** en ambos.
- 🔑 **CÓMO GRADEA LA CÁTEDRA — verificado** (`moulinette/__main__.py:161`): `if student_output != correction["expected_output"]` → **compara el OUTPUT de llamar a la función, NO los argumentos**. Corolario: los args solo importan **funcionalmente**. Esto es lo que abre P9 (ver §7.4).

### 7.1 El principio — lo nuevo de esta sección

- 🔑 **Una regla automática es legítima solo si se puede escribir SIN haber visto la respuesta.** Es el criterio para separar "arreglar" de "hacer trampa".
- 🔑 **Anclar a una FIRMA ESTRUCTURAL es legítimo; anclar a un VOCABULARIO es hardcodeo.** `corrida de ≥2 chars idénticos` (agnóstico: sirve para `****`, `#####`, `-----`) = legítimo. `asterisks → *` (exige diccionario) = tabla de búsqueda = **prohibido por el subject**.
- **Tres mecanismos distintos, no los mismo:**
  1. **Máscara de esquema** (durante la escritura) — de `functions_definition.json`: JSON válido, nombres de parámetro exactos, tipos. Ya existe.
  2. **Máscara derivada de la query** (durante la escritura) — "el valor no puede desviarse del texto". **DESCARTADA, medida** (§7.5).
  3. **Reparación post-hoc** (después de escribir) — **A, B y C viven acá**. No se toca un solo logit.
- 🔑 **Reparar exige que la verdad esté en la entrada.** Si el valor correcto no aparece en la frase del usuario, no hay a qué reparar. **P9 se creyó "imposible" por esto — y el error fue mío**: solo miré **una** fuente de información (la query). La regla C usa **dos** (query + convención del dominio), y esa segunda es la que la legitima.

### 7.2 Las tres reglas

| | Regla | Distorsión que deshace | Mecanismo | Estado |
|---|---|---|---|---|
| **A** | Si el valor string aparece verbatim en la query y el char izquierdo es **puntuación** → estirar hasta el borde | copia **truncada** (`home/user/...` ← `/home/user/...`) | post-hoc, `_snap_to_query_span` | ✅ **`bd07ad1`** · +7 tests |
| **B** | Si el valor NO es verbatim, pero aparece al quitar las comillas dobles de la query → **restaurar** las comillas **internas** (nunca las que abrazan todo) | copia **sin comillas** (`Say hello…` ← `Say "hello"…`) | post-hoc, `_restore_internal_quotes` | 🟡 **medida, NO implementada** |
| **C** | Si el valor es **enteramente** una corrida de ≥2 chars idénticos y **no** aparece literal en la query → el modelo **contó** en vez de **parametrizar** → deshacer el conteo (§7.6) | copia **contada** (`****` ← `*`) | post-hoc, colapsar la corrida | 🟡 **medida, NO implementada** |

### 7.3 Las mediciones que sostienen todo (probes en scratch, sin modelo)

| Regla | Dispara sobre | Falsos positivos |
|---|---|---|
| **A** | 22 outputs: **0** en público, **1** en privado (test 8) | 0 |
| **B** | 22 outputs: **1** (test 11), **0** en público | 0 |
| **C** | **38** valores string: **1** | 0 |

### 7.4 C — el hallazgo, y por qué el regex NO hacía falta

Propuesta del usuario. Regla: si el `replacement` es una corrida de chars idénticos y la frase **no la muestra**, el modelo no está describiendo el dato sino **contando las coincidencias** — y en cualquier API de sustitución (`re.sub`, `sed`, `replace` de JS) el `replacement` es una **plantilla aplicada a todas las coincidencias**, no una copia por coincidencia.

🔑 **El hallazgo que abre P9**: con el `replacement` colapsado, el output **coincide**:

```
nuestro  regex ([aeiouAEIOU]) + replacement *  ->  'Pr*gr*mm*ng *s f*n'
esperado regex [aeiouAEIOU]  + replacement *  ->  'Pr*gr*mm*ng *s f*n'
```

El grupo de captura **no cambia *qué* se matchea**, y como el grader compara el **output** (§7.0), **el regex no hace falta tocarlo**. Una sola pieza mueve el test.

**Resultado end-to-end con el código de la cátedra:**

| Suite | Antes | Con C |
|---|---|---|
| **Pública** | 10/11 (90,9%) | **11/11 (100%)** |
| **Privada** | 9/11 (81,8%) | **10/11 (90,9%)** |

⚠ **Con C desaparece el riesgo del "margel de 1 test"** del público (estaba al borde: 10/11 pasa, 9/11 reprueba).

### 7.5 DESCARTADAS POR MEDICIÓN — no reintentar

| Descartada | Por qué murió |
|---|---|
| **Máscara decoder-level "unique-anchored"** (y su variante "refinada") | Insegura en outputs **correctos**: `SELECT`→ forzaría `SQL query 'SELECT…'` · `/home`→`/home/user/data.json with utf-8 encoding` · `utf`→`utf-8 encoding`. **Raíz: el decoder no sabe dónde termina el value** — el span de la query incluye el texto de cola, así que "forzar hacia el span" siempre sobre-extiende. La "refinada" también muere porque **`/` y `-` son puntuación** y disparan falsos positivos DENTRO del value. |
| **C-bloque** (unidad >1 char, `ababab`→`ab`) | **Rota**: sobre P9 devuelve `**` en vez de `*` → el público bajaría a 10/11. |
| **Quitar el grupo de captura** si el `replacement` no tiene backreference | Da exactamente `[aeiouAEIOU]` (la respuesta de la cátedra) pero **ganancia medible = CERO** (los 3 tests ya pasan por output). Mismo costo, riesgo extra. |

### 7.6 C tiene un agujero — por eso va "refinada"

La versión cruda (colapsar siempre a 1) **falla medida**:

| query | valor | cruda | **refinada** |
|---|---|---|---|
| `…with ***` | `*****` | `*` ❌ | **`***`** ✅ |
| `Set padding to ===` | `=====` | `=` ❌ | **`===`** ✅ |
| `…with asterisks` (P9) | `*****` | `*` ✅ | `*` ✅ |
| `…with **` | `**` (verbatim) | `**` ✅ | `**` ✅ |

**La versión refinada**: si la corrida no está en la query, **buscar en la query la corrida más larga que SÍ aparece** y usar esa; si la query no muestra ninguna, usar **un** carácter. Misma norma de "respetar lo que la frase dice", aplicada a la cantidad.

⚠ **Riesgo residual**: si la frase contiene `**` por casualidad (markdown) y el modelo cuenta `****`, colapsaría a `**`. Probabilidad baja; se documenta, no se previene.

### 7.7 B es conservadora — demo de ambigüedad (medida)

| query | valor | B | por qué |
|---|---|---|---|
| `Format template: Say "hello" to {name}` | `Say hello to {name}` | → `Say "hello" to {name}` | el caso real |
| `Format template: Say "hi" to {user}` | `Say hi to {user}` | → `Say "hi" to {user}` | el espejo |
| `Replace all numbers in "Hello 34 I'm 233…" with NUMBERS` | `Hello 34 I'm 233…` | **silencio** | las comillas **abrazan** todo: son delimitadores, no contenido |
| `Say "hello" and hello` | `hello` | **silencio** | ya es literal |
| `Echo "abc" and abc to stdout` | `abc` | **silencio** | dos spans coinciden |
| `Format: "x" and "x"` | `x` | **silencio** | solo spans delimitados |
| `Format template: Use "a" or "b" for {x}` | `a` | **silencio** | ambigüedad real |

🔑 **Ante duda o más de una opción, B se queda callada.** Eso es el comportamiento buscado.

### 7.8 La teoría unificadora

> **El modelo es un COPIADOR, y mete distorsiones mientras copia. Cada regla deshace una distorsión concreta.**

Trunco (**A**) · dropeo de comillas internas (**B**) · conteo de repeticiones (**C**). Las tres son post-hoc, narrow, con firma estructural y 0 falsos positivos sobre lo medido. No son tres trola sueltas: son **una familia con una sola norma** — *el valor tiene que estar respaldado por la frase del usuario; si no, es una invención nuestra y se normaliza*.

### 7.9 ✅ RESUELTO — la corroboración se hizo

1. ✅ **B y C implementadas** en `src/validator/output_validator.py` (misma familia que A), encadenadas con A en `_repair_string_value`. Commit `3087fc5`.
2. ✅ **Smoke end-to-end corrido** (público **y** privado, generación real, `run_private_smoke.sh both`): **11/11 y 11/11**. La corroboración que esta subsection pedía está hecha.
3. ✅ **`PENDIENTES_ENTREGA.md:151` corregido** — P9 ya no figuraba como "límite del modelo, el KPI se cumple con 10/11". Vive en el submódulo `docs` → commit aparte + bump del puntero.
4. ✅ **Tests agregados**: 247 → **267**. Ver §7.9-bis.
5. 🟡 **Push pendiente** — `main` 2 commits adelante (`bd07ad1`, `3087fc5`). Sin pedido explícito no se pushea.

#### 7.9-bis Plan de tests — ejecutado

**C — deben disparar:** `****` + frase sin corridas → `*` · `*****` + frase con `***` → `***` · `=====` + frase con `===` → `===`
**C — NO deben disparar:** `NUMBERS` (no es corrida) · `dog` · `*` sola (largo < 2) · `**` presente literal en la frase · parámetro no-string · frase vacía
**B — deben disparar:** el caso real (test 11) · el espejo `Say "hi" to {user}`
**B — NO deben disparar:** valor ya literal · comillas que abrazan todo (test público de `substitute`) · dos spans que coinciden · solo spans delimitados · parámetro no-string
**Suite completa:** los 247 tests existentes + estos, y `flake8`/`mypy` limpios (recordar: `max-line-length = 120`, **E203 no ignorado** → slices sin espacio antes de `:`).

**Resultó en 20 tests nuevos** (`tests/test_output_validator.py`), agrupados por regla: 8 de B · 8 de C · 4 de la cadena A→B→C. Dos de ellos cubren **el orden de la cadena** y **los no-string**, que el plan original no listaba.

⚠️ **Un test hubo que re-apuntar, no borrar.** `test_snap_leaves_value_absent_from_prompt` afirmaba que `****` quedaba `****` — a través de `build_function_call`. Con la regla C eso ya no es cierto a través del pipeline, así que ahora **prueba la regla A directo** (`_snap_to_query_span("****", prompt)`). Sigue documentando lo que quería documentar ("A sola no hace nada") sin depender del pipeline.

---

## 8. 📐 MAPA DE MARGENES — dónde está el riesgo numérico real (2026-10-04)

### 8.1 La pregunta

El usuario tiene el presentimiento de que **los bugs del modelo desaparecen en máquinas ajenas**. `CLAUDE.md:51` lo respalda: *float16 (10 bits de mantisa) puede invertir empates técnicos de logit*, y el test de humo en GPU está pendiente. Nadie lo había cuantificado. Esta sección lo cuantifica.

### 8.2 Dónde está la decisión (importante para instrumentar)

El argmax **real** está en `token_filter.py:179`, dentro de la rama M1:

```python
best_id = max(range(len(logits)), key=lambda i: logits[i])   # ~151K posiciones
```

O sea: **argmax global sobre todo el vocabulario**, sin estrechar. Si ese token winner es estructuralmente válido, M1 devuelve `{best_id}` y el modelo decidió.

⚠️ **Trampa de instrumentación (costó una corrida entera de 40 min)**: espiar `_pick_best_token` (`constrained_generator.py:671`) **no sirve** — ahí ya llega `allowed` estrechado a 1 por M1, así que se mide al superviviente contra nada y sale "0 decisiones, sin margen". Hay que patchear **`compute_allowed_ids`** en el namespace del consumidor (`constrained_generator` lo importa por nombre).

### 8.3 La medición

281 steps con decisión real de modelo, sobre los 22 prompts (11 públicos + 11 privados):

| zona | steps |
|---|---|
| margen < 0,05 → **peligro float16** | **0** |
| margen < 0,10 (zona gris) | 0 |
| margen < 0,20 (ajustado) | 1 |
| margen < 0,50 (moderado) | 2 |
| margen ≥ 1,00 (cómodo) | **277** |

**Mínimo 0,193 · mediana 8,745 · máximo 18,29.** float16 sobre un logit de magnitud ~20-30 da un error absoluto de ~0,01-0,015: **no hay ninguna decisión a menos de 0,05**. Haría falta ~13x más ruido del que float16 entrega para mover lo más ajustado.

### 8.4 El hallazgo — y por qué encaja con §7

**El step más ajustado del proyecto entero es exactamente P9:**

```
0.19323  [public 9 step 19]  '****'  vs  ' *'   rama=M1
```

El modelo **genuinamente quiere** emitir `****` — no es un artefacto del filtro — y gana por apenas 0,193 logits sobre `' *'` (con espacio inicial). Eso explica por qué P9 era el test que fallaba: es la decisión **30x más ajustada** que el resto, donde lo típico es 5-8.

🔑 **El punto que importa**: la única decisión numéricamente frágil del proyecto **coincide con la única que ya tiene una regla de reparación**. El riesgo residual no es "un flip rompe un test que hoy pasa" — eso está cubierto por C. La regla quedó clavada justo donde el modelo está a punto de decidir mal.

### 8.5 Lo que esta medición NO cubre

- **La variable que más importa sigue sin testear: GPU / float16.** El 0,193 es un argumento analítico, no una ejecución.
- Batch > 1 · otra versión de torch · cuantización · kernel de atención distinto.
- **El margen top1-vs-top2 es una cota SUPERIOR del riesgo**: un flip sólo daña si el runner-up además es estructuralmente válido y produce respuesta incorrecta. El riesgo real es menor que lo medido.
- **Todos los márgenes son de 1 hilo.** El cruce crudo 1 vs 4 hilos es el pendiente §5.15.
- Instrumento: `/tmp/opencode/margins2.py` → `/tmp/opencode/margins2.json` (⚠ se pierden al reiniciar; §5.11).
