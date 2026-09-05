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

## 📊 ARCHIVOS CLAVE

| Archivo | Propósito | Cuándo usar |
|---------|-----------|-------------|
| CRONOGRAMA_GENERAL.md | Plan maestro | Al inicio del día |
| PROGRESS_TRACKER.md | Registro diario | Al final del día |
| SESSION_START_PROTOCOL.md | Protocolo de inicio | Cada sesión |
| SISTEMA_SEGUIMIENTO.md | Resumen ejecutivo | Revisión semanal |

---

## 🎯 CHECKPOINTS IMPORTANTES

### 7 septiembre (Fin Semana 1):
- Call Me Maybe: Phase 2-3 en progreso
- Horas acumuladas: 20h

### 14 septiembre (Fin Fase 1):
- Call Me Maybe: **COMPLETO**
- Horas acumuladas: 48h

### 21 septiembre (Fin Fase 2):
- Flying: **COMPLETO**
- Horas acumuladas: 68h

### 29 septiembre (Fin Fase 3):
- Codection: **COMPLETO**
- Horas acumuladas: 100h

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
