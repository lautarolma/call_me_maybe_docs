# SESSION_START_PROTOCOL.md
## Protocolo de Inicio de Sesión - Escuela 42

### 🎯 PROPÓSITO
Cada vez que inicies una jornada de trabajo, este protocolo se ejecuta automáticamente para:
1. Verificar tu avance real vs esperado
2. Identificar bloqueos o retrasos
3. Recomendar acciones específicas para el día
4. Mantener el ritmo necesario para cumplir el deadline

---

## 📋 CHECKLIST DE INICIO

### Paso 1: Contexto Rápido
```
¿Qué fecha es hoy? ________
¿Qué día del cronograma es? ________
¿En qué fase debería estar? ________
```

### Paso 2: Verificación de Avance
```
Proyecto actual: ________________
Fase completada: ________________
Último commit: ________________
Horas trabajadas hoy: ________________
```

### Paso 3: Evaluación
```
¿Voy a tiempo? [ ] Sí [ ] No
Si no, ¿cuántos días atrasado? ________
¿Cuál es la causa principal? ________
```

### Paso 4: Plan del Día
```
Objetivo principal: ________________
Tareas específicas: ________________
Bloqueos potenciales: ________________
Hora de fin estimada: ________________
```

---

## 🚨 ALERTAS AUTOMÁTICAS

### Si vas atrasado (1-2 días):
- **Acción:** Identificar tareas que pueden simplificarse
- **Prioridad:** Completar MVP antes que perfeccionar
- **Tiempo extra:** Considerar 1-2 horas adicionales

### Si vas atrasado (3+ días):
- **Acción:** Revisar alcance de cada proyecto
- **Prioridad:** Entregar funcionalidad básica completa
- **Tiempo extra:** Evaluar reducir features no esenciales

### Si vas adelantado:
- **Acción:** Mantener ritmo, no bajar la guardia
- **Prioridad:** Pulir calidad del código
- **Tiempo extra:** Considerar features bonus

---

## 📊 MÉTRICAS DE SEGUIMIENTO

### Diarias:
- Horas trabajadas
- Commits realizados
- Tareas completadas
- Bloqueos encontrados

### Semanales:
- Fases completadas vs planificadas
- Horas acumuladas vs estimadas
- Calidad del código (tests, linting)
- Documentación actualizada

---

## 🎯 RECOMENDACIONES POR FASE

### Call Me Maybe (Python + ML):
- **Enfoque:** Completar fases 2-7 secuencialmente
- **Tiempo estimado:** 48 horas (12 días)
- **Clave:** No perderte en detalles de ML, priorizar funcionalidad

### Flying (Python + Pathfinding):
- **Enfoque:** Parser → Simulación → Pathfinding → Visual
- **Tiempo estimado:** 28 horas (7 días)
- **Clave:** Usar BFS primero, optimizar después

### Codection (C + Concurrencia):
- **Enfoque:** Estructura → Hilos → Mutexes → Lógica → Planificación
- **Tiempo estimado:** 32 horas (8 días)
- **Clave:** No aprender concurrencia desde cero, usar recursos existentes

---

## 🔄 FLUJO DE TRABAJO RECOMENDADO

### Mañana (2-3 horas):
1. Ejecutar protocolo de inicio (5 min)
2. Revisar cronograma del día (5 min)
3. Trabajar en tarea principal (2-2.5 horas)

### Tarde (2-3 horas):
1. Continuar tarea o cambiar a nueva (si completada)
2. Tests y documentación (30 min)
3. Commit y actualización de tracker (15 min)

### Cierre (15 min):
1. Documentar avance en PROGRESS_TRACKER.md
2. Identificar tareas para mañana
3. Guardar contexto en Engram

---

## 📝 PLANTILLA PARA NOTES DE SESIÓN

```markdown
### [FECHA] - [DÍA DEL CRONOGRAMA]

**Proyecto:** [Nombre]
**Fase:** [Número/Nombre]
**Horas trabajadas:** [X]

**Avance:**
- [ ] Tarea 1
- [ ] Tarea 2
- [ ] Tarea 3

**Bloqueos:**
- [Descripción del bloqueo]

**Decisiones tomadas:**
- [Decisión y justificación]

**Para mañana:**
- [Tarea específica]
```

---

*Protocolo activo desde: 3 septiembre 2026*
*Última actualización: 3 septiembre 2026*
