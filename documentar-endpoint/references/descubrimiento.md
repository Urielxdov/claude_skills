# Qué inferir de cada vista

El objetivo de este paso es llegar a la entrevista con el máximo ya resuelto. Cada
campo que infieras del código es una pregunta que no le haces al usuario.

## Del schema, no de `urls.py`

`python manage.py spectacular --file -` es la verdad. De ahí salen:

- `operationId`, método y path de cada operación
- qué operaciones ya tienen `summary` / `description`
- qué operaciones ya tienen `x-registry` (y con qué campos)
- el bloque `info`: si `title`, `description` o `contact` vienen vacíos, el
  servicio entero llegará al hub sin ficha. No es trabajo de este skill
  arreglarlo —es un ajuste de `SPECTACULAR_SETTINGS`— pero sí reportarlo

Leer `urls.py` sirve para lo otro: mapear cada operación **a la clase o función
que la implementa**, que es lo que vas a editar.

## Señales por campo

### `summary` y docstring

Fuentes, en orden de fiabilidad:

1. Docstring existente de la vista o del método.
2. Nombre de la clase/acción (`AutorizarPagoView`, `reenviar`).
3. El serializer: sus campos dicen de qué entidad se habla.
4. El queryset y el modelo.

Redacta un `summary` de una línea, en imperativo o tercera persona, sin el verbo
HTTP ("Autoriza un cargo", no "POST para autorizar un cargo"). **Preséntalo como
propuesta**: el nombre de una clase no prueba la intención de negocio.

### `requiere.auth`

Se infiere y no se pregunta. Mira, en este orden:

- `permission_classes` en la vista; si no está, el default de
  `REST_FRAMEWORK["DEFAULT_PERMISSION_CLASSES"]` en settings.
- `authentication_classes` y su default equivalente.
- Clases de permiso propias: léelas. Un `TienePermisoPagos` suele nombrar un
  scope o un grupo concreto.
- Decoradores de throttling o de scope OAuth2.

Escribe el resultado en una línea legible: `"Bearer, scope pagos:write"`,
`"Sesión Django, requiere staff"`, `"Público"`.

Si la vista es `AllowAny`, dilo explícitamente (`"Público"`). Un endpoint sin
autenticación es justo el que conviene tener marcado.

### `requiere.idempotente`

Punto de partida por verbo:

| Verbo | Valor tentativo |
|---|---|
| GET, HEAD, OPTIONS | `True` |
| PUT, DELETE | `True` |
| POST, PATCH | `False` |

Ajusta si el código lo contradice: un `POST` que busca una cabecera
`Idempotency-Key`, o que hace `get_or_create`, es idempotente aunque el verbo
diga lo contrario. Un `PUT` que incrementa un contador, no lo es.

Cuando el código contradiga al verbo, **confírmalo en la entrevista**: es
exactamente el tipo de detalle que un consumidor asume mal.

### `requiere.efectos`

Rastrea en la vista y en lo que llama:

| Señal en el código | Efecto a declarar |
|---|---|
| `.save()`, `.create()`, `.update()`, `bulk_create` | escritura en la entidad X |
| `.delete()` | borrado de X |
| `transaction.atomic` | señala que hay varias escrituras acopladas |
| `.delay()`, `.apply_async()` | dispara la task Y de forma asíncrona |
| `send_mail`, `EmailMessage` | envía correo a Z |
| `post_save` / señales propias | efecto indirecto, sigue el receptor |
| publicación a cola/bus | emite el evento E |

Lo que veas, escríbelo. Lo que sospeches que existe pero no puedas seguir (una
señal con receptor en otra app, un handler dinámico), llévalo a la entrevista
como pregunta concreta, no como hueco.

### `equipo`

Por defecto **no se escribe**: el dueño sale de `info.contact` y vale para todo el
servicio. Solo se declara por operación cuando encuentres señal de que ese
endpoint es de otro equipo —un `CODEOWNERS` que asigna esa app a alguien
distinto, un docstring que lo dice, una convención de nombres de módulo—. Ante la
duda, se omite: un dueño heredado y correcto es mejor que uno explícito e
inventado.

### `depende-de`

Ver `dependencias.md`.

### `caso-de-uso`

**No lo infieras.** El código dice qué hace un endpoint; no dice quién lo llama ni
en qué momento del negocio, y ese es justo el dato que el hub no puede obtener
por su cuenta. Va siempre a la entrevista.

## Alcance del rastreo

Sigue las llamadas de la vista hacia dentro: métodos del serializer (`create`,
`update`, `validate`), servicios en `services/`, managers del modelo. Dos o tres
saltos bastan. Más allá se convierte en auditoría del proyecto entero y deja de
ser documentación de un endpoint.

## Vistas que no son DRF

Una `View`, `TemplateView` o función con `@require_http_methods` de Django plano
**no aparece en el OpenAPI**. No la anotes: `@extend_schema` ahí no hace nada.

Avísalo de forma explícita, porque el efecto es contraintuitivo: el runtime sí la
reporta como `inbound`, así que en el hub existirá como nodo con tráfico y sin
documentación posible. La salida es convertirla a DRF o registrarla a mano en el
hub; ninguna de las dos es trabajo de este skill.
