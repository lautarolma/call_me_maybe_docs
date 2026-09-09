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

