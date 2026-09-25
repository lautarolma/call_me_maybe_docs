# Resumen ejecutivo — call_me_maybe (Sujeto VIII)

**Pedido de extensión de plazos — Estado real al 25/09/2026**

---

## 1. Qué es el proyecto

Pipeline que convierte texto libre en **function calls con JSON válido** usando un
LLM local (`Qwen3-0.6B`) con decoder por state machine + trie. El cómputo corre en
una CPU de 4 núcleos (i7-7700HQ) con un SDK **sin KV-cache** (cada token cuesta un
forward que reprocesa todo el contexto, ~2.6 s).

## 2. Avance medible (cuadro de métricas por fecha)

| Fecha (aprox.) | Hito | Suite 11 prompts | Forwards | P2 aislado | P8 aislado | Acc. función | Acc. full (M14) | Tests green | KPI <5' |
|---|---|---|---|---|---|---|---|---|---|
| 23/09 | Baseline original (Task 4.3) | ~24 min | ~340* | 113.2 s | 242.5 s | — | 91%** | 164 | ✗ (4.8x) |
| 24/09 | Opt2 — header estático + BUG-011/012 (`d6592d0`, `794a470`) | **15.1 min (-45%)** | 314 | 48.5 s (-57%) | 125.9 s (-48%) | 100% | 82% | 170 | ✗ (3x) |
| 25/09 madrugada | 2º tramo lineal (tails) → **BUG-013 detectado** (trigger viciado) | *corrida corrupta* | — | *inválido* | 56→49 fwd | — | — | 179 | — |
| 25/09 noche | **Oráculo por estado N1 (T1–T6)** — BUG-013 resuelto | **7.6 min (-50%)** | **133** | **16.8 s (-85%)** | **80.4 s (-67%)** | **100%** | **82%** | **195** | ✗ (~1.5x) |

\* forwards del baseline estimados por proporción de timing. ** scoring previo al endurecimiento de variantes regex (P9/P10); con el scoring actual el refactor mantiene 82% **sin degradar** accuracy entre hitos.

### Gráfica representativa — evolución del timing de la suite completa (min)

```
23/09   24.0  ████████████████████████▏  baseline original
24/09   15.1  ███████████████▏           Opt2 header estático   (-45%)
25/09    7.6  ███████▏                   oráculo por estado N1   (-50% sobre Opt2, -68% total)
est.    ~5.5  █████▏                     techo físico CPU (con B′), KPI <5' a ~1.1x
```

## 3. Logros técnicos (resumen)

- **-68% de latencia total** de la suite en 48 h de trabajo efectivo (24 → 7.6 min), sin degradar accuracy ni romper ningún test.
- **Forwards reducidos 2.4x** (314 → 133); toda la **estructura JSON del output a 0 forwards** (IN_OBJECT, PARAMS_OBJECT, VALUE_END): el modelo solo elige los *values*.
- **195 tests green** (164 baseline, +31), flake8 y mypy limpios en cada hito.
- **3 bugs de raíz resueltos**: BUG-011 (escapes en `name`), BUG-012 (byte-exactness del header), BUG-013 (trigger viciado → oráculo por estado, con gate derivable y dominios disjuntos).
- **Accuracy función 100%** en los 11 prompts; los 2 fallos de M14 son **semánticos de scoring** (regex con grupo de captura funcionalmente equivalentes), no del decoder.

## 4. Por qué se pide la extensión

- **Límite físico del KPI en esta CPU**: con la estructura a 0 forwards, el residuo son los ~96 *values libres* que el modelo decide. El techo honesto de la optimización estructural es **~5.5 min** (<5' inalcanzable sin KV-cache — fuera del subject — o mejor hardware). La suite actual ya está a **~1.5x del KPI**, con B′ (validado) como última palanca.
- **Trabajo restante**: tasks 5.1–5.3 (validator + persistencia de output + formato) y 6.x (pipeline non-interactive + reports) están **sin empezar**; la decisión de scoring M14 (relajar vs exigir) sigue abierta.
