# PROGRESS_TRACKER.md
## Tracking de Avance - Escuela 42

### Última Actualización: 18 septiembre 2026

---

## 📈 MÉTRICAS GENERALES

| Proyecto | Días Completados | Fases Completadas | Estado |
|----------|------------------|-------------------|--------|
| Call Me Maybe | 2/12 | 2/7 | 🟢 En progreso |
| Flying | 0/7 | 0/6 | ⚪ No iniciado |
| Codection | 0/8 | 0/8 | ⚪ No iniciado |

---

## 📅 SEGUIMIENTO DIARIO

### Semana 1 (3-7 septiembre)

**3 septiembre (Día 1)**
- Horas trabajadas: __
- Avance: Call Me Maybe - Verificación de Phase 1
- Bloqueos: Ninguno
- Notas: Inicio de cronograma general

**4 septiembre (Día 2)**
- Horas trabajadas: __
- Avance: __
- Bloqueos: __
- Notas: __

**5 septiembre (Día 3)**
- Horas trabajadas: __
- Avance: __
- Bloqueos: __
- Notas: __

**6 septiembre (Día 4)**
- Horas trabajadas: __
- Avance: Recorrido de código — paradas 4-5 consolidadas (output.py + function_definition.py)
- Bloqueos: Ninguno
- Notas: Confirmada parada 5 (function_definition.py), listo para parada 6 (input_loader.py). Día buffer usado para consolidar M2 antes de Phase 3.

**7 septiembre (Día 5)**
- Horas trabajadas: __
- Avance: Recorrido de código — parada 6 confirmada (input_loader.py, short-circuit de condiciones)
- Bloqueos: Ninguno
- Notas: Listo para parada 7 (function_loader.py)

---

### Jornada Nocturna — 4/5 septiembre 2026 (Día 10 del cronograma)

- **Horas hiperfoco**: ~5h (ritmo pomodoro 30min trabajo / 30min pausa)
- **Pausas**: 30min guitarra, 30min cena, siesta 03:00–06:00 (3h)
- **Horas netas de código**: ~5h reales
- **Avance**:
  - ✅ Phase 2 Prompt Engineering COMPLETA (Task 2.1, 2.2, 2.3 + DoD cerrado)
    - `src/prompt/prompt_builder.py` + 13 tests → 36 tests en verde, lint limpio
  - ✅ Ajuste cronograma Opción B (+1h/día, Día 10 como recuperación, entrega 21/09 intacta)
  - ✅ Nota B8 (objects/arrays) en `function_definition.py`
  - ✅ Sección ARGPARSE en `TeoricNotes.md`
  - ✅ `PERFIL_TRABAJO.md` creado (10 secciones, guardado en Engram)
  - ✅ Recorrido de código: paradas 1-3 completas (`__main__.py`, `cli.py`, `pipeline.py`)
- **Bloqueos**: Ninguno
- **Estado frente al plan**: ✅ AL DÍA — sin desfasaje. Listo para Phase 3 (Día 11, Lun 09/07)

---

## 🎯 CHECKPOINTS

### Checkpoint 1: 7 septiembre (Fin Semana 1)
- [ ] Call Me Maybe: Phase 2-3 en progreso
- [ ] Horas acumuladas: 20h
- [ ] Commits: 5+

### Checkpoint 2: 14 septiembre (Fin Fase 1)
- [ ] Call Me Maybe: Completo
- [ ] Horas acumuladas: 48h
- [ ] Commits: 15+

### Checkpoint 3: 21 septiembre (Fin Fase 2)
- [ ] Flying: Completo
- [ ] Horas acumuladas: 68h
- [ ] Commits: 25+

### Checkpoint 4: 29 septiembre (Fin Fase 3)
- [ ] Codection: Completo
- [ ] Horas acumuladas: 100h
- [ ] Commits: 35+

---

## 🚨 ALERTAS Y ACCIONES

### Si vas atrasado:
1. Identificar causa raíz
2. Evaluar si es bloqueador o optimizable
3. Tomar decisión: acelerar o ajustar alcance
4. Documentar lección aprendida

### Si vas adelantado:
1. No bajar la guardia
2. Usar tiempo extra para pulir
3. Considerar features bonus
4. Documentar buenas prácticas

---

## 📝 NOTAS DE SESIÓN

### 3 septiembre 2026
- Se creó cronograma general para 3 proyectos
- Call Me Maybe tiene Phase 1 completa
- Flying y Codection sin empezar
- Estrategia: secuencial (Call Me Maybe → Flying → Codection)

### 4-5 septiembre 2026 (Jornada nocturna, Día 10)
- **Hito**: Phase 2 cerrada — prompt_builder + 13 tests, 36 tests en verde, lint limpio
- **Cronograma ajustado**: Opción B (+1h/día desde D10, entrega 21/09 intacta)
- **Recorrido de código**: paradas 1-3 completas, retomar desde parada 4 (output.py)
- **Metodología**: pomodoros 30/30 funcionaron, 5h hiperfoco con siesta 03-06
- **Módulo más difícil**: Phase 3 (Decoder Core) arranca Día 11 (Lun 09/07)
- **Engram**: perfil de trabajo guardado (topic `perfil/usuario-trabajo`), posición recorrido guardada (topic `recorrido/posicion`)

### 6 septiembre 2026 (Día buffer, domingo)
- **Recorrido de código**: paradas 4-5 CONSOLIDADAS — output.py (ficha de registro) + function_definition.py (Literal, ficha anidada, validador `mode="after"`, andamio de name). Parada 5 confirmada por el usuario.
- **Estado frente al plan**: ✅ AL DÍA, sin desfasaje. Próximo archivo del recorrido: parada 6 (`src/loader/input_loader.py`).
- **Próximo hito de implementación**: Phase 3 (Decoder Core) arranca Día 11 — Lunes 7 septiembre (Task 3.1, `state.py`).

### 8-9 septiembre 2026 (retomado tras suspensión + error "certificate is not yet valid" por reloj desincronizado)
- **Recorrido**: parada 7 CONFIRMADA (`function_loader.py`) — con hallazgo y corrección de BUG-002:
  - Asimetría detectada: `input_loader` validaba lista vacía, `function_loader` no → `[]` pasaba en silencio (0 funciones = falla asegurada contra el corrector)
  - Duplicados O(n²) con `names.count()` → fail-fast O(n) con `set` en una sola pasada
  - Decisión: Opción A (rechazo duro) aprobada por el usuario → 37 tests en verde (+1 nuevo), flake8 + mypy limpios, E2E OK
  - Doc actualizada: BITACORA_BUGS (BUG-002), PLAN_IMPLEMENTACION (Task 1.8), PLAN_DIDACTICO (sección cargador), TeoricNotes (sección Counter/set/fail-fast), RECORRIDO (parada 7)
- **Estado frente al plan**: ✅ AL DÍA. Parada 8 (`vocab_loader.py`) EN REVISIÓN — pendiente de re-explicación.

### 9 septiembre 2026 (noche — cierre de sesion, el usuario fue a descansar)
- **Recorrido**: parada 8 PRESENTADA pero NO confirmada. El usuario se fue con disonancias sin resolver (unicode/decodificacion entre dos analisis) -> queda PENDIENTE de re-explicacion (retomar en la proxima sesion con: byte-to-unicode, roundtrip identidad, `model.decode` vs `encode+decode`).
- **Hallazgo BUG-003 (doc <-> codigo divergentes en vocab_loader)**: la spec (Task 1.9 + didactico) ensenaba `token_text.encode('utf-8').decode('utf-8')` — roundtrip identidad que NO deshace la byte-to-unicode table -> indice por 'G' inservible. El codigo real usa `model.decode([token_id])` — correcto. Doc actualizada al approach real (Task 1.9 + didactico + TeoricNotes).
- **Codigo NO tocado** (solo docs). 37 tests siguen en verde, flake8 + mypy limpios.
- **Proxima sesion**: 1) re-explicar parada 8 (unicode/decodificacion) hasta confirmacion -> 2) recien ahi marcar la parada 8 y pasar a la 9 (`src/prompt/prompt_builder.py`) -> 3) Phase 3 (Decoder Core) sigue como proximo hito de implementacion.

### 9 septiembre 2026 (tarde — segunda sesion)
- **Recorrido**: parada 8 CONFIRMADA tras re-explicacion de unicode/byte-to-unicode/roundtrip identity. BUG-003 ya documentado en sesion anterior.
- **Typo corregido**: `function_definition.py` tenia `name: st` (de sesion paralela) -> revertido a `name: str`. Working tree limpio.
- **37 tests en verde** (uv run pytest).
- **Parada 9 INICIADA**: `src/prompt/prompt_builder.py` — en revision.
- **Estado frente al plan**: AL DIA. Proximo paso: completar revision parada 9 -> Phase 3 (Decoder Core) como proximo hito.

---

### 9-11 septiembre 2026 (sesion retomada)
- **Recorrido**: parada 8 CONFIRMADA (re-explicacion byte-to-unicode, roundtrip identity, tokens especiales vs bytes incompletos) -> parada 9 CONFIRMADA (`prompt_builder.py`: format deferred vs f-string, enumeracion de funciones, tipos como contrato con el decoder).
- **Recorrido de codigo COMPLETO** (paradas 1-9). Queda opcional la parada 10 (tests como referencia cruzada).
- **Typo corregido**: `name: st` -> `name: str` en function_definition.py (error de sesion paralela). Working tree limpio. 37 tests en verde.
- **Estado frente al plan**: AL DIA. Proximo hito: **Phase 3 (Decoder Core)** — arranca con Task 3.1 (`state.py`).

### 14 septiembre 2026 (Día 16 — retomado, pre-sesión de madrugada)
- **Checkpoint registrado al CERRAR jornada** (no al inicio, como pide el header — esta entrada se escribe para retomar).
- **Estado REAL verificado contra el código** (no de memoria):
  - ✅ Phase 1 (loaders+models) y Phase 2 (prompt_builder) COMPLETAS — `src/loader/*`, `src/models/*`, `src/prompt/prompt_builder.py`.
  - ✅ **Task 3.1 (state machine) COMPLETA**: `src/decoder/state.py` — 451 líneas, 15 fases (`DecoderPhase` enum str), `simulate()`, `update_from_text()` (atómico), `expected_first_chars()`, `_advance_char()` con `match/case`, handlers por rol (ROOT/OBJECT/KEY/STRING/NUMBER/LITERAL/VALUE_END), `@dataclass(slots=True)`.
  - ✅ `tests/test_state.py` EXISTE y está desarrollado (383 líneas: ROOT→COMPLETE char-por-char, keys_enclosed solo keys de parameters, slots sin `__dict__`, edge cases de number/literal/escape).
  - ❌ **Tasks 3.2–3.5 PENDIENTES**: `trie.py`, `schema_validator.py`, `token_filter.py` no existen; `test_trie.py`/`test_token_filter.py` no existen.
  - ⚠️ PROGRESS_TRACKER previo decía "37 tests / 9-11 sept" — DESACTUALIZADO (no cuenta Phase 3).
- **Bonus B5 documentado en `PLAN_IMPLEMENTACION.md`** (Inciso 5.1, L814-846): prefix injection EVALUADA Y DESCARTADA (impacto ~1-5%, acoplamiento al formato, riesgo accuracy) + estado real de los 4 items del stub B5 (batch/KV-cache descartados, profiling válido, `__slots__` ya implementado). Recomendado en su lugar: opportunistic masking post-profiling.
- **Nota teórica añadida** a `docs/notes/TeoricNotes.md` (sección SYNTAX vs SEMÁNTICA): el state machine valida gramática JSON, NO reglas de negocio (key vacía pasa; schema_validator/Task 3.3 es quien rechaza); casos edge confirmados contra código (`{"":1}` OK, `{"":}` falla, `{"a" 1}` falla, `{"":"x"}` OK, `""` OK).
- **Estado frente al plan**: ATRASADO ~5-6 días vs CRONOGRAMA_TRABAJO (hoy Día 16 esperaba Task 4.1). Quedan 13 tasks en ~7 días (3.2-3.5 + Phases 4-6).
- **Próximo paso**: Task 3.2 — `src/decoder/trie.py` + `tests/test_trie.py` (build_trie con las 5 funciones reales, valid_next_chars("fn_a") == {"d"}, is_complete_name).
- **📌 PENDIENTE DIDÁCTICO (anotado 14 sept)**: traducir las 2 regex de numbers de `src/decoder/state.py` (L49-54) a formato entendible — `_NUMBER_PREFIX_RE` ("número a medio terminar", valida PREFIXO incremental char-por-char en `_step_number`) vs `_NUMBER_RE` ("número completo", decide cierre con `,`/`}`/ws). El usuario NO consigue leerlas con claridad: demasiado abstracta la delimitación de casos. Hacer una sección en `TeoricNotes.md` con desglose pieza por pieza (signo, cero líder, fracción, exponente), los casos clave (`2.` admite solo dígitos, `2e` admite digits/+-/terminal, `2.e` DEBE fallar, leading zeros rechazados), y por qué x2 regex en vez de una estricta única.
- **📌 REVISAR MAÑANA (pendiente de decisión del usuario)**: `docs/notebooklm_sources/state_machine.md` (31 KB, creado 12 sept) está SIN VERSIONAR en el submódulo docs. Decidir si se trackea (`git -C docs add notebooklm_sources/ && git -C docs commit -m "docs: add notebooklm state machine source" && git -C docs push` + bump puntero) o se deja fuera (material transitorio de NotebookLM). El usuario NO decidió al cierre de la jornada — revisar a primera hora sin presión.

---

### 16 septiembre 2026 (jornada de ayer — CIERRE de sesión, retomado tras el 14-sep)

- **Horas trabajadas**: registro de cierre (jornada nocturna).
- **Avance REAL verificado contra el código y git log** (no de memoria):
  - ✅ **Task 3.2-3.3 COMPLETAS** (commit `cac725a`): `src/decoder/trie.py` (build_trie, find_node, is_complete_name, valid_next_chars) + `src/decoder/schema_validator.py` (SchemaContext: update, current_expected_type, required_keys_remaining, all_required_present, can_close_params) + tests.
  - ✅ **Task 3.4 COMPLETA** (commit `ac9cd10`, 850 insertions): `src/decoder/token_filter.py` (compute_allowed_ids — pre-filtro Fase 1 por primer char → simulación Fase 2 → post-filtro Fase 3) + `allows_token` con las 4 cláusulas ANDed (name→trie, param key con trigger por CAMBIO de current_key, value type, params close con keys_enclosed SIMULADO) + `tests/test_token_filter.py` (vocab mock BPE-realista, ~22 tests).
  - ✅ **Suite completa en verde: 151 tests** (129 baseline + 22 nuevos de test_token_filter), flake8 + mypy limpios (18 archivos), `make lint` OK.
  - ⚠️ En la jornada se corrigieron 3 fallos del primer run: (1) byte tokens del mock vocab no deben ir a buckets de chars (BYTE_IDS), (2) walk de `"Javier"` (el token ya incluye las comillas de cierre), (3) test de value-type pineaba el gap B8 como comportamiento esperado.
- **Estado frente al plan**: la Fase 3 del decoder (Tasks 3.1-3.4) quedó COMPLETA — pero sigue el desfasaje acumulado vs cronograma original (14-sep: ATRASADO ~5-6 días). La jornada de ayer cerró las tasks PENDIENTES del decoder, lo que reduce la deuda de Phase 3 a cero. Siguen pendientes Phases 4-6.
- **Nota del usuario sobre la jornada de ayer**: la percibió como "de poco avance" (en términos de cronograma: solo se recuperó el atraso de Phase 3, no se avanzó sobre Phase 4). Registro la percepción, aunque en volumen de trabajo fueron 2 commits grandes (850 inserciones).
- **Pendiente para la jornada de hoy**: documentar los ESQUEMAS del filter (NIVEL_1 pipeline, NIVEL_2 MAPA 3 bandas, cláusulas C1-C4, slow-motion, gaps) en TeoricNotes.md + trackeo de sesiones.

### 17 septiembre 2026 (hoy — APERTURA de sesión, 06:00)

- **Horas trabajadas**: 1.5h al momento de registrar (cuentan desde las 06:00).
- **Avance**:
  - ✅ Esquemas del filter (Task 3.4) documentados en TeoricNotes.md — NIVEL_1 pipeline, NIVEL_2 MAPA con las 3 bandas (estructura/values/reconexión) + ubicación de cláusulas ◈1-◈4 + reglas de lectura, árboles de decisión C1-C4, slow-motion `, "b": 3.0`, tabla de gaps.
  - ✅ Trackeo: cierre de la jornada de ayer (16-sep) + apertura de hoy con 1.5h acumuladas.
  - ✅ CRONOGRAMA_GENERAL actualizado: Phase 2-3 marcadas, estado general "Phase 3 ✅ (decoder 16/09)".
- **Plan del día** (según SESSION_START_PROTOCOL):
  - Objetivo principal: cerrar documentación teórica de Phase 3 + arrancar **Task 4.1 (el loop de generación)**.
  - Prioridad: el usuario está en modo teórico (1h máx por la regla de oro del cronograma) → después implementación.
- **Estado frente al plan**: AL DÍA con el retrabajo; el desfasaje de Phase 3 quedó saldado ayer — el siguiente hito real es Task 4.1.

### 17 septiembre 2026 (hoy — CIERRE de sesión, ~5h de hiperfoco)

- **Horas trabajadas**: ~5h en TOTAL (1.5h al apertura + ~3.5h de estudio en hiperfoco). Fue puro ESTUDIO del flujo de implementación: schema_validator + token_filter completos, sin tocar código de producción (solo docstrings).
- **Avance**:
  - ✅ Documentados los ESQUEMAS del filter en TeoricNotes.md (NIVEL_1, NIVEL_2 3 bandas, C1-C4, slow-motion, tabla de gaps).
  - ✅ **Análisis estructural de las 4 cláusulas de allows_token** (a pedido del usuario con su resumen del defecto): verificado contra el código que las cláusulas juzgan estados límite pre/post y que `return True` = abstención cuando el token no coincide con las condiciones buscadas (L235/302/336/338/380/382/385/405).
  - ✅ **HALLAZGO nuevo documentado** en el docstring de `_allows_param_key`: un token que ENTRA a parameters y abre la PRIMERA key en el mismo paso (`'", "parameters": {"a'`) arranca en depth 0 commiteado → `self._depth != 1` abstiene y la key jamás se valida semánticamente (ni en este token ni en los siguientes, porque el trigger por cambio no vuelve a disparar). Gap REAL no documentado previamente.
  - ✅ Decisión de arquitectura: NO re-diseñar el filter (los gaps requieren tokens largos multi-fase, raros en BPE real); se documentan en docstrings y el pase fino post-argmax queda como vía de cierre viable en Task 4.1 (re-simular 1 token por step, costo despreciable).
  - ✅ Trabajo en contexto de Progreso: Mapas y esquemas generados con Python para alineación perfecta, `�E`→`❌` corregido, scan sin caracteres corruptos.
- **Bloqueos**: Ninguno.
- **Estado frente al plan**: ✅ AL DÍA con la teoría de Phase 3. **El usuario declara estar listo para Task 4.1 (el loop de generación) con teoría y entendimiento al día.**
- **Próximo paso**: Task 4.1 — generator que llama `compute_allowed_ids` + argmax (el pase fino post-argmax del hallazgo entra como inciso de esa task).

### 17 septiembre 2026 (hoy — IMPLEMENTACIÓN de Task 4.1 + Inciso 4.1.1)

- **Horas trabajadas**: jornada completa (~5h teoría previa + implementación en esta sesión).
- **Avance REAL verificado contra el código y la suite** (no de memoria):
  - ✅ **Task 4.1 COMPLETA**: `src/decoder/constrained_generator.py` (NUEVO) — `generate()` con el ciclo del plan (encode → compute_allowed_ids → argmax sobre allowed → commit con `update_from_text` → decode), `MAX_TOKENS = 200`, corte sin repair en `allowed` vacío.
  - ✅ **Inciso 4.1.1 (pase fino post-argmax) IMPLEMENTADO**: `_pick_best_token` + `_passes_fine_validation` — re-simulación char-por-char del GANADOR con SchemaContext fresco; si falla, se descarta y se rehace el argmax. Cierra los 5 gaps de BUG-004 (abstención de las cláusulas por estados límite): key inexistente en 1ra entrada a parameters, type erróneo en key+value+cierre, cierre de params sin required, duplicado exacto de key, y COMPLETE sin params.
  - ✅ `src/decoder/schema_validator.py`: flag aditivo `_params_object_seen` (lo setea `update()` al ver PARAMS_OBJECT; NO rompe el contrato de Task 3.3) + `has_seen_params_object()`, sembrado desde el schema real al fino.
  - ✅ `tests/test_constrained_generator.py` (NUEVO, 12 tests): 9 del pase fino (gaps 2/3/4, slip duplicado, params object) + 3 end-to-end con FakeModel (argmax sobre target fijo, max_tokens=0, garbage target→ second best).
  - ✅ **Suite completa: 163 tests en verde** (151 + 12 nuevos), flake8 + mypy limpios.
  - ✅ **BUG-004 documentado** en `docs/tracking/BITACORA_BUGS.md` (severidad ALTA conceptual, causa raíz: contrato pre/post de allows_token solo ve 2 puntos del recorrido por token, la abstención es límite de información no bug de lógica).
  - ✅ **Inciso 4.1.1 marcado en `docs/design/PLAN_IMPLEMENTACION.md`** (entre Task 4.1 y 4.2): mecanismo (sembrado de flag, orden allows→update, gaps cerrados), costos (1 re-simulación por step), desvíos (id2decoded, _pick_best_token).
- **Bugs de implementación corregidos durante la jornada** (registrados en BUG-004, lecciones 3-4): (1) el pase fino actualizaba el schema ANTES de `allows_token` → `self.* == new_state.*` y las cláusulas de cambio/depth no gatillaban — el orden correcto es allows(pre)→update(post); (2) los tokens de test del gap 4 arrancaban con `"` desde VALUE_END (sintaxis inválida: solo `,`/`}`/ws son válidos, ver `expected_first_chars`) — el token real es `', "parameters": {...}'`.
- **Bloqueos**: Ninguno.
- **Estado frente al plan**: ✅ Phase 4 arrancada (Task 4.1 completa). Próximo hito: Task 4.2 (smoke test con el modelo REAL — requiere el entorno con Qwen 0.6B).

### 18 septiembre 2026 (hoy — BUG-005 + sistema de alerta endline)

- **Horas trabajadas**: jornada de cierre de bug + sistema de seguimiento.
- **Avance REAL verificado contra el código y la suite** (no de memoria):
  - ✅ **BUG-005 RESUELTO** (dimensiones del flujo de generación): `Small_LLM_Model.encode()` devuelve tensor 2D [1,N] → el `.tolist()` directo dejaba `list[list[int]]` → `get_logits_from_input_ids` armaba tensor 3D y crasheaba con TypeError. Fix: `encode(prompt)[0].tolist()` + `prompt_length` guardado ANTES del loop (elimina el re-encode final). Tests: mock `_FakeTensor` ahora replica la forma 2D + assert de contrato en `FakeModel.get_logits` + test de regresión N-tokens (prompt de 3 ids inexistentes). **Suite: 164 green**, flake8 + mypy limpios. BITACORA_BUGS (BUG-005) + concordancia en PLAN_DIDACTICO y PLAN_IMPLEMENTACION (también se enlazó el desvío id2decoded donde el didáctico aún enseñaba id2token).
  - ✅ **Commits**: docs `b7ea228` + main `40b6cac` (bump) + `d607d4e` (fix), pusheados.
- **Evaluación de avance vs cronograma** (pedida por el usuario):
  - Real: Phases 1-3 + Task 4.1 (~20/31 tasks) · didáctico Empezando M10 (teoría al día) · suite 164 green.
  - Plan nominal: hoy Día 20 esperaba Phase 6 → **atraso ~3 días en implementación**.
  - Restan 9 tareas (4.2, 4.3, 5.1-5.3, 6.1-6.4) ≈ 36-38h vs ~20-24h disponibles hasta 21/09 → **riesgo ALTO** de entregar 23-24/09 si no se trabaja el finde (19-20/09, buffer).
  - Cascada: Flying (debía iniciar 15/09) y Codection sin arrancar → cronograma general (03/10) en riesgo; estimado real ~07-10/10.
- **Sistema ALERTA ENDLINE creado** (pedido del usuario: premisa evaluable en cada frontera):
  - Inciso **🛎 ALERTA ENDLINE** en `GUIA_RAPIDA.md`: tabla de endlines, orden de ataque vigente, puntos de foco, MVP = Phases 1-6 SOLO.
  - **Skill `avance-mvp`** (`.opencode/skills/avance-mvp/SKILL.md`, versionada en el repo): trigger en apertura de jornada, frontera de task y post-Task 4.3; salida COMPACTA ≤7 líneas; plan A/B/C post-4.3 (A: ≥90% seguir · B: 80-89% iterar prompts ≤0.5 jornada · C: <80% escalar el MISMO día).
  - Checkpoints viejos de GUIA_RAPIDA reemplazados por versión vigente; CRONOGRAMA_GENERAL actualizado a "Phase 4 en curso (18/09)".
- **Bloqueos**: Ninguno.
- **Estado frente al plan**: Phase 4 en curso (Task 4.1 + BUG-005 ✓). **Próximo hito: Task 4.2 — smoke test con el modelo real Qwen3-0.6B** (primer contacto real; sin bloqueos conocidos desde el fix).

*Este archivo se actualiza al inicio de cada sesión de trabajo*

### 18 septiembre 2026 (hoy — CIERRE: Task 4.2 smoke PASS funcional + hallazgo de performance)

- **Horas trabajadas**: sesión tarde (~2h).
- **Avance REAL verificado contra la ejecución** (no de memoria):
  - ✅ **Task 4.2 COMPLETA (criterio funcional del plan)**: smoke test con el modelo REAL **Qwen3-0.6B** (`/tmp/opencode/smoke_42.py` + copia de respaldo en `.scratch_task42/`). Query `"What is the sum of 2 and 3?"` (primer prompt real de `function_calling_tests.json`): `generate()` devolvió JSON parseable con `name: "fn_add_numbers"` y `parameters: {"a": 2, "b": 3}`, `ok(COMPLETE)=True`. **PASS funcional** ✅
  - ✅ Entorno validado: modelo carga 20.5s, vocab 151.643 tokens / 17.805 buckets en 25.4s, prompt de 190 tokens.
  - 🔴 **HALLAZGO CRÍTICO de performance (registrado en Engram, topic `phase4/performance-timing`)**: forward real ≈ **2.842s/step @32 tokens en la VM actual (3 cores, 7.8Gi)** — el plan estimaba 150-200ms/step. `generate()` completo: **~308s para UN solo prompt**. Proyección: 11 prompts (Task 4.3) ≈ **55+ min** vs KPI del subject (<5 min en CPU). **Incumplimiento por más de 10x.**
- **Análisis (con el usuario)**:
  - Causa raíz: SDK re-corre el modelo completo en cada step (sin KV-cache) + ventana creciente (190→260 tokens) + CPU débil. El pase fino NO es el problema (simula chars, no llama al modelo).
  - Scaling torch CPU para 0.6B NO es lineal (se aplana ~16 cores): a 8 cores (máquina nueva: 24Gi/8 proc) ≈ 2.5-3x → ~19-22 min para 11 prompts — **aún fuera de 5 min**; CPU-only necesitaría 40-64+ cores de servidor con incertidumbre; la vía real es GPU (CUDA/MPS, SDK auto-detects) que da 20-50x.
- **DECISIÓN DEL USUARIO**: la VM pasa a **8 procesadores / 24Gi RAM** (realista al dispositivo de entrega) — "mejorar hardware no es la solución", se testea en condiciones reales y se decide el KPI con evidencia. Pendiente: correr `bench_scaling.py` (resguardado en `.scratch_task42/`) en la máquina nueva ANTES de Task 4.3.
- **Checkpoint de retorno** (la sesión se cierra para reiniciar con más recursos):
  1. Al volver: máquina con 8 cores/24Gi → correr `uv run python .scratch_task42/bench_scaling.py` (bench de threads 1-8) ≈ 2-3 min.
  2. Re-correr smoke (`smoke_42.py`) si hace falta revalidar entorno.
  3. Task 4.3: 11 prompts, accuracy ≥90%, timing total — con el bench decidir si KPI <5 min es alcanzable o se justifica/mide contra la máquina real.
  4. Alerte ENDLINE post-4.3: aplicar plan A/B/C (skill avance-mvp).
- **Estado frente al plan**: Phase 4 en curso. Task 4.2 funcional ✅, performance bajo investigación con la máquina mejorada (18/09, ~2h).

### 18 septiembre 2026 (hoy — Task 4.3 COMPLETA: accuracy 91% VERDE + timing rojo por hardware)

- **Horas trabajadas**: sesión tarde-noche (cierre de 4.3 + inicio de auditoría de latencia).
- **Avance REAL verificado contra la ejecución** (no de memoria):
  - ✅ **Bench de scaling en la máquina ACTUAL (6 vCPU / 15Gi — NO la 8/24Gi pactada)** (`uv run python .scratch_task42/bench_scaling.py`, secuencia real de 190 tokens):
    - threads=1: **4540 ms/step** | threads=2: **2695** | threads=4: **2611 (ÓPTIMO)** | threads=6: **5123 (PEOR que 1 thread)**
    - Lectura: el scaling se aplana 2→4 (+3%) y COLLAPSA de 4→6 — evidencia de memory-bandwidth bound + topología compartida (host i7-7700HQ = 4 cores físicos / 8 HT; 6 vCPU mapean sobre 4 físicos + 2 HT compartidos).
  - ✅ **Task 4.3 COMPLETA** (`.scratch_task42/task43_accuracy.py`, 11 prompts, modelo REAL Qwen3-0.6B, threads=4 — el óptimo del bench):
    - **accuracy función: 11/11 (100%)** — el constrained decoder + prompt SIEMPRE eligen la función correcta.
    - **accuracy full (quality bar M14): 10/11 (91%) ≥ 90% ✅** — scoring corregido en 1 caso: prompt 10 `regex='([aeiouAEIOU])'` = grupo capturador, FUNCIONALMENTE EQUIVALENTE a `[aeiouAEIOU]`; el ground truth del script no contemplaba el signo de agrupación → error del SCORING, no del modelo.
    - Fallo real restante: **prompt 9** `replacement='NUMBER'` vs "with NUMBERS" del prompt (singular vs plural). Output JSON válido con source_string y regex correctos — fallo del modelo, no del scoring.
    - **timing generación: 34.9 min** (por prompt: 122-330s; el más caro = regex larga de substitute) | total con carga: 35.0 min. **KPI <5 min INCUMPLIDO en CPU** — rojo por HARDWARE, no por código.
    - Resultados persistidos en `.scratch_task42/results_43.json` (11 entries; output crudo guardado solo en los no-OK para no duplicar).
  - ✅ Suite sigue 164 green (no se tocó código de producción — solo scripts temporales de scratch).
- **Checkpoint post-Task 4.3 (plan A/B/C del skill avance-mvp)**:
  - Situación MIXTA: accuracy alcanza **A** (91% ≥ 90%) · timing cae en **B** (>5 min).
  - **Decisión PAUSADA a pedido del usuario**: primero AUDITORÍA de diseño y latencia (reevaluar arquitectura con agentes, verificar la hipótesis de "latencia especial de CPU", comparar con repo de un colega que el usuario aportará: `Mario_Call_me_maybe`). Recomendación en pie del usuario: Opción A — documentar timing como limitación física de la máquina real y seguir a Phase 5.
- **Hipótesis del usuario a verificar** ("¿puede estar ocurriendo una latencia especial en mi CPU?"):
  - Los datos del bench NO apoyan un defecto de CPU, sino límite físico: (a) el costo base del forward está dentro/mejor del rango teórico para 0.6B FP32 sin KV-cache; (b) el colapso 4→6 threads = topología compartida (4 físicos) + bandwidth.
  - Verificación complementaria pendiente en esta sesión: frecuencia/turbo/throttling, soporte AVX2, load del host, y comparativa de arquitectura vs repo de Mario.
- **Estado frente al plan**: Phase 4 COMPLETA funcionalmente (4.1-4.3) con accuracy verde; performance bajo AUDITORÍA (18/09, sesión tarde-noche).

*Este archivo se actualiza al inicio de cada sesión de trabajo*
