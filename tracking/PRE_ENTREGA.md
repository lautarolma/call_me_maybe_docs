# PRE-ENTREGA — Estrategia de preparación para la defensa

> **Creado**: 2026-10-06. Fuente de verdad de la estrategia de entrega y del
> orden de ataque. Vive en `docs/` (local, fuera del repo principal).
> Compañeros: `PENDIENTES_ENTREGA.md` (lista de trabajo) y
> `ESTADO_ACTUAL.md` (estado medido).

---

## 1. Dirección del repo

- **Repo principal = ENTREGA PURA**: `src/`, `tests/`, `data/input/`,
  `README.md`, `uv.lock`, `.flake8`, `pyproject.toml`, `.gitignore`.
- **FUERA del repo principal** (gitignored, no auditados en este ciclo):
  `docs/`, `moulinette/`, `.opencode/`, `CLAUDE.md` (eliminado 06/10),
  `.scratch_task42/`.
- `docs/` era submodule → convertido a **directorio local standalone**
  (06/10). Material de preparación privado: tracking, diseño, notas.
- **Fuente de verdad de estado**: `docs/tracking/ESTADO_ACTUAL.md`.

## 2. Decisiones de diseño registradas (no se reabren sin dato nuevo)

1. **Pydantic = frontera de I/O.** Validación pura de entrada (`loader/`)
   y salida (`FunctionCall` en `validator/`). El inner loop del decoder usa
   `@dataclass(slots=True)` por costo de instanciación (~200μs vs ~30ns).
   Todo lo que entra y sale del sistema pasa por pydantic; no hace falta
   integrarlo paso a paso en la estructura del decoder — sería imposible
   sin romper el ratio <5'. **No es pendiente: es decisión de arquitectura.**
2. **Reglas A/B/C post-hoc** en `src/validator/output_validator.py`:
   familia unificada (truncado · comillas internas · conteo de corridas).
   11/11 público + 11/11 privado. Costo: 0,4 ms por suite.
3. **Latencia local fuera de KPI por hardware** (~325 s vs 300 s el 05/10).
   ✅ **Resuelto 09/10 en campus**: caja real (i5-8500, CPU-only, cache HF
   frío) → **2'43" (163 s)**, 137 forwards, **PASS** con 137 s de margen.
4. **`echo_view` = higiene de consola** (pantalla = disco post-validación).
   No es feature ni bonus. El entregable es el JSON de `data/output/`.
5. **B7 visualización de generación: DIFERIDA.** Foundation existente:
   infraestructura de métricas (`MetricsRun`, `DecoderPhase`),
   `report_prompt_metrics()` (tabla per-prompt), tablas de score de los
   smokes. **Gap**: capa step-by-step token a token con color
   (verde=allowed · rojo=blocked · amarillo=estado), gate `isatty()` +
   `NO_COLOR`. Diseño en `PLAN_DIDACTICO.md` M13 B7.

## 3. Auditoría (3 puntos, en orden)

1. **Frontera de entrega**: qué hay en el repo principal que no es
   requisito del subject → gitignore/borrar. NO se audita `docs/` ni
   `moulinette/` (decisión 06/10).
2. **Review superficial por módulo** (durante el recorrido): dead code,
   duplicidades, nombres poco descriptivos, complejidad evitable.
   Se propone el cambio; si está bien, se sigue.
3. **Síntesis didáctica**: sintetizar docstrings. Bugs / decisiones de
   diseño / refactorización solo se contrastan con `docs/` para detectar
   obsolescencia. **Ante duda: se pregunta, no se hace deep-dive.**

## 4. Recorrido teórico (orden de defensa)

```
loader/ → models/ → prompt/ → decoder/ (state → trie → schema_validator
→ token_filter → constrained_generator) → validator/ → utils/metrics
→ pipeline/ → __main__/
```

Por módulo, en cada pase:

1. **Análisis de flujo con los docstrings VIEJOS** → material de defensa
   (qué enseñaban, qué datos didácticos preservar).
2. **Docstring NUEVO acordado** (norma §5).
3. **Review superficial + contraste con docs** (auditoría puntos 2 y 3).

Se re-lee el módulo con los docstrings nuevos → lectura oficial de entrega.

## 5. Formato de docstrings (norma única)

- **Inglés** en TODO: código, docstrings, comentarios visibles. README ya
  está en inglés. (Reversa de la decisión anterior de "español es estilo
  de la casa" — requerimiento explícito del usuario 06/10.)
- **Google-style**: primera línea en imperativo, indentada, seguida de
  detalle, `Args:`, `Returns:`, `Raises:`, `Attributes:` cuando aplica.
- Cierre de triple comillas **indentado**, alineado con el cuerpo.
- **CERO referencias** a `docs/`, historial interno o conversaciones.
- Prosa extra solo si es **indispensable** para entender el código.
- Type hints estrictos en firmas; el docstring no repite el tipo si el
  símbolo ya lo dice.

## 6. Commits planificados (OK del usuario dado 06/10)

| # | Commit | Contenido |
|---|---|---|
| 1 | `fix(models)` | `echo_view` post-validación (3 archivos, ya en verde: 272 tests) |
| 2 | `chore(repo)` | Frontera: `docs/` fuera del track, `.gitignore`, `.gitmodules` borrado |
| 3 | `chore(repo)` | `CLAUDE.md` eliminado (nunca estuvo trackeado; contenido migrado) |
| 4 | docs locales | `ESTADO_ACTUAL` + `PENDIENTES_ENTREGA` + `PRE_ENTREGA` — commit en el repo local de `docs/` |
| 5+ | `refactor(docs)` | Docstrings ENGLISH Google-style, tandas por módulo, con OK por tanda |

## 7. Reglas que no se rompen

- NO librerías fuera del subject. NO commits sin OK. NO tocar stashes.
- `data/output/` git-ignored. `data/correction/` jamás se versiona.
- `uv.lock` SÍ se versiona (el subject lo pide).
- Nunca medir latencia con `opencode` vivo (contamina 2,6x).
- Métricas e instrumentación **no se borran** (bonus B7 + evidencia).
