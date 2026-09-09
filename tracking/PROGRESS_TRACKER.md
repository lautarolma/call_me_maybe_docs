# PROGRESS_TRACKER.md
## Tracking de Avance - Escuela 42

### Última Actualización: 9 septiembre 2026

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

### 9 septiembre 2026 (noche — cierre de sesión, el usuario fue a descansar)
- **Recorrido**: parada 8 PRESENTADA pero NO confirmada. El usuario se fue con disonancias sin resolver (unicode/decodificación entre dos análisis) → **queda PENDIENTE de re-explicación** (retomar en la próxima sesión con: byte-to-unicode, roundtrip identidad, `model.decode` vs `encode+decode`).
- **Hallazgo BUG-003 (doc ↔ código divergentes en vocab_loader)**: la spec (Task 1.9 + didáctico) enseñaba `token_text.encode('utf-8').decode('utf-8')` — roundtrip identidad que NO deshace la byte-to-unicode table → índice por 'Ġ' inservible. El código real usa `model.decode([token_id])` — correcto. Doc actualizada al approach real (Task 1.9 + didáctico + TeoricNotes).
- **Desmentido registrado**: circuló la afirmación "vocab_loader no tiene test unitario / necesita el modelo real" — FALSO: `TestLoadVocab` con FakeModel existe (tests/test_loader.py L95-125) y pasa.
- **Código NO tocado** (solo docs). 37 tests siguen en verde, flake8 + mypy limpios.
- **Próxima sesión**: 1) re-explicar parada 8 (unicode/decodificación) hasta confirmación → 2) recién ahí marcar la parada 8 y pasar a la 9 (`src/prompt/prompt_builder.py`) → 3) Phase 3 (Decoder Core) sigue como próximo hito de implementación.
- 🚨 **PENDIENTE CRÍTICO encontrado al cierre**: `src/models/function_definition.py` tiene un cambio SIN commitear que rompe el código: `name: str` fue cambiado a `name: st` (typo — `st` no existe en ningún lado del proyecto → NameError). El árbol estaba LIMPIO al inicio de la sesión → el cambio apareció durante esta (posible otra sesión activa o edit a mano). **NO se commiteó ni se revirtió — queda en el working tree para que el usuario decida** (revertir o corregir). Verificar ANTES de cualquier corrida de tests.

---

*Este archivo se actualiza al inicio de cada sesión de trabajo*
