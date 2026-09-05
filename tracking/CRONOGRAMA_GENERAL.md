# CRONOGRAMA GENERAL - ESCUELA 42
## Deadline: 3 octubre 2026 (30 días calendario desde 3 septiembre)

---

## 📊 ESTADO ACTUAL

| Proyecto | Estado | Días Asignados | Horas Estimadas |
|----------|--------|----------------|-----------------|
| Call Me Maybe | Phase 1 ✅ | 12 días | 48h |
| Codection | Sin empezar | 8 días | 32h |
| Flying | Sin empezar | 5 días | 20h |
| Buffer/Contingencia | - | 3 días | 12h |
| **TOTAL** | - | **28 días** | **112h** |

---

## 🎯 OBJETIVO PRINCIPAL
Entregar los 3 proyectos a tiempo, priorizando:
1. **Completitud** sobre perfección
2. **Funcionalidad** sobre optimización
3. **Entrega** sobre conocimiento profundo

---

## 📅 CRONOGRAMA DETALLADO

### FASE 1: CALL ME MAYBE (3 sept - 14 sept) - 12 días

#### Semana 1: Foundation + Core (3-7 sept)
| Día | Fecha | Tarea | Meta | Estado |
|-----|-------|-------|------|--------|
| 1 | 3 sept | Verificar avance Phase 1, planificar Phase 2 | Confirmar base sólida | ⬜ |
| 2 | 4 sept | Phase 2: Vocabulary + Tokenization | Tokenizador funcional | ⬜ |
| 3 | 5 sept | Phase 2: Continuación + tests | Tests pasando | ⬜ |
| 4 | 6 sept | Phase 3: Decoder core (state machine) | State machine básica | ⬜ |
| 5 | 7 sept | Phase 3: Trie + filtro | Trie funcional | ⬜ |

#### Semana 2: Integration + Polish (8-14 sept)
| Día | Fecha | Tarea | Meta | Estado |
|-----|-------|-------|------|--------|
| 6 | 8 sept | Phase 3: Tests + integración | Phase 3 completa | ⬜ |
| 7 | 9 sept | Phase 4: Function calling | Detección de funciones | ⬜ |
| 8 | 10 sept | Phase 4: Validación de parámetros | Validación robusta | ⬜ |
| 9 | 11 sept | Phase 5: Constrained decoding | Decoding restringido | ⬜ |
| 10 | 12 sept | Phase 5: Optimización | Performance aceptable | ⬜ |
| 11 | 13 sept | Phase 6: Integration tests | Tests end-to-end | ⬜ |
| 12 | 14 sept | Phase 7: Polish + documentation | MVP completo | ⬜ |

---

### FASE 2: FLYING (15 sept - 21 sept) - 7 días

#### Desarrollo Intensivo
| Día | Fecha | Tarea | Meta | Estado |
|-----|-------|-------|------|--------|
| 13 | 15 sept | Parser: Lectura de archivos | Parser funcional | ⬜ |
| 14 | 16 sept | Parser: Validación + errores | Manejo de errores | ⬜ |
| 15 | 17 sept | Simulación: Turnos + movimiento | Simulación básica | ⬜ |
| 16 | 18 sept | Pathfinding: BFS implementation | BFS funcional | ⬜ |
| 17 | 19 sept | Pathfinding: Optimización + restricciones | Restricciones de capacidad | ⬜ |
| 18 | 20 sept | Visualización + output | Output formateado | ⬜ |
| 19 | 21 sept | Tests + documentation | Proyecto completo | ⬜ |

---

### FASE 3: CODECTION (22 sept - 29 sept) - 8 días

#### Desarrollo en C con Concurrencia
| Día | Fecha | Tarea | Meta | Estado |
|-----|-------|-------|------|--------|
| 20 | 22 sept | Estructura base + Makefile | Compilación limpia | ⬜ |
| 21 | 23 sept | Hilos básicos (pthread_create/join) | Hilos funcionando | ⬜ |
| 22 | 24 sept | Mutexes + sincronización básica | Acceso seguro a recursos | ⬜ |
| 23 | 25 sept | Lógica de negocio (estados de coder) | Estados implementados | ⬜ |
| 24 | 26 sept | Planificación FIFO + cooldown | FIFO funcional | ⬜ |
| 25 | 27 sept | Planificación EDF + heap | EDF implementado | ⬜ |
| 26 | 28 sept | Monitor thread + burnout detection | Detección precisa | ⬜ |
| 27 | 29 sept | Tests + depuración + norminette | Proyecto completo | ⬜ |

---

### FASE 4: BUFFER Y CONTINGENCIA (30 sept - 2 oct) - 3 días

| Día | Fecha | Tarea | Meta | Estado |
|-----|-------|-------|------|--------|
| 28 | 30 sept | Testing general de los 3 proyectos | Todo funcional | ⬜ |
| 29 | 1 oct | Correcciones finales + documentation | Documentación completa | ⬜ |
| 30 | 2 oct | Pre-entrega + checklist final | Listo para entregar | ⬜ |

---

## 📊 TRACKING DE PROGRESO

### Call Me Maybe
- [ ] Phase 1: Foundation ✅ (completada antes del cronograma)
- [ ] Phase 2: Vocabulary + Tokenization
- [ ] Phase 3: Decoder Core
- [ ] Phase 4: Function Calling
- [ ] Phase 5: Constrained Decoding
- [ ] Phase 6: Integration
- [ ] Phase 7: Polish

### Flying
- [ ] Parser
- [ ] Simulación
- [ ] Pathfinding
- [ ] Visualización
- [ ] Tests
- [ ] Documentation

### Codection
- [ ] Estructura base + Makefile
- [ ] Hilos básicos
- [ ] Mutexes + Sincronización
- [ ] Lógica de negocio
- [ ] Planificación FIFO
- [ ] Planificación EDF
- [ ] Monitor + Burnout
- [ ] Tests + Documentation

---

## 🔄 PROTOCOLO DE INICIO DE SESIÓN

**Cada vez que inicies una jornada de trabajo, ejecuta:**
1. `mem_context` para ver historial reciente
2. Verificar fecha actual vs cronograma
3. Calcular avance esperado vs real
4. Recomendaciones específicas para el día

---

## ⚠️ REGLAS DE ORO

1. **NO más de 1 hora de teoría por día** — si no entiendes algo, pide ayuda
2. **Commit diario** — aunque sea pequeño, deja registro
3. **Tests primero** — implementa tests antes de código complejo
4. **Documentación incremental** — no dejes todo para el final
5. **Pide ayuda temprano** — si algo toma más de 2 horas, consulta

---

## 🚨 SEÑALES DE ALERTA

- Si vas 2+ días atrasado → Recalibrar expectativas
- Si un proyecto toma el doble del tiempo → Evaluar MVP vs completo
- Si no hay commits en 3 días → Revisar bloqueos

---

*Última actualización: 3 septiembre 2026*
*Próxima revisión: 4 septiembre 2026 (inicio de sesión)*
