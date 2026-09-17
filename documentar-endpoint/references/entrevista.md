# La entrevista

Preguntas lo que el código no demuestra. Nada más.

La medida de una buena entrevista aquí es cuántas preguntas **no** hiciste porque
el paso de descubrimiento ya las había resuelto.

## Qué se pregunta y qué no

| Campo | ¿Se pregunta? |
|---|---|
| `requiere.auth` | **No.** Sale de `permission_classes` |
| `depende-de` | **No.** Sale del código |
| `requiere.efectos` | Solo los que no pudiste seguir |
| `requiere.idempotente` | Solo si el código contradice al verbo |
| `summary` / docstring | Se propone y se confirma, no se pregunta en blanco |
| `caso-de-uso` | **Siempre.** El código no sabe quién lo llama ni para qué |
| `equipo` | **No.** Se hereda de `info.contact`. Solo se pregunta si el servicio tiene varios dueños y no sabes cuál aplica |

## Por lotes, no por campo

Con seis endpoints y cinco campos cada uno, preguntar campo por campo son treinta
interrupciones y una conversación abandonada a la mitad.

Agrupa así:

1. **Una tanda para confirmar propuestas.** Presenta los `summary` y
   `caso-de-uso` propuestos de todo el grupo, y pregunta cuáles hay que corregir.
   Lo normal es que la mayoría estén bien; solo se itera sobre los que no.
2. **Una tanda para lo no inferible**, agrupada por tema (precondiciones de
   negocio, efectos que no pudiste seguir), no por endpoint.

Usa `AskUserQuestion` con opciones concretas siempre que puedas. "¿Cuál es el
caso de uso?" en abierto obliga a redactar; tres propuestas plausibles más la
opción de escribir la suya, no.

## Cómo formular

**Propón, no interrogues.** En vez de dejar el campo vacío, lleva tu mejor lectura
del código y pide confirmación:

> Para `POST /v1/pagos/autorizar` propongo:
> *"Checkout de la app móvil: reserva el importe antes de confirmar el pedido."*
> ¿Es correcto, o lo llama algo más?

**Di en qué te basaste.** "Por el `get_or_create` de la línea 88 diría que este
POST sí es idempotente, aunque el verbo sugiera lo contrario. ¿Confirmas?" El
usuario puede corregir el razonamiento, no solo el resultado.

**Agrupa lo que comparte respuesta.** Cinco endpoints del mismo ViewSet suelen
compartir precondiciones. Pregúntalo una vez.

## Cuándo parar

Si el usuario no sabe la respuesta —pasa, sobre todo con endpoints heredados—
escribe `"TODO: confirmar con el equipo"` en ese campo y sigue. Queda visible en
el código, visible en el OpenAPI y visible en el hub, que es exactamente donde
alguien lo verá y lo arreglará.

Un `TODO` explícito es honesto. Un campo ausente se lee como "no aplica", y una
descripción inventada se lee como un hecho. De las tres, solo la primera envejece
bien.
