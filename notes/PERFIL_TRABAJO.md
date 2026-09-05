# 👤 Perfil de trabajo — laviles (call_me_maybe / escuela 42)

> Documento vivo: recopila las maneras, dinámicas y posturas que el usuario
> suele pedir en planificaciones, recorridos de código y correcciones.
> Propósito: servir de base para generar, en el futuro, un perfil de agente
> (AGENTS.md personal / skill) con estas especificidades ya incorporadas.
>
> **Origen de los datos**: memoria persistente (Engram, sesiones 2026-08-17 →
> 2026-09-04) + conversación del 2026-09-04 (jornada de cierre de Phase 2).
> Cada ítem con `[registro]` consta en memoria; con `[inf.]` fue inferido de
> la dinámica reciente de esta jornada.
> **Última actualización**: 2026-09-05

---

## 1. Contexto del usuario (quién es, de dónde viene) `[registro]`

- Estudiante de **escuela 42**, terminando el nivel C: `libft`, `get_next_line`, `push_swap`.
- Python: piscine + proyecto propio (a-maze-ing). **Sin trasfondo de programador formal**: aprendió "de oído" con tutorías de IA.
- Sabe qué es un token/logit y conceptos de procesamiento; **NO sabía BPE ni MVP** → la didáctica arranca desde ahí (junior-deep).
- Estudia en paralelo con **Google NotebookLM**: los documentos didácticos deben ser **autocontenidos** (cada módulo independiente).
- Meta actual: 3 proyectos de 42 (call_me_maybe, codection, flying) con **deadline firme 3 de octubre 2026**.

## 2. Idioma y tono `[registro]`

- **Español rioplatense con voseo** ("dale", "¿se entiende?", "es así de fácil", "hermano", "ponete las pilas").
- **Términos técnicos en inglés** (no traducir `token`, `decoder`, `constrained decoding`).
- Explicaciones **pedagógicas nivel junior-deep**: concepto → problema → solución → ejemplo.
- Estilo general ya configurado en `~/.config/opencode/AGENTS.md` (Senior Architect, profesor apasionado).

## 3. Comunicación y corrección — la postura central `[registro] + [inf.]`

> **El usuario lo dijo textualmente (2026-09-04)**: *"está bien el tono amable
> amigable, pero que prime lo verídico y la intención de que no me quede con
> una idea vaga o incorrecta porque intentes darme la razón"*.

1. **Prima la veracidad sobre la amabilidad.** Si se equivoca, decírselo con evidencia y explicación técnica del porqué.
2. **Si el agente se equivocó**: reconocerlo con prueba, sin excusas ("si estaba mal, reconocerlo con evidencia").
3. **Nunca darle la razón por cortesía.** Confirmar SOLO cuando esté verificado.
4. **Verificar antes de afirmar**: "dejame verificar" y chequear código/documentos antes de responder.
5. Cuando una afirmación del agente es incorrecta, el usuario espera la corrección del agente con explicación (no que insista).

## 4. Proceso de decisión `[registro]`

- **En decisiones que tocan su código, esperar SU decisión.** No decidir por él: presentar opciones y esperar la elección (ej: Opción B de excluir `tests/` de mypy; Opción B del ajuste de cronograma).
- Ofrecer **alternativas con tradeoffs** cuando corresponda, y dejar que elija.
- **Nunca asumir respuestas**: preguntar y esperar (regla del AGENTS.md: "When asking a question, STOP and wait for response").
- Las decisiones tomadas se **registran** (Engram + BITACORA_BUGS) con el razonamiento.

## 5. Planificación y seguimiento `[registro] + [inf.]`

- **Planes con estructura de fases y tareas numeradas** (fases 1-6 MVP, bonus 7 aparte).
- **MVP primero**: el bonus se agenda SOLO si sobra tiempo, nunca dentro del timeline.
- **Cronogramas con fecha de entrega firme.** Ante un desfase: NO estira la fecha; **añade hora extra por jornada** para compensar dentro del plazo ("pretendo tener el plan cumplido en el plazo estipulado").
- **Protocolo de inicio de jornada**: verificar el cronograma, informar avance, dar consejo/recordatorio del día.
- Le gusta la **matemática honesta del ajuste** (mostrar la cuenta: días-equivalente vs días disponibles).
- Documentos de plan autocontenidos, con checkboxes de DoD trazables.

## 6. Método de aprendizaje y revisión `[registro] + [inf.]`

- **Recorridos guiados POR DEMANDA**: el usuario dirige el ritmo, hace preguntas, ordena avanzar. El agente NO se adelanta de archivo.
- **Confirmación de entendimiento antes de avanzar**: insistir en "¿se entiende?" y esperar; si hay duda, resolverla en la parada.
- **Repaso deliberado de fases anteriores** al abordar las nuevas (recorrido arranca desde `__main__` aunque la fase nueva sea otra).
- **Prefiere entender el PORQUÉ** (diseño, tradeoffs) y no solo el cómo.
- Autotest exprés al cierre de cada parada (preguntas de verificación mental).
- Registro de aprendizaje en markdown: `TeoricNotes.md`, `RECORRIDO_EJECUCION.md` (pizarra con posición/progreso).

## 7. Documentación y registros que mantiene `[registro]`

- `docs/notes/TeoricNotes.md` — apuntes conceptuales.
- `docs/notes/RECORRIDO_EJECUCION.md` — pizarra del recorrido file-by-file (posición + notas por archivo + gotchas).
- `docs/tracking/BITACORA_BUGS.md` — bugs y procedimientos (entradas con fecha + contexto).
- Código **anotado con comentarios técnicos inline en español** (nivel junior-profundo) para estudio — todo `src/` comentado.
- Documentación de apuntes: le gusta que las aclaraciones/conceptos queden como **notas en docs**, no solo en conversación.

## 8. Restricciones de entorno (contexto físico) `[registro]`

- **RAM total 3.8GB** → cargar Qwen3-0.6B fp32 (2.4GB) OOM-killea (exit 137) sin flags.
  - **SIEMPRE**: `MALLOC_ARENA_MAX=1 OMP_NUM_THREADS=1 MKL_NUM_THREADS=1 uv run python -m src`.
- Disco limitado (27GB; snap viejas pendientes de sudo, ~1.5GB).
- PyTorch **CPU index** en pyproject.toml (evita ~4GB de paquetes nvidia).
- Presupuesto de tokens del agente como recurso (cache reads ~90%, gastar con criterio).

## 9. Convenciones de código y repo `[registro]`

- **Commits**: conventional commits, **NUNCA "Co-Authored-By" ni atribución IA**. No hacer build tras cambios.
- Sujeto 42: prohibidos `dspy`, `outlines`, `torch`, `transformers`, `huggingface` en `src/`; permitidos `numpy`, `json`, `pydantic` (obligatorio para datos I/O). Sin atributos privados del SDK.
- Linters: `flake8 .` + `mypy .` con los 5 flags del subject (Makefile `lint`); `lint-strict` = `--strict`.
- DoD por fase: tests verdes + `make lint` limpio + ejecución end-to-end.

## 10. Cómo usar este perfil a futuro

Este documento puede convertirse en:
1. **`~/.config/opencode/AGENTS.md`** (personal) — agregando las secciones 2-4 y 6 al AGENTS.md global existente.
2. **Un skill** (`usuario-perfil` en `~/.config/opencode/skills/`) — para que el pattern se auto-cargue por contexto.
3. **Prompt de sistema** para futuros agentes del pipeline lama.

> Regla: el perfil se mantiene VIVO — cada jornada puede descubrir un patrón nuevo; se actualiza acá y se re-guarda en Engram.