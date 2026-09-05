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

