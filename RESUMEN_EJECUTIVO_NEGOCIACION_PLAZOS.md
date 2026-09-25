# Resumen ejecutivo — call_me_maybe (Sujeto VIII)

**Pedido de extensión de plazos — Estado real al 25/09/2026**

---

## 1. Qué es el proyecto

Pipeline que convierte texto libre en **function calls con JSON válido** usando un
LLM local (`Qwen3-0.6B`) con decoder por state machine + trie. El cómputo corre en
una CPU de 4 núcleos físicos (i7-7700HQ, 6 vCPU) con un SDK **sin KV-cache** (cada
token cuesta un forward que reprocesa todo el contexto, ~2.6-2.8 s).

## 2. Avance medible (serie histórica de métricas registradas)

| Fecha | Hito | Suite 11 prompts (gen) | Forwards | P2 aislado | P8 aislado | Acc. función | Acc. full (M14) | Tests | KPI <5' |
|---|---|---|---|---|---|---|---|---|---|
| 17-18/09 | Task 4.2 — smoke modelo real | ~308 s **por prompt** | — | — | — | — | — | 164 | proyección **55+' (11x)** |
| 18/09 | **Task 4.3 ORIGINAL** (6 vCPU) | **34.9 min** (35.0 c/carga) | ~800* | — | — | 11/11 (100%) | 10/11 (91%)** | 164 | ✗ ~7x |
| 23/09 | ref pre-refactor (pre-índice Opt1) | 27.6 min (1656 s) | ~638 (58/p) | 113.2 s | 242.5 s | 11/11 (100%) | 9/11 (82%)*** | 164 | ✗ ~5.5x |
| 24/09 | **Opt2 — header estático** + BUG-011/012 (`d6592d0`, `794a470`) | **15.1 min (908 s)** | **314 (28.5/p)** | 48.5 s (-57%) | 125.9 s (-48%) | 11/11 (100%) | 9/11 (82%)*** | 170 | ✗ 3x |
| 25/09 | **Oráculo por estado N1 (T1–T6)** — BUG-013 resuelto | **7.6 min (454 s)** | **133 (12/p)** | **16.8 s (-85%)** | **80.4 s (-67%)** | **11/11 (100%)** | 9/11 (82%)*** | **195** | ✗ ~1.5x |
| 25/09 (2ª) | Oráculo N1 + **governor sin límite de batería** | **6.5 min (389 s)** | 133 (12/p) | — | — | 11/11 (100%) | 9/11 (82%)*** | 195 | ✗ ~1.3x |
| est. | **B′** (autocompletar fn_name, validado) | **~5.5 min** | ~97 | — | — | — | — | — | ~1.1x (techo físico CPU) |

\* forwards del 18/09 estimados (34.9' / ~2.6 s por forward ≈ 800). ** scoring del 18/09 relajaba
equivalentes funcionales (P10 grupo capturador). *** con el scoring actual estricto (variantes exactas),
el refactor mantiene 82% **sin degradar accuracy entre hitos**; los 2 fallos son semánticos del modelo
(regex con grupo de captura, plural NUMBERS/NUMBER), no del decoder.

### Gráfica representativa — evolución del timing de la suite completa (generación, min)

```
18/09   34.9  ██████████████████████████████████▏  Task 4.3 original (acc 91%)
23/09   27.6  ███████████████████████████▏        ref pre-refactor (pre-índice)
24/09   15.1  ███████████████▏                    Opt2 header estático
25/09    7.6  ███████▏                            oráculo por estado N1
25/09²    6.5  ██████▏                            + governor sin límite de batería (-14%)
est.    ~5.5  █████▏                             techo físico CPU (con B′)
KPI <5' │█████│                                    1 char ≈ 1 min; la marca = límite del KPI
```

## 3. Logros técnicos (resumen)

- **-78% de latencia total** desde la corrida original (34.9 → 7.6 min en 7 días de trabajo efectivo),
  sin degradar accuracy ni romper ningún test.
- **Forwards reducidos 2.4x en la última fase** (~638 → 133, estructural a 0); toda la estructura JSON
  del output sale por inyección estática: el modelo solo elige los *values*.
- **195 tests green** (164 baseline, +31), flake8 y mypy limpios en cada hito.
- **3 bugs de raíz resueltos**: BUG-011 (escapes en `name`), BUG-012 (byte-exactness del header),
  BUG-013 (trigger viciado → oráculo por estado con gate derivable y dominios disjuntos).
- **Accuracy función 100%** en los 11 prompts desde la primera corrida; el constrained decoder + prompt
  siempre eligen la función correcta.

## 4. Por qué se pide la extensión

- **Límite físico del KPI en esta CPU**: con la estructura a 0 forwards, el residuo son los ~96 *values
  libres* que el modelo decide. El techo honesto de la optimización estructural es **~5.5 min** (<5'
  inalcanzable sin KV-cache — fuera del subject — o mejor hardware). Con el governor sin limitación
  de batería la suite baja a **6.5 min (~1.3x del KPI)** — empezamos a 7x — y B′ (validado) es la última
  palanca restante.
- **Trabajo restante**: tasks 5.1–5.3 (validator + persistencia de output + formato) y 6.x (pipeline
  non-interactive + reports) están **sin empezar**; la decisión de scoring M14 (relajar vs exigir) sigue
  abierta.
