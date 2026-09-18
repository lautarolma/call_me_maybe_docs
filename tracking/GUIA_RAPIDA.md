# GUÍA RÁPIDA - SISTEMA DE SEGUIMIENTO
## Escuela 42 - 3 Proyectos

---

## 🚀 CÓMO USAR EL SISTEMA

### Al inicio de cada sesión:
1. **Pregúntame:** "¿Cuál es el estado de mi cronograma?"
2. **Yo ejecutaré:**
   - Verificación de fecha vs cronograma
   - Cálculo de avance esperado vs real
   - Recomendaciones específicas para el día
3. **⚠️ ALERTA ENDLINE OBLIGATORIA:** cargar la skill **avance-mvp** y
   emitir la alerta compacta (≤7 líneas) ANTES de empezar a trabajar.

### Al final de cada sesión:
1. **Actualiza PROGRESS_TRACKER.md** con:
   - Horas trabajadas
   - Tareas completadas
   - Bloqueos encontrados
2. **Haz commit** con mensaje claro
3. **Guarda contexto** en Engram
4. **⚠️ OBLIGATORIO — docs/ es un SUBMÓDULO PRIVADO:**
   - Si cambiaste algo en `docs/`, commiteá al repo PRIVADO
     (`cd docs && git add . && git commit -m "..." && git push`) y
     actualizá el puntero en el PRINCIPAL
     (`git add docs && git commit -m "chore: bump docs submodule" && git push`).
   - Nunca dejar docs/ sin versionar en el privado, ni el submódulo
     desactualizado en el público.

---

## 🛎 ALERTA ENDLINE — PREMISA EVALUABLE EN CADA FRONTERA

> **Regla de oro: cada apertura de jornada, cada cierre/apertura de task y
> el hito Task 4.3 disparan la evaluación de avance CONTRA EL DEADLINE.**
> Implementación: skill `avance-mvp` (`.opencode/skills/avance-mvp/SKILL.md`).
> La emisión de la alerta es OBLIGATORIA y COMPACTA (≤7 líneas) para no
> ensuciar la lectura.

### El endline en números (vigente al 18-sep-2026)

| Proyecto | Deadline | Estado real | Riesgo |
|----------|----------|-------------|--------|
| **call_me_maybe (MVP)** | **21/09/2026** | Phase 4 en curso (Task 4.1 ✓, 4.2-4.3 ⬜) | 🔴 ALTO: ~3 días de atraso; sin margen |
| Flying | debía 15-21/09 | ⬜ no iniciado | 🔴 ya corrido |
| Codection | 22-29/09 | ⬜ no iniciado | 🟠 depende de los otros |
| Cronograma general 3 proyectos | 03/10/2026 | en riesgo | 🔴 estimado real ~7-10/10 |

### Orden de ataque VIGENTE (inamovible hasta aviso)

```
call_me_maybe: 4.2 smoke real → 4.3 accuracy ≥90% → 5.1 validador → 5.2-5.3 pipeline
               → 6.4 DoD (6.1/6.2/6.3 recortables si aprieta) →  [MVP VERDE]
luego: Flying → Codection (alcance a renegociar según cierre de call_me_maybe)
```

### Puntos de foco (qué NO perder de vista)

1. **Task 4.2**: PRIMER contacto con Qwen3-0.6B real — thinking tokens
   (`<|begin_of_thought|>`), timing ~200ms/step, RAM/CPU.
2. **Task 4.3**: accuracy ≥90% (≥10/11) + <5 min total. La tarea MÁS incierta.
3. **Frontera post-4.3 = CHECKPOINT DE RE-EVALUACIÓN** (plan A/B/C en la skill):
   - **A** (≥90%): seguir orden → objetivo 21-22/09.
   - **B** (80-89% o >5min): 1 iteración acotada de prompts (≤0.5 jornada),
     decidir el MISMO día con el usuario.
   - **C** (<80% o bloqueante): escalar al usuario el MISMO día; renegociar
     alcance/deadline. No arrastrar el problema a la jornada siguiente.
4. **MVP = Phases 1-6 SOLO** (31 tasks). Phase 7/BONUS, refactors fuera de
   plan y documentación teórica extra están PROHIBIDOS hasta MVP verde.

---

## 📊 ARCHIVOS CLAVE

| Archivo | Propósito | Cuándo usar |
|---------|-----------|-------------|
| CRONOGRAMA_GENERAL.md | Plan maestro | Al inicio del día |
| PROGRESS_TRACKER.md | Registro diario | Al final del día |
| SESSION_START_PROTOCOL.md | Protocolo de inicio | Cada sesión |
| SISTEMA_SEGUIMIENTO.md | Resumen ejecutivo | Revisión semanal |

---

## 🎯 CHECKPOINTS (VIGENTES — el detalle vive en la 🛎 ALERTA ENDLINE)

| Checkpoint | Meta | Estado real |
|------------|------|-------------|
| **21/09/2026** | call_me_maybe MVP (Phases 1-6) entregado | 🟡 en riesgo — Phase 4 en curso |
| **post-Task 4.3** | Checkpoint accuracy → plan A/B/C | ⬜ pendiente (disparador obligatorio) |
| **03/10/2026** | 3 proyectos entregados (Flying + Codection) | 🟡 en riesgo — sin iniciar |

> Los checkpoints históricos (7/14/29 sept) quedaron obsoletos con el desfase
> real del proyecto; no usarlos como referencia.

---

## 🚨 ALERTAS RÁPIDAS

### Si vas 1-2 días atrasado:
- **Acción:** Simplificar tareas, priorizar MVP
- **Tiempo extra:** 1-2 horas diarias

### Si vas 3+ días atrasado:
- **Acción:** Revisar alcance, reducir features
- **Tiempo extra:** Evaluar reducir scope

### Si no hay commits en 3 días:
- **Acción:** Revisar bloqueos, pedir ayuda

---

## 📝 COMANDOS RÁPIDOS

### Para iniciar sesión:
```
¿Cuál es el estado de mi cronograma?
```

### Para actualizar tracker:
```
Actualizo mi progreso del día
```

### Para revisar semana:
```
¿Cómo voy esta semana?
```

### Para evaluar retraso:
```
¿Cuánto estoy atrasado y qué hago?
```

---

## 🎯 PRIORIDADES

### 1. Entregar a tiempo
- Mejor 2 proyectos completos que 3 incompletos
- MVP funcional antes que perfección

### 2. Mantener calidad
- Tests básicos siempre
- Documentación incremental
- Commits diarios

### 3. No quemarse
- Máximo 8 horas diarias
- Descansos cada 2 horas
- Dormir bien

---

## 🆘 EMERGENCIA

### Si un proyecto se complica:
1. **Evaluar:** ¿Es bloqueador o optimizable?
2. **Decidir:** ¿Acelerar o ajustar alcance?
3. **Documentar:** ¿Qué aprendí?

### Si el tiempo no alcanza:
1. **Priorizar:** Call Me Maybe + Flying (ambos Python)
2. **Codection:** Entregar MVP mínimo
3. **Explicar:** En evaluación qué faltó

---

## 📈 ÉXITO

### Señales de que vas bien:
- Commits diarios
- Tests pasando
- Documentación actualizada
- Sin bloqueos mayores

### Señales de alarma:
- Sin commits en 2+ días
- Tests fallando sin fix
- Documentación desactualizada
- Estrés excesivo

---

*Sistema activo desde: 3 septiembre 2026*
*Deadline: 3 octubre 2026*
*¡Éxito! 💪*
