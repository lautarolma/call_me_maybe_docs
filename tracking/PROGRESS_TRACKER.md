# PROGRESS_TRACKER.md
## Tracking de Avance - Escuela 42

### Última Actualización: 17 septiembre 2026

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

*Este archivo se actualiza al inicio de cada sesión de trabajo*
