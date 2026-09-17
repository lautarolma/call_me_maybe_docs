EN el caso de las variables PyObj con o sin Slots (__slot__) hemos vistas que estas fuerzan la generacion de una variable/objeto de python lo mas afin a una variable sencilla y simplificada, sin tantos atributos especiales propios de las variables objeto. Lo cual les permite una velocidad de acceso a sus atributos mejorada y optimizacion de uso de memoria.
 **EL caso de las verificaciones segun la naturaleza del dato** 

    Para datos internos: DataClass. Para generar la variable que lo contenga, pues sera un __slot__ data con atributos optimizados, pues no requiere de validacion al estar preestablecido por el sistema sin intervencion de terceros externos.
    Para datos externos (I/O): Pydantic. Ha de pasar por verificacion previa, pues, provenientes de una APIs, un JSON, o imput d eusuario

---

## NOTAS REVISION CARGADORES + MODELOS (capa de I/O pydantic)

### Tipos de parametro: Literal y los objetos
`ParameterType = Literal["string", "number", "boolean", "null"]` son los 4 tipos
ESCALARES de JSON. Los COMPUESTOS (objects/dicts y arrays) se excluyen del MVP
de forma DELIBERADA: validar un parametro anidado pide maquina de estados +
schema recursivo (muy mas complejo). Para habilitar el BONUS B8 (nested function
arguments) habria que agregar "object" y "array" al Literal y extender el
schema_validator. Es una frontera de complejidad intencional, no un olvido.

### CODEPENDENCIA de ParameterDef y el `name` (el detalle que mas cuesta)
El JSON de entrada NO trae el nombre del parametro DENTRO del objeto; el nombre
es la KEY del dict padre:
    "parameters": { "a": {"type": "number"} }   <- la key "a" ES el nombre
Cuando pydantic valida `parameters: dict[str, ParameterDef]`, intenta construir
un ParameterDef por cada valor a partir de `{"type": "number"}` — pero NO viene
`name`. Por eso `name: str = Field(default="", ...)`:
  - SIN default="" -> pydantic FALLA en la construccion ("falta name"), y como
    falla ANTES de que corra cualquier validator, nunca se llenaria.
  - CON default="" -> pydantic construye el ParameterDef con name="" (placeholder
    temporal), y recien DESPUES corre el validator mode="after" que pisa name con
    la key correcta:  param.name = key.
Secuencia:  construccion (placeholder "") -> validator after (llena name) -> OK.
El default="" es un ANDAMIO mecanico, no un valor de negocio: nunca ves name=""
final porque el validator siempre lo pisa. La "codependencia" es: el campo name
necesita existir (con default) para que pydantic construya el objeto, y el objeto
construido es lo que el validator necesita para sincronizarlo.

### Donde vive el validator y como se usa
El validator es un METODO DENTRO de la clase (no una cosa suelta):
    @field_validator("parameters", mode="after")
    @classmethod
    def _sync_parameter_names(cls, params): ...
Decoradores:
  - @field_validator(...) -> de pydantic, declara "esto valida el campo X"
  - @classmethod -> OBLIGATORIO en pydantic v2 (corre sobre la clase, antes de la instancia)
Modos:
  - mode="before" -> corre ANTES de validar, recibe el dato CRUDO (pre-procesar)
  - mode="after"  -> corre DESPUES de validar, recibe el dato YA tipado (mutar/ajustar)
Si podes usar estos decoradores en el proyecto: SI, pydantic ya es dependencia.

### Otros detalles de la revision
- FunctionCall.parameters usa `dict[str, JSONValue]` con default_factory=dict
  (NO `dict = {}`) para evitar el mutable default compartido entre instancias.
- JSONValue = str | int | float | bool | None: ojo que bool es subclase de int;
  un validador que distinga True de 1 debe chequear bool ANTES que int.
- loaders traducen TODO a ValueError (fail fast + exception chaining `from exc`)
  para que el caller solo atrape una excepcion.

---

## OPTIMIZACIÓN: ORDEN DE CONDICIONES Y CORTOCIRCUITO (del recorrido: src/loader/input_loader.py)

> La lección de la línea 45 de input_loader.py: `not isinstance(data, list) or len(data) == 0`.
> El ORDEN en que escribís las condiciones NO es cosmético — es optimización Y correctitud.

### El mecanismo: short-circuit (cortocircuito) / evaluación perezosa
Python evalúa booleanos con `and`/`or` de forma perezosa, de izquierda a derecha:
- `A or B`  → evalúa A. Si A es truthy → **NUNCA evalúa B** (ya decidió).
- `A and B` → evalúa A. Si A es falsy → **NUNCA evalúa B** (ya decidió).
Si B no se evalúa, **no paga el costo de B**. Punto. Esa es toda la jugada.

### La regla de oro: Barato Primero, Costoso Después
Poné PRIMERO la condición más barata Y la que más frecuentemente corte.
Jerarquía de costos (de barato a caro, órdenes de magnitud):

| Costo | Operación | Nota |
|---|---|---|
| ~ns (CPU pura) | `isinstance`, `is None`, comparaciones simples | no toca memoria externa |
| ~ns | lookup en dict / acceso a atributo | O(1), hash table |
| O(1) u O(n) | `len()` | O(1) en list/str/dict; NO existe en generadores/iteradores (TypeError) |
| ~ns-µs | iteración / construcción de estructuras | O(n) sobre los datos |
| **µs-ms** | **syscalls / I/O** (open, read, os.path.exists, network, procesos) | context switch, disco/red: LO MÁS CARO por lejos |

Un `isinstance` no es solo "más barato que un len": es que **no requiere que el objeto
tenga nada** — un len sobre algo sin `__len__` se cae con TypeError. El orden correcto
PROTEGE la operación de después.

### Dos categorías (el matiz importante)
1. **CORTOCIRCUITO POR CORRECCIÓN (guard / guarding)**: la 2ª condición SOLO tiene
   sentido si la 1ª vale. No es optimización: sin el orden, CRASH.
   `isinstance(x, list) and x[0]` — x[0] rompería si x no es lista.
   `data and len(data)` — len rompería si data no tiene __len__.
   Aplicado en el proyecto: `not isinstance(data, list) or len(data) == 0`.
   Si isinstance falla → `len(data)` NUNCA se ejecuta → no TypeError sobre un dict.
2. **CORTOCIRCUITO POR RENDIMIENTO**: ambas condiciones valen solas, pero ponés
   primero la más barata y la que más corta → en el caso común pagás solo UNA.
   `user is not None and user.is_admin and fetch_permisos(user)` — la syscall de
   fetch_permisos solo corre si el usuario existe Y es admin. Orden invertido =
   I/O innecesaria AUNQUE el user sea None.

Regla práctica: **las condiciones que NO acceden a I/O y que cortan seguido, VAN
PRIMERO.** El caso de rendimiento clásico: validar formato del input en memoria ANTES
de tocar disco (`or` para fallar temprano, `and` para avanzar solo si todo va bien).

### El gotcha fino: and/or NO devuelven bool, devuelven el OPERANDO
`[] or "default"`  → `"default"` (el 1er falsy decide → devuelve el 2º)
`"x" and 42`       → `42` (el 1er truthy queda → devuelve el 2º)
`0 or "" or "ok"`  → `"ok"` (el último operando evaluado gana)
Si necesitás un bool de verdad: `bool(expresion)`.

### Analogía de obra
Es el encargado que revisa que el PLANO EXISTA (isinstance, gratis) antes de mandar a
traer el MATERIAL (len/I/O, caro). Si el plano no existe, ni llamás al camión. Y si el
plano está, recién ahí contás cuántos ladrillos hay. Nunca al revés.

---

## ARGPARSE (del recorrido: src/cli.py)

### La idea central: es un REGISTRO declarativo
No le programás "cómo parsear": le DECLARÁS los flags que aceptás y él se encarga
de leer, transformar, validar la forma y generar la ayuda. Tres pasos:
  1. Crear el parser (ArgumentParser con description)
  2. Declarar flags con add_argument (nombre, type, default, help)
  3. parse_args(argv) -> Namespace (objeto-bolsa con acceso por atributo: args.input)

### -h / --help: flags ESPECIALES que argparse crea solo
No hay que declararlos. Cuando el parser los detecta:
  - imprime el texto de ayuda -> TERMINA con exit code 0 -> NO entra a la lógica.
  - EL APRENDIZAJE CLAVE: la ayuda NO es "un modo de ejecución", es "un modo de
    SALIDA del parser". El parse_args es lo ÚLTIMO que corre antes de devolver
    el control; por eso la ayuda corta ahí mismo, sin llegar a run().
  - El texto se genera AUTOMÁTICAMENTE desde description + el help de cada
    add_argument. No escribís ni una línea de ayuda manual.

### Cómo lee los flags
  - parse_args(None) lee sys.argv[1:] (argv[0] = nombre del programa, se ignora).
  - Empareja cada "--flag valor" y mapea --functions_definition -> atributo
    functions_definition (los guiones medios se convierten en _ si los usaras).
  - type=Path NO es tipado: es un CALLABLE de transformación. Internamente hace
    Path(valor_del_string_crudo). Si el callable falla, argparse aborta solo.

### Flags opcionales vs requeridas (lo que no aparece en cli.py pero hay que saber)
  - POR DEFECTO TODAS SON OPCIONALES: si el flag no está en argv, usa el default.
  - Requerida se declara con required=True -> si falta, argparse imprime error +
    usage en stderr y hace sys.exit(2) (aborta SOLO, nunca llega a tu código).
  - Flag DESCONOCIDO: también error + usage en stderr + exit 2.
  - Diferencia CRÍTICA de exit codes (esto hay que sabérselo):
      --help/-h        -> ayuda en STDOUT + exit 0 (éxito)
      error de CLI     -> mensaje en STDERR + exit 2 (usage error)
      error de runtime -> exit 1 (nuestro código, ver __main__.py)
  O sea: 0 = ayuda/éxito, 1 = fallo del programa, 2 = fallo del USO de la CLI.
  Por eso cli.py no valida NADA manualmente: argparse ya cubre la FORMA.

### Qué NO valida argparse (y no debe)
  - NO valida que el ARCHIVO exista ni que el contenido sea válido. Eso es
    trabajo de los loaders (Phase 1) y del pipeline (Phase 5). argparse solo
    valida la FORMA de la invocación. Separación de responsabilidades: el CLI
    define la interfaz; la lógica valida los datos.

---

## MODELO ML: ENTRENAMIENTO vs INFERENCIA (del recorrido: src/pipeline.py)

> Síntesis de lo que NO sabía y de mis confusiones al ver `Small_LLM_Model()`,
> `.eval()`, `dropout`, `requires_grad` y `batchNorm`. Todo ML, nada de código.

### La analogía que lo unifica todo: la fábrica con MILLONES de perillas
- El modelo (Qwen3) es una fábrica: entra texto → sale texto.
- Adentro hay millones de **perillas ajustables**, cada una con un número.
- **El número de cada perilla = un PESO (weight).**
- La combinación de TODAS las perillas = lo que el modelo "sabe".
  Cambiás las perillas → el modelo responde distinto.

### ENTRENAMIENTO = ajustar las perillas (aprender)
1. Le mostrás millones de ejemplos (input + respuesta correcta).
2. Él responde, comparás con la correcta y calculás el **ERROR** (qué tan lejos estuvo).
3. Los **GRADIENTES** son la "brújula": te dicen hacia dónde girar cada perilla
   para que el error baje.
4. Girás según la brújula, repetís miles de veces → el error baja → aprendió.

👉 **CONFUSIÓN ACLARADA (la duda central):** el entrenamiento SÍ produce un
**cambio PERMANENTE en los pesos** (quedan grabados en disco, archivo safetensors).
Ese es el objetivo del entrenamiento.

### INFERENCIA = usar la fábrica ya ajustada (sin tocar nada)
- El modelo **YA viene entrenado** (perillas fijas).
- Solo pasás texto → recibís output. NO ajustás ninguna perilla.
- Cada pasada input→output se llama **forward** ("hacia adelante", una sola dirección).

### ¿Qué es `requires_grad`? (MI MAYOR CONFUSIÓN)
> **`requires_grad` NO entrena. Solo decide si se RASTREA la info necesaria para
> (potencialmente) entrenar.**

- `requires_grad=True` → PyTorch guarda la "brújula" (gradientes) en cada forward.
  Solo sirve si vas a entrenar.
- `requires_grad=False` → NO guarda la brújula. Punto.

**Confusión que tuve:** creí que "gradientes → afina la inferencia" y que aumentaba
la capacidad de inferencia a costo de memoria/velocidad, y que podía cambiar el modelo
de forma permanente. **TODO ESO ES FALSO:**
- El entrenamiento NO se hace en tiempo de ejecución → se hace ANTES, offline, para
  producir los pesos. La inferencia solo los USA.
- En NUESTRO proyecto NO entrenamos NADA (Qwen3 ya viene entrenado). Solo inferencia.
- `requires_grad=False` no mejora NI empeora la calidad → solo hace la inferencia
  MÁS BARATA (menos memoria, más rápida), porque no rastrea gradientes.
- **NO hay cambio permanente**: como no tocás las perillas, el modelo queda igual
  después de correrlo. `requires_grad` es una config de tiempo de ejecución, no cambia
  el modelo en disco.

**En una línea:** el entrenamiento (con gradientes) cambia el modelo permanentemente;
`requires_grad=False` solo te ahorra el costo de rastrear gradientes que NO vas a usar
porque no entrenás.

### CONCEPTOS QUE NO SABÍA (en criollo)
- **Forward:** una pasada input→output por el modelo. Su opuesto (backward /
  backpropagation) usa los gradientes para entrenar → nosotros NUNCA lo hacemos.
- **Dropout:** durante el entrenamiento, "apagar al azar" algunas perillas en cada
  pasada, para que el modelo no dependa de un solo camino. Lo fuerza a aprender
  caminos alternativos → patrones más generales. En inferencia se desactiva porque
  querés la fábrica COMPLETA y determinista (mismo input → mismo output).
- **Overfitting:** NO es "sobreestructuración" (mi intuición, casi). Es: el modelo se
  aprende de MEMORIA los ejemplos de entrenamiento en vez de la REGLA general.
  Analogía: estudiante que memoriza el examen viejo en vez de entender la materia →
  le va bien en lo que vio, mal con preguntas nuevas. Dropout sirve para EVITARLO.
- **BatchNorm:** detalle interno de entrenamiento que normaliza valores de cada capa
  para que no exploten. NO hace falta dominarlo para este proyecto. Solo sabé que su
  comportamiento cambia entre train y eval (por eso `.eval()` lo conmuta con dropout).

### POR QUÉ `.eval()` (qué hace exactamente)
- Por defecto PyTorch arranca en modo `train` (dropout ACTIVO).
- `.eval()` conmuta a modo inferencia: apaga dropout y ajusta batchnorm.
- En inferencia NO querés aleatoriedad → mismo input debe dar siempre mismo output.
  Con dropout activo, cada corrida daría resultado distinto y degradado.

### LO RELEVANTE PARA MI PROYECTO (para no perderme)
- NO voy a entrenar nada en call_me_maybe. El modelo viene entrenado.
- Mi laburo real = **inferencia** + **el DECODER (constrained decoding)** = Phase 3,
  donde está toda la complejidad. El entrenamiento/dropout/gradientes son contexto,
  no mi día a día.
- Con saber "el modelo viene afinado y yo solo lo uso" + "requires_grad=False = no
  voy a entrenar, ahorro costos", tengo lo que necesito.

---

## COUNTER, SET Y FAIL-FAST: COMPLEJIDAD EN LA DETECCIÓN DE DUPLICADOS (del recorrido: src/loader/function_loader.py — BUG-002)

### El problema: duplicados en una lista de nombres
Detectar nombres repetidos es un clásico. La versión naive que teníamos:

    names = [fn.name for fn in functions]
    dupes = [n for n in names if names.count(n) > 1]

`names.count(n)` recorre la lista COMPLETA por cada n → **O(n²)**.
Para 10 funciones es irrelevante; para 100.000 es un desastre.
Regla mental: OJO con `count()` / `index()` / `in` dentro de un loop sobre la MISMA lista.

### Las herramientas de la stdlib (misma idea, distinto sabor)
- **`set`**: estructura de hash table: consultar/insertar es O(1). Un loop con
  `seen.add()` y `n in seen` queda O(n) total. Ideal para "¿ya lo vi?" (cortar en el 1°).
- **`collections.Counter`**: cuenta apariciones en UNA pasada (O(n)) y devuelve
  `{elemento: cantidad}`. Ideal para "cuántas veces aparece cada cosa" — NO corta
  en el primero, porque quiere el conteo completo.
- **`collections.defaultdict(int)`**: el "Counter manual"; útil cuando además
  necesitás acumular OTRA cosa por clave (p.ej. listas de items por nombre).

### La decisión del proyecto (function_loader.py — Opción A, BUG-002)
Un solo loop que construye Y valida (fail-fast real):

    seen: set[str] = set()
    functions: list[FunctionDef] = []
    for item in data:
        fn = FunctionDef(**item)          # pydantic valida el item
        if fn.name in seen:               # O(1) — ¿ya lo vimos?
            raise ValueError(...)         # cortamos en la PRIMERA reincidencia
        seen.add(fn.name)
        functions.append(fn)

Ventajas:
- O(n) total (set = hash table).
- Fail-fast REAL: corta la ejecución INMEDIATAMENTE, sin construir los modelos
  que siguen (la versión vieja construía TODO y recién después contaba).
- El mensaje es el primer repeat (singular), no la lista de todos los duplicados.

### La asimetría detectada (el bug de diseño)
`input_loader.load_prompts` validaba "lista no vacía" (`len(data) == 0 → error`)
pero `function_loader.load_functions` NO: un `functions_definition.json` con `[]`
pasaba en silencio → pipeline con 0 funciones → fallo asegurado contra el corrector
(compara el SET de funciones declaradas vs esperadas).
Lección: **loaders hermanos deben validar LA MISMA forma**; la validación de
"no vacío" es contrato del loader, no un lujo. (La spec de diseño tenía la
asimetría documentada — otro motivo para revisar specs hermanas cuando un defecto
aparece en una capa.)

**En una línea:** si tu detector de duplicados usa `count()` dentro de un loop,
es O(n²); un `set` lo deja O(n) y te da fail-fast gratis.

---

## BYTE-LEVEL BPE, BYTE-TO-UNICODE Y EL ROUNDTRIP IDENTIDAD (del recorrido: src/loader/vocab_loader.py — BUG-003)

> Pendiente de re-explicar con el usuario (2026-09-09, parada 8 en revisión). La duda
> central: ¿por qué el código decodifica con `model.decode` y NO con Python?
> ¿Qué tiene que ver el `Ġ`?

### Los bytes se pintan de unicode para poder "vivir" en un string
- Los tokenizadores byte-level BPE trabajan a nivel BYTE, pero el vocab.json guarda
  STRINGS. Para meter bytes invisibles/no imprimibles (0x00-0xFF) en strings unicode,
  se usa una **byte-to-unicode table**: cada byte 0-255 se mapea a un carácter
  "visible".
- El ESPACIO (byte 0x20) se mapea a `Ġ` (U+0120, una G con macrón). El token del
  texto " the" se guarda como `Ġthe`. Parece una letra rara, pero es UN ESPACIO
  disfrazado.

### El vocab.json guarda tokens con ESA máscara puesta
- `"Ġthe"` NO es "G-the"; es "␣the" (espacio+the) con la máscara puesta.
- El texto REAL no se obtiene mirando el string: hay que aplicar la tabla INVERSA
  (carácter mapeado → byte). Eso es trabajo del TOKENIZER.

### ¿Quién aplica la tabla inversa? EL TOKENIZER (`model.decode`)
- `model.decode([token_id])` → texto REAL: `'Ġthe'` → `' the'`.
- En el SDK interno, decode hace: token → caracteres mapeados → bytes → UTF-8 → str real.
- Por eso el loader necesita el MODELO y no puede "decodificar a mano".

### POR QUÉ `token_text.encode('utf-8').decode('utf-8')` ES UNA TRAMPA (el bug de la spec)
- `s.encode('utf-8')` → bytes; `.decode('utf-8')` → string de nuevo.
- Para TODO string unicode válido es **identidad**: `'Ġthe'` sale `'Ġthe'`.
- NO deshace la tabla → el primer char sería `'Ġ'` (mapeado), no `' '` (real) →
  el índice quedaría agrupado por caracteres-mapeados → el decoder busca
  `tokens_starting_with[" "]` y NO encuentra `Ġthe` → fallo lógico silencioso.
- Bonus: `encode('utf-8')` no falla para strings unicode válidos → el try/except
  de la spec era casi código muerto.

| Operación | `'Ġthe'` (mapeado) | Resultado | ¿Deshace la tabla? |
|---|---|---|---|
| `encode('utf-8')` + `decode('utf-8')` (Python) | → bytes → str | `'Ġthe'` | ❌ identidad |
| `model.decode([token_id])` (tokenizer) | → tabla inversa | `' the'` | ✅ sí |

### Por qué se indexa por el PRIMER carácter DECODIFICADO
- El decoder (Phase 3) filtra candidatos por el primer carácter del texto REAL que
  está generando (`' '`, `'{'`, etc.). Con el índice por caracteres-mapeados,
  `Ġthe` caería en `Ġ` y nunca se encontraría al buscar `' '`.
- Pre-indexación = pago único O(V tokens × D decode); toda consulta después es O(1)
  con `setdefault` + `set`.

### Por qué existen los `<byte>` (BYTE_CATEGORY)
- Tokens especiales (`<|endoftext|>`, `<|im_start|>`, etc.) decodifican a vacío o
  fallan → no tienen primer carácter útil → bucket de estacionamiento.
- Ojo API: `decode([token_id])` recibe LISTA (el SDK lo espera así); un int suelto
  puede crashear.
- ❌ NO son "bytes UTF-8 incompletos" (lo que decía la 1ª versión del didáctico):
  el vocab ya viene en texto mapeado, no con bytes sueltos. Ese modelo mental era
  incorrecto → corregido en BUG-003.

**En una línea:** el vocab.json muestra los tokens con una máscara (byte-to-unicode)
— `Ġ` es un espacio; el texto real SOLO se obtiene decodificando con el tokenizer
(`model.decode`); `encode+decode` de Python es un roundtrip identidad que no sirve.

## SYNTAX vs SEMÁNTICA: QUÉ VALIDA EL STATE MACHINE (del decoder: src/decoder/state.py)

### La confusión de base (la que costó entender)
- El state machine NO valida "si el valor es correcto para el negocio" ni "si la key
  existe ni si está vacía". Valida UNA SOLA cosa: **¿esto sigue siendo JSON sintácticamente
  válido?** Es un validador de GRAMÁTICA sobre un string, no de reglas del dominio.
- Eso significa: `{"": 1}` PASA por la state machine (sintaxis OK). Que el schema
  rechace la key vacía es OTRO nivel (semántica → schema_validator, Task 3.3).

### Los DOS niveles (la distinción que faltaba)
1. **Sintaxis JSON** (esto hace state.py): secuencia de caracteres válida, cierre de
   comillas, usos correctos de `:` `,` `{` `}`, number parcial/completo, escapes
   `\n` `\uXXXX`, literales true/false/null.
2. **Reglas del problema / schema** (NO hace acá → Task 3.3): que la key exista, que no
   esté vacía si el schema lo prohíbe, que los required params estén presentes, que el
   valor tenga el tipo que el negocio espera.

### El caso "key vacía": por qué NO se invalida acá
- `{"": 1}` → ROOT → `{` → OBJECT_OPEN → `"` → KEY_START → `"` → KEY_END → `:` → COLON
  → `1` → IN_NUMBER_VALUE → `}` → COMPLETE. **Válido sintácticamente.**
- Si la key vacía no debe existir, esa regla vive en el schema/validador de negocio,
  que corre DESPUÉS de la state machine.

### KEY_END no "valida el valor de la key"; espera el próximo token
- Después de cerrar una key el parser solo permite: whitespace (opcional) y luego `:`.
- `{"":1}` → válido · `{"":}` → inválido (post-`:` no hay value) · `{"a" 1}` → inválido
  (falta `:`) · `{"":"x"}` → válido.

### El mismo patrón en otros `_step_*`: acepta estructura, no semántica final
- En string: `""` es un string CORRECTO y cierra en VALUE_END — no hay chequeo
  "string no puede estar vacío".
- KEY_START + `"` inmediato → KEY_END (key vacía) es una transición legal.

### Casos edge confirmados (en pocas líneas)
- `{"":1}` → pasa (key vacía es legal en sintaxis).
- `{"":}` → **KEY_END→COLON** acepta `:`, pero luego `}` a la derecha de `:` no es
  value → falla (COLON no acepta `}`).
- `{"a" 1}` → **KEY_END** rechaza: tras la key cerrada solo valen ws y `:`.
- `{"":"x"}` → pasa (string value).
- `""` → pasa (string vacío válido).

**En una línea:** el state machine es el árbitro de SINTAXIS (gramática JSON char por
char) y es INTENCIONALMENTE ciego a la semántica (qué keys existen, tipos esperados,
required keys) — eso es trabajo de schema_validator.py (Task 3.3). La separación
sintaxis ↔ semántica es el diseño: cada capa valida UNA sola cosa.

---

## GRAMMAR NUMÉRICA: LAS DOS REGEX (del decoder: src/decoder/state.py)

### El problema: dos momentos distintos, dos validaciones distintas
- Mientras se ESCRIBE el número necesitás saber si el próximo carácter sigue siendo
  parte del número (validación incremental char por char).
- Al CERRAR el value (`,` / `}` / whitespace) necesitás saber si lo acumulado es un
  número JSON válido COMPLETO.
- Una sola regex no sirve para ambos: la de prefijo debe aceptar estados "en
  construcción" que la estricta debe rechazar (y al revés).

### `_NUMBER_RE` (estricta): decide al cerrar
`-?(0|[1-9][0-9]*)(\.[0-9]+)?([eE][+-]?[0-9]+)?`

Pieza por pieza:
- `-?` → signo menos OPCIONAL, solo al inicio. El `+` no existe como signo de número
  (solo como signo de exponente).
- `0|[1-9][0-9]*` → O un cero SOLO, O un primer dígito 1-9 seguido de cualquier
  cantidad de dígitos → **rechaza leading zeros**: `01`, `-01` FALLAN.
- `(\.[0-9]+)?` → fracción opcional: punto + AL MENOS un dígito → `2.` NO es válido
  al cerrar.
- `([eE][+-]?[0-9]+)?` → exponente opcional: e/E, signo opcional, AL MENOS un dígito
  → `1e`, `1e+` NO cierran.

Uso: `_is_valid_json_number()` — `_NUMBER_RE.fullmatch(number_buffer)` justo antes
de cerrar el value en `_step_number`.

### `_NUMBER_PREFIX_RE` (prefijo): valida mientras se escribe
`-?(0|[1-9][0-9]*)(\.[0-9]+([eE][+-]?[0-9]*)?|\.[0-9]*|[eE][+-]?[0-9]*)?`

La diferencia con la estricta está TODA en el final: los `*` toleran partes
pendientes. `"-"` es un prefijo legal (falta el dígito), `"2."` también (puede venir
el dígito de la fracción), `"2e"` y `"2e+"` también (falta el exponente).

Uso: `_NUMBER_PREFIX_RE.fullmatch(number_buffer + char)` — el carácter candidato solo
se acumula si mantiene el prefijo válido.

### El truco fino (bug real corregido en Task 3.1)
La alternancia decimal está **desanidada a propósito**:
`\.[0-9]+([eE]...)?` primero (fracción con dígitos + exponente opcional), luego
`\.[0-9]*` (fracción pendiente, SIN exponente), luego el exponente.

Resultado: **`2.e` DEBE fallar** — un punto sin dígitos deja la fracción pendiente,
y el exponente solo es legal DESPUÉS de al menos un dígito.

### Tabla de casos clave (verificada contra el código)
| Buffer | ¿Prefijo válido? | ¿Número completo? |
|--------|------------------|-------------------|
| `-` | ✅ sí | ❌ no |
| `0` | ✅ sí | ✅ sí |
| `01` | ❌ no | ❌ no (leading zero) |
| `2.` | ✅ sí | ❌ no (solo dígitos pueden seguir) |
| `2e` | ✅ sí | ❌ no (solo dígitos y `+`/`-`, falta exponente) |
| `2.e` | ❌ no | ❌ no |
| `2.5` | ✅ sí | ✅ sí |
| `2.5e+2` | ✅ sí | ✅ sí |

**En una línea:** dos momentos → dos criterios → dos regex: la estricta mataría
`"2."` y `"2e+"` a mitad de camino (que son prefijos legítimos); la de prefijo dejaría
pasar `"2."` o `"1e+"` al cerrar. Ninguna sola resuelve los dos momentos.

---

## EL CONTRATO function_call (qué valida exactamente el decoder)

### Lo que el prompt le exige al modelo
`"Output ONLY a JSON object with 'name' and 'parameters' fields. No explanation."` —
el decoder valida esa forma canónica.

### Las cláusulas en 3 grupos
1. **Estructura**: UN solo objeto JSON (arranca con `{`); el `}` final del objeto
   raíz detiene la generación (COMPLETE).
2. **Contenido**: key `name` (string) + key `parameters` (objeto); keys libres
   (cualquier carácter, sin escapes en keys); strings con escapes
   `\" \\ \/ \n \t \r \b \f` y `\uXXXX` (4 dígitos hex); values de 4 tipos:
   string / number / boolean / null.
3. **Límites**: UN nivel de anidamiento (depth ≤ 1) — adentro de `parameters` no hay
   objetos anidados; los literales `true` / `false` / `null` son exactos.

**En una línea:** el decoder valida la FORMA (cláusulas del contrato); el trie
(Task 3.2) valida que `name` sea una función existente, y el schema validator
(Task 3.3) que las keys de `parameters` y sus tipos correspondan a la función elegida.

---

## LOS 16 ESTADOS EN 4 GRUPOS + CÓMO VIAJA EL CONTEXTO (del decoder: src/decoder/state.py)

### La agrupación por rol (la misma del match/case)
- **Apertura**: `ROOT` (espera `{`).
- **Estructura de objeto**: `OBJECT_OPEN` / `IN_OBJECT` / `PARAMS_OBJECT`.
- **Keys**: `KEY_START` / `IN_KEY` / `KEY_END`.
- **Values**: `COLON` (arranque del value) + `IN_STRING_VALUE` / `IN_NUMBER_VALUE` /
  `IN_BOOL_VALUE` / `IN_NULL_VALUE` / `ESCAPE_IN_STRING`.
- **Cierre**: `VALUE_END` / `COMPLETE`.

### Estados de espera vs acumulación
- **Espera** (KEY_END, COLON, VALUE_END): solo aceptan whitespace (no consumidor) +
  el token siguiente exacto (`:`, el arranque del value, o `,`/`}`). Son los "semáforos"
  entre piezas.
- **Acumulación** (IN_KEY, IN_STRING, IN_NUMBER): aceptan cualquier carácter que siga
  la gramática y lo acumulan en un buffer.
- `VALUE_START` existe en el enum pero es **INALCANZABLE**: el plan A6.3 lo reservaba
  como paso intermedio de COLON; la implementación saltea directo al estado de value
  concreto.
- `COMPLETE` tolera whitespace extra (un modelo puede emitir una nueva línea después
  del `}` final sin romper nada).

### Por qué el dispatcher es `match/case` por rol y no if/elif monolítico
1. **Limpieza de lectura**: cada rol de dominio tiene UNA función pequeña y enfocada
   (handler) — en vez de una pared de if/elif mezclando keys, strings y números.
2. **Independencia de testeo**: se instancia un `DecoderState` en cualquier fase y se
   prueba SOLO la transición que interesa (ej: `_step_number` con buffers
   determinados) sin recorrer todo el árbol de transiciones.
3. **Costo honesto**: `match` sobre enum compila a comparaciones secuenciales en
   CPython (NO es un jump table O(1)). El overhead por delegar a handlers ronda
   ~100-200ns por carácter — irrelevante contra el bottleneck real: el regex de
   number cuesta ~1-5μs por carácter.
4. **Alternativas descartadas**: patrón GoF State (una clase por estado) — terrible
   rendimiento en Python (~200-300ns extra por llamada) y sobre-ingeniería para 16
   estados finitos; tabla DFA — JSON no es un lenguaje regular (necesita contexto y
   acciones por transición), las lambdas por celda agregan más overhead que el match.

### Los campos: cómo viaja el contexto
| Campo | Rol |
|-------|-----|
| `phase` | La fase actual de la máquina |
| `current_key` | Key cuyo value se está leyendo (acumulador) |
| `keys_enclosed` | Keys YA cerradas, SOLO las de `parameters` (depth 1) |
| `depth` | 0 = output object, 1 = parameters |
| `number_buffer` / `bool_buffer` | Acumulan el number/literal en curso |
| `unicode_remaining` | Dígitos hex pendientes de un `\uXXXX` en curso |

### Por qué `number_buffer: str` y no booleanos (desvío documentado del plan)
Los flags `number_has_digit` / `number_has_dot` del plan original no alcanzan:
`"01"` (leading zero) y `"1"` tienen `has_digit = True`; `"2."` y `"2.5"` tienen
`has_dot = True`. Se necesita la cadena acumulada completa para distinguir.

### La regla crítica de `keys_enclosed`
NUNCA se muta in-place; siempre se reemplaza (`set | {...}`). Si se mutara el set
compartido, el shallow copy de `simulate()` contaminaría al original y viceversa —
la regla es la que hace seguro el copy barato.

**En una línea:** la máquina son 5 roles de dominio agrupados en handlers (match/case),
16 estados donde 3 son semáforos de espera, y el contexto viaja en campos con buffers
de acumulación — con `keys_enclosed` inmutable por diseño para que copiar el estado
cueste ~50ns.

---

## LA API: EXPLORAR vs COMMITEAR + LA LLAVE DEL FILTER (del decoder: src/decoder/state.py)

### `simulate(text)` → `(bool, state)` — EXPLORA sin tocar el estado real
- Trabaja sobre `copy(self)` — shallow copy barata (~50ns), segura gracias a la regla
  de `keys_enclosed`.
- Recorre los caracteres del token UNO POR UNO; si CUALQUIER carácter falla →
  `(False, self)`: el MISMO objeto original (aliasing deliberado y seguro — el
  generator solo usa el 2° elemento cuando el 1° es `True`).
- Si todo pasó → `(True, estado_resultante)`: el generator usa ese estado directo,
  sin re-simular el token.

### `update_from_text(text)` → `bool` — AVANZA el estado real (muta), atómico
- Igual que simulate (copiar + avanzar char por char), pero al final COMMITEA los
  campos de la copia en `self` uno por uno (con slots no hay `__dict__.update`;
  además es mypy-friendly).
- Si algo falla a mitad de camino, `self` queda EXACTAMENTE como estaba — el generator
  nunca se queda con un estado a medio token.
- Desvío del plan: retorna `bool` (el plan decía `-> None`).

### `expected_first_chars()` → `set[str]` — la llave de Fase 1 del filter
- Devuelve los caracteres con los que PUEDE arrancar el próximo token en el estado
  actual.
- `"*"` es comodín: cuando keys y strings son libres significa "cualquier carácter
  real es posible" → el filter interpreta el comodín como "saltarse el pre-filtro".
- Los tokens del bucket `<byte>` NUNCA matchean caracteres concretos → quedan fuera
  en Fase 1; cuando hay `"*"` los decide la Fase 2 (simulate).
- Caso fino: con `unicode_remaining > 0` devuelve SOLO el set de hex — el pre-filtro
  solo deja pasar tokens que arrancan con un dígito hex.

**En una línea:** tres métodos, tres roles distintos: `simulate` explora barato (sin
efectos), `update_from_text` commitea atómico (o no commitea nada), y
`expected_first_chars` es la llave que el filter usa para no simular los ~151K tokens.

---

## PARA CONTEXTO: LAS 3 FASES DEL FILTER + EL LOOP (Tasks 3.4 y Phase 4)

### Las 3 fases de `compute_allowed_ids`
1. **Fase 1 — pre-filtro por primer carácter**: `expected_first_chars()` del estado
   → junta los buckets `vocab.tokens_starting_with` de cada char (con `"*"` se
   saltea). Reduce ~151K tokens a típicamente 2K-15K candidatos.
2. **Fase 2 — simulación char-by-char**: por cada candidato, `state.simulate(...)`.
   Es candidato real solo si TODOS sus caracteres mantienen JSON válido.
3. **Fase 3 — post-filtro de schema**: trie para keys parciales de "name", keys de
   parameters vía schema, tipos de values esperados, y bloqueo de `}` de parameters
   hasta `all_required_present()`.

### La regla de oro (un token es allowed SOLO SI...)
1. Todos sus caracteres mantienen JSON válido (Fase 2).
2. En KEY_START/IN_KEY el key parcial es prefijo de una key conocida.
3. En IN_STRING_VALUE de "name" el parcial es prefijo de un nombre de función válido.
4. En número cumple la gramática (las dos regex).
5. En string no hay restricción adicional.
6. Al cerrar `}` de parameters TODOS los required están presentes.

### El loop de generación (Phase 4 — Task 4.1)
```
logits → allowed_ids → si vacío: warning + repair → argmax sobre allowed → append
→ token_text → state.update_from_text → schema.update → si COMPLETE: break
→ si se agotó el límite de steps: break (safety)
```
- **NO usar EOS**: los modelos chicos lo emiten prematuramente; el criterio de fin ES
  `state.phase == COMPLETE`.
- `MAX_TOKENS = 200` como safety net: el output esperado es ~30-60 tokens; si a los
  200 no se completó, el output probablemente está roto → cortar antes que generar
  infinito (detalle completo en la sesión del 15 sept).

**En una línea:** el filter es un embudo de 3 niveles (primer char → simulación →
schema) y el loop termina SOLO por COMPLETE o por safety — nunca por EOS.

---

## ESQUEMAS DEL FILTER: NIVEL_1, NIVEL_2 (MAPA 3 BANDAS), CLÁUSULAS Y GAPS (Task 3.4)

### Cómo leer esta sección
- La sección anterior explica las 3 fases con texto; acá están los ESQUEMAS
  (los mapas) para verlo todo de una.
- NIVEL_1 = el pipeline completo (`compute_allowed_ids`, token_filter.py).
- NIVEL_2 = el MAPA de la state machine en 3 bandas, con la ubicación exacta
  de las 4 cláusulas del schema (el que hay que saber leer).
- Después: los árboles de decisión de cada cláusula (C1-C4), un token
  multi-fase en cámara lenta (D) y los gaps (E).

---

## NIVEL_1 · PIPELINE: EL EMBUDO DE 3 FASES (token_filter.py)

```
              VOCABULARIO COMPLETO (vocab.tokens_starting_with)
                     ~151K tokens BPE
                            │
                            ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ FASE 1 · PRE-FILTRO por PRIMER carácter                     │
   │ estado.expected_first_chars() → buckets por char inicial    │
   │ ("*" = comodín → se saltea: keys/strings libres)            │
   │ REDUCE: ~151K → ~2K-15K candidatos                          │
   └──────────────────────────────────────────────────────────────┘
                            │
                            ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ FASE 2 · SIMULACIÓN char-por-char (state.simulate)          │
   │ ¿TODO el token mantiene JSON válido? (sintaxis pura)        │
   │ Trabaja sobre copia barata (~50ns), NO toca el estado real  │
   └──────────────────────────────────────────────────────────────┘
                            │
                            ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ FASE 3 · POST-FILTRO de schema (allows_token, PURA)         │
   │ 4 cláusulas ANDed — cada una con su TRIGGER (ver NIVEL_2)   │
   │   1. name → trie     2. param key → schema                  │
   │   3. value type      4. params close (required)             │
   └──────────────────────────────────────────────────────────────┘
                            │
                            ▼
                        ALLOWED_IDS
            (vacío → warning + repair, en Task 4.1)
```

**En una línea:** cada fase deja pasar MENOS tokens; la Fase 2 solo prueba
sintaxis (barata, por candidato), la Fase 3 exige semántica (schema) y es
PURA: no muta nada, SOLO decide True/False — el estado real se actualiza
después, una sola vez, con el token elegido.

---

## NIVEL_2 · MAPA DE LA STATE MACHINE EN 3 BANDAS (el que hay que saber leer)

### Las 3 bandas (roles de la máquina)
| Banda | Rol | Estados |
|-------|-----|---------|
| **1 · ESTRUCTURA** | El esqueleto: llaves, comillas de key, dos puntos, comas. Arma la FORMA del JSON. | ROOT, OBJECT_OPEN, KEY_START, IN_KEY, IN_OBJECT, PARAMS_OBJECT, COMPLETE |
| **2 · VALUES** | El contenido: strings, numbers, booleans, null. Rellena los values. | IN_STRING_VALUE, IN_NUMBER_VALUE, IN_BOOL_VALUE, IN_NULL_VALUE, ESCAPE_IN_STRING |
| **3 · RECONEXIÓN** | Los SEMÁFOROS: no acumulan, esperan UN token exacto y deciden hacia dónde sigue el flujo. En el mapa están dibujados EN la banda donde trabajan: KEY_END (espera `:`) y VALUE_END (espera `,` o `}`) reconectan la 1; COLON (espera el arranque del value) baja de la 1 a la 2. | KEY_END, COLON, VALUE_END |

### El mapa (estados REALES de state.py)
```
┌────────────────── BANDA 1 · ESTRUCTURA (esqueleto) ──────────────────┐
│                                                                        │
│    ROOT ──'{'──▶ OBJECT_OPEN ──'"'──▶ KEY_START ──?──▶ IN_KEY ──'"'──▶ KEY_END  │
│      ▲                                                               ':'     │
│      │                                                                ▼      │
│    COMPLETE ◀─'}'─ IN_OBJECT ◀──────────────── VALUE_END ◀───────┘           │
│      ▲            ▲    '}' (depth 1) ◈4      (vuelve del value)              │
│      │            │                                                         │
│      │ (depth 0)  └──────▶ PARAMS_OBJECT ──'"'──▶ KEY_START ◈2 (depth 1)     │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────── BANDA 3 · RECONEXIÓN (semáforo COLON) ────────────────┐
│                                                                        │
│     COLON (semáforo) espera el arranque del value:                     │
│       '"' → string      '-' / '0'-'9' → number      't' / 'f' → boolean      'n' → null  │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌──────────────────── BANDA 2 · VALUES (contenido) ────────────────────┐
│                                                                        │
│     IN_STRING_VALUE ⇄ ESCAPE_IN_STRING     ◈1 trie del "name" (key name, depth 0)  │
│     IN_NUMBER_VALUE                        ◈3 tipo del value (depth 1 + fn)  │
│     IN_BOOL_VALUE / IN_NULL_VALUE          (literales exactos)         │
│                                                                        │
│     (cierre del value: '"' / ',' / '}' / ws) ──▶ VALUE_END → vuelve a BANDA 1  │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Cómo leerlo (reglas de lectura)
1. **El texto del JSON viaja en U**: la Banda 1 arma el esqueleto de
   izquierda a derecha (ROOT → ... → KEY_END); cuando ve `:` (COLON) el
   flujo BAJA por la Banda 3 hacia la Banda 2 para leer el value; al
   cerrarse el value (`"` / `,` / `}` / ws) viaja por VALUE_END y SUBE a
   la Banda 1 (decide `,` → siguiente key, o `}` → cierre del objeto).
2. **La Banda 3 NO acumula nada**: son semáforos de espera. Si el estado
   actual es KEY_END, el ÚNICO token que avanza es `:` (o whitespace). Por
   eso el pre-filtro (Fase 1) pregunta "¿con qué char puede arrancar el
   próximo token?" → expected_first_chars devuelve JUSTO esos.
3. **Los ◈ son los puntos donde el schema interviene** (Fase 3):
   - **◈1** (trie del name): dentro de la Banda 2, pero SOLO cuando la key
     es `"name"` y depth 0 — el value en curso es el nombre de la función.
   - **◈2** (param key): en la Banda 1, al entrar a KEY_START/IN_KEY con
     depth 1 — la key de un parámetro se valida contra el schema.
   - **◈3** (value type): en la Banda 2, al arrancar el value de un
     parámetro (depth 1) — el tipo que declara debe coincidir con el schema.
   - **◈4** (params close): en la flecha `PARAMS_OBJECT ─'}'▶` (depth 1→0) —
     el `}` de cierre de parameters exige que TODOS los required estén.
4. **depth es el ascensor entre plantas**: depth 0 = output object (name +
   parameters), depth 1 = adentro de parameters. Las cláusulas 2, 3 y 4
   SOLO miran depth 1; la 1 SOLO mira depth 0 con key "name".
5. **Las cláusulas se "abstienen" (True) cuando el token no pasó por su
   punto de intervención**: un token que no tocó la key de un parámetro no
   se valida contra la cláusula 2. Abstener ≠ permitir: significa "no es mi
   trabajo juzgar este token".

**En una línea:** la máquina tiene 3 bandas con 3 roles (esqueleto / contenido /
semáforos), y el schema es un guardia que interviene en 4 puntos puntuales
(◈1-◈4) — el 90% de los tokens pasa por su banda sin que ninguna cláusula
juzgue, y eso está bien.

---

## C1-C4 · LOS ÁRBOLES DE DECISIÓN DE CADA CLÁUSULA (schema_validator.py)

### C1 · _allows_name_value → el trie contra el value de "name"
```
¿El token TERMINA DENTRO del value de "name"?
(post ∈ _NAME_READ_PHASES ∧ key=="name" ∧ depth 0)
  │ SÍ ──────────────────────────────► ¿find_node(trie, name_buffer)?
  │                                     │ hay nodo ──► ✅ PERMITIDO (el name sigue construyéndose)
  │                                     │ no hay   ──► ❌ BLOQUEADO
  ▼ NO
¿El token SALIÓ del value de "name" en este step?
(pre ∈ _NAME_READ_PHASES ∧ key=="name" ∧ depth 0)
  │ SÍ ──────────────────────────────► ¿is_complete_name(trie, buffer)?
  │                                     │ nombre completo ──► ✅ (cerró el name bien)
  │                                     │ buffer ""/parcial ──► ❌ (ej: '{"name": {' → bloqueado)
  ▼ NO
✅ ABSTIENE (el token no tocó el value de "name")
```
**Truco:** las DOS ramas leen `new_state.name_buffer` — el buffer que
acumula LA STATE MACHINE, no el schema. Por eso el schema no re-parsea nada.

### C2 · _allows_param_key → keys de parameters contra el schema
```
¿depth == 1? (estamos DENTRO de parameters)
  │ NO ──► ✅ ABSTIENE (output keys: fuera del scope del schema)
  ▼ SÍ
¿Cambió el texto de la key (new_state.current_key != self._current_key)?
  │ NO ──► ✅ ABSTIENE (este token no escribió la key)
  ▼ SÍ ──► ¿función seleccionada?
              │ NO ──► ❌ BLOQUEA TODO (available vacío; refuerzo deliberado)
              ▼ SÍ
        available = params de la fn − keys_enclosed COMMITEADO
              ▼
        ¿La key quedó ABIERTA (post ∈ KEY_START/IN_KEY)?
              │ SÍ ──► ¿alguna available empieza con la key? (prefijo)
              │ NO ──► ¿la key EXACTA está en available? (membership)
```
**Por qué el trigger es por CAMBIO y no por fase:** el token `, "b": 3.0`
lee la key "b" a MITAD de token y termina en IN_NUMBER_VALUE (fase de
value, no de key) — las fases no lo verían; el cambio current_key "a"→"b"
es la ÚNICA señal. (Casos reales: ejemplo en slow-motion, diagrama D.)

### C3 · _allows_value_type → el tipo del value de un parámetro
```
¿El token TERMINA en fase de value (post ∈ _VALUE_PHASES)?
  │ SÍ ──► kind = _PHASE_KIND[post]        (la fase declara el tipo)
  ▼ NO
¿El token ARRANCÓ en COLON (pre) y el value se abrió+cerró EN ESTE token
('2,', 'true}', '"x",')?
  │ SÍ ──► kind = _VALUE_START_KINDS[primer char del texto (lstrip)]
  │ NO ──► ✅ ABSTIENE
      ▼ (kind definido)
¿kind es None? ──► ✅ ABSTIENE ('{' objeto anidado no está en el mapa)
      ▼
¿depth == 1 ∧ función seleccionada ∧ el parámetro existe en el schema?
  │ NO ──► ✅ ABSTIENE (name va por trie; key desconocida = default allow)
  ▼ SÍ
kind == param.type ──► ✅ permitido / ❌ bloqueado
```
**Por qué es idempotente:** la fase de un value en curso NO cambia token a
token → continuaciones re-chequeadas dan el mismo veredicto (mismo phase →
mismo kind). Los tokens largos de un string no se re-evalúan mal.

### C4 · _allows_params_close → el '}' de cierre de parameters
```
¿Salimos de depth 1 → depth 0 en este token?
(self._depth == 1 ∧ new_state.depth == 0)
  │ NO ──► ✅ ABSTIENE (no es el cierre de parameters;
  │                     el '}' del output object NO gatilla aquí)
  ▼ SÍ
¿función seleccionada?
  │ NO ──► ❌ BLOQUEADO (conservador: sin función no se sabe qué required exige)
  ▼ SÍ
¿params de la fn ⊆ keys_enclosed del estado SIMULADO?
  │ ⊆ ──► ✅ permitido cerrar
  │ ⊄ ──► ❌ faltan required keys
```
**Detalle fino:** compara contra `new_state.keys_enclosed` (SIMULADO), no el
commiteado — si el cierre viene junto al value final (`"b": 3}`), ese value
ya se registró en la copia simulada y cuenta para el ⊆.

---

## D · CÁMARA LENTA: UN TOKEN MULTI-FASE EN UN SOLO STEP (slow-motion)

Token: `, "b": 3.0` — arranca DENTRO del value anterior, cierra, abre key,
value nuevo. Un solo token BPE recorre 3 bandas. `simulate` lo ve char a char:

```
ESTADO PRE (commiteado):  phase=IN_NUMBER_VALUE  current_key="a"  depth=1
                          keys_enclosed={"a"}    fn_seleccionada con params {a: int, b: float}

 char  │ fase tras el char             │ efecto
───────┼───────────────────────────────┼─────────────────────────────────────────
 ','   │ VALUE_END                     │ value de "a" cerrado (Banda 2 → 1)
 ' '   │ VALUE_END                     │ whitespace: no consume
 '"'   │ KEY_START                     │ arranca key NUEVA (current_key → "")
 'b'   │ IN_KEY                        │ current_key="b"  ◈2 GATILLA (cambio "a"→"b")
 '"'   │ KEY_END                       │ key cerrada; "b" sigue en available (escapa al commiteado)
 ':'   │ COLON                         │ semáforo: esperando value (Banda 3)
 ' '   │ COLON                         │ whitespace: no consume
 '3'   │ IN_NUMBER_VALUE               │ value arranca  ◈3 kind="number"
 '.'   │ IN_NUMBER_VALUE               │ fracción (prefijo regex OK)
 '0'   │ IN_NUMBER_VALUE               │ "3.0" cierra sin ',' → sigue IN_NUMBER_VALUE

ESTADO POST (simulado): phase=IN_NUMBER_VALUE  current_key="b"  depth=1
                        keys_enclosed={"a"}  (¡"b" NO se cerró aún: solo se encuadra al ',')
```

**Por qué funciona (toda la cadena):**
- ◈1 no aplica (key="b", no "name").
- ◈2 sí gatilló en `'b'`: disponible = {a,b} − keys_enclosed commiteado {a}
  = {b}; key "b" abierta → prefijo OK.
- ◈3 sí gatilló en `'3'`: post ∈ IN_NUMBER_VALUE → kind "number"; el schema
  de "b" dice float → ✅.
- ◈4 no aplica (depth queda 1).
- El step devuelve True → el generator usa el post-state tal cual (sin
  re-simular). El estado REAL se actualiza una sola vez.

---

## E · LOS GAPS (limitaciones deliberadas del scope — documentadas en el código)

| # | Gap | Dónde | Por qué existe |
|---|-----|-------|----------------|
| 1 | Key duplicada EXACTA reconstruida en un solo token (reset "" + rebuild idéntico) escapa al trigger por cambio | cláusula 2 | El trigger compara textos de key; si el texto final es IGUAL al commiteado, "no hubo cambio" → abstiene. Rarísimo en BPE (necesitaría el token que contenga la key completa con contexto). Test: `test_identical_rebuild_slip_is_documented`. |
| 2 | Token que ATRAVIESA el value de "name" completo (ej: `'}, "name": "fn_greet",'`) no lo valida contra el trie | cláusula 1 | Ni el pre ni el post están en fase de name: las dos ramas abstienen. Para validarlo habría que re-simular el span del name dentro del token (parser paramétrico = B8 futuro). |
| 3 | El `}` del OUTPUT object (`{"name":"fn"}` sin "parameters") se completa igual | cláusula 4 | El plan solo gatea el cierre de parameters; el schema NO exige presence del key "parameters". Consecuencia deliberada del plan. |
| 4 | `', "b": "x",'` (key+value+cierre TODO en un token) esquiva la cláusula 3 | cláusula 3 | Ni termina en fase de value ni arrancó en COLON por pre: no hay de dónde leer el tipo. Requiere parser paramétrico (B8). |

**En una línea:** los gaps son TODOS "el token hizo demasiado en un solo
step" — para taparlos se necesita un parser paramétrico (B8); en
vocabularios BPE reales son raros y el plan los acepta como scope.

