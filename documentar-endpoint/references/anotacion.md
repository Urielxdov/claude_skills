# Contrato de anotación

## El reparto: docstring vs. `@extend_schema`

| Pieza | Qué lleva | Por qué ahí |
|---|---|---|
| **Docstring** de la vista o acción | La prosa larga: qué hace, para quién, precondiciones y efectos en lenguaje corrido | drf-spectacular ya lo usa como `description` (`AutoSchema.get_description()` devuelve el docstring de la acción, o el de la clase) |
| **`@extend_schema(summary=...)`** | Una línea, ≤80 caracteres | `AutoSchema.get_summary()` devuelve `None` siempre: **el summary nunca sale del docstring.** Si no lo pasas, el endpoint aparece sin título en el hub |
| **`@extend_schema(extensions=...)`** | Lo estructurado que OpenAPI no sabe representar | El hub lo consume como datos, no como texto |

**Regla anti-duplicación**: el propósito vive en `summary` + docstring. No lo repitas
dentro de `x-registry`.

**Regla asimétrica — la parte que más se equivoca:**

- APIView, `@api_view`, `@action` → **no pases `description=`**. Existe el método, y
  su docstring ya es la descripción. Duplicarlo deja dos textos que divergen.
- Acciones implícitas de ViewSet (`list`, `retrieve`, `create`, `update`,
  `partial_update`, `destroy`) → **sí pasa `description=`**, porque el método no
  existe en tu código y no hay docstring donde escribirlo.

## Forma canónica

```python
from drf_spectacular.utils import extend_schema


class AutorizarPagoView(APIView):
    permission_classes = [TienePermisoPagos]

    @extend_schema(
        summary="Autoriza un cargo contra el emisor",
        extensions={
            "x-registry": {
                "version": 1,
                "caso-de-uso": "Checkout de la app móvil y del portal de pagos.",
                "requiere": {
                    "auth": "Bearer, scope pagos:write",
                    "precondiciones": ["cliente con KYC aprobado"],
                    "idempotente": False,
                    "efectos": [
                        "reserva fondos por 7 días",
                        "emite evento pago.autorizado",
                    ],
                },
                "depende-de": [
                    {
                        "servicio": "emisor",
                        "llamada": "POST /auth/",
                        "origen": "pagos/services/emisor.py:42",
                    },
                ],
            }
        },
    )
    def post(self, request):
        """Autoriza un cargo contra el emisor y reserva el importe.

        Valida fondos disponibles, aplica antifraude y deja el importe
        reservado siete días. No mueve dinero: eso lo hace la captura.
        """
        ...
```

## Por qué una sola clave `x-registry` anidada

Y no `x-caso-de-uso`, `x-requiere`, `x-depende-de` sueltas.

Dos razones. Una de forma: `sanitize_specification_extensions` **descarta en
silencio** toda clave que no case con `^x-`, así que el prefijo no es negociable y
conviene tener uno solo que revisar. Y una de fondo: una sola extensión que
versionar, que validar, que el hub lee de un campo, y que se quita entera si esto
cambia de forma. El precio es que en un Swagger UI plano se lee peor que tres
claves sueltas.

El `version: 1` interno permite migrar el formato más adelante sin adivinar.

## Los campos

| Campo | Tipo | Origen |
|---|---|---|
| `version` | `int` | Siempre `1` |
| `caso-de-uso` | `str` | **Se pregunta.** Quién lo usa y en qué momento del negocio |
| `equipo` | `str` | **Se omite casi siempre.** Solo si este endpoint tiene otro dueño que el del servicio |
| `requiere.auth` | `str` | **Se infiere** de `permission_classes` / `authentication_classes` |
| `requiere.precondiciones` | `list[str]` | **Se pregunta.** Estado previo que el llamador debe garantizar |
| `requiere.idempotente` | `bool` | **Se infiere** del verbo; se confirma si el código lo contradice |
| `requiere.efectos` | `list[str]` | **Se infiere** parcialmente; se completa preguntando |
| `depende-de` | `list[dict]` | **Se infiere.** Ver `dependencias.md` |

Dos reglas de forma:

- **Todo JSON-serializable puro**: `str`, `bool`, `int`, `list`, `dict`. Nada de
  `gettext_lazy`, `Decimal`, enums ni `Path` — revientan al serializar el schema.
- **Lo que no aplica se omite**, no se pone vacío. Un campo ausente dice "no
  aplica"; un `"caso-de-uso": ""` es ruido que alguien tendrá que investigar.

## Los dos niveles de documentación

Todo lo que el hub sabe de un servicio llega por el OpenAPI, en dos niveles. Este
skill escribe uno solo:

| Nivel | Dónde vive en el schema | Quién lo configura |
|---|---|---|
| Servicio | `info.title`, `info.description`, `info.contact` | `SPECTACULAR_SETTINGS`, una vez por proyecto |
| Operación | `summary`, `description`, `x-registry` | Este skill, endpoint por endpoint |

`info.contact` es el equipo dueño por defecto de todo el servicio, y es el único
dato del catálogo del hub sin otra fuente posible:

```python
SPECTACULAR_SETTINGS = {
    "TITLE": "Facturación",
    "DESCRIPTION": "Emisión y timbrado de CFDI.",
    "CONTACT": {"name": "equipo-facturacion"},
    "VERSION": "1.0.0",
}
```

Si faltan, `registry_dump` lo avisa y el servicio aparece en el catálogo sin
nombre, sin resumen o sin dueño. Es un ajuste de una sola vez: señálalo en el
reporte final del paso 7, no bloquees la anotación por eso.

`x-registry.equipo` existe para el caso contrario: un endpoint concreto que
pertenece a un equipo **distinto** del `CONTACT` del servicio, cosa que pasa en
servicios grandes con varios dueños. Fuera de ese caso se omite. Repetir el dueño
del servicio en cada operación son cuarenta copias de un dato que cambia de golpe,
y la primera vez que cambie quedarán treinta y nueve mintiendo.

## Las cuatro formas

### A. APIView con un solo método

Como la forma canónica de arriba: `@extend_schema` sobre el método.

### B. APIView con varios métodos

Un solo `@extend_schema` sobre la clase **no puede** llevar un `x-registry` distinto
por método. Usa `@extend_schema_view`:

```python
@extend_schema_view(
    get=extend_schema(summary="Consulta el estado del pago", extensions={...}),
    post=extend_schema(summary="Autoriza un cargo", extensions={...}),
)
class PagoView(APIView):
    ...
```

### C. ViewSet

```python
@extend_schema_view(
    list=extend_schema(
        summary="Lista las facturas del cliente autenticado",
        description="Devuelve las facturas emitidas, más recientes primero.",
        extensions={"x-registry": {...}},
    ),
    retrieve=extend_schema(
        summary="Detalle de una factura",
        description="...",
        extensions={"x-registry": {...}},
    ),
)
class FacturaViewSet(ModelViewSet):
    """Facturas de venta."""
```

### D. Acciones `@action`

```python
    @extend_schema(summary="Reenvía la factura por correo", extensions={...})
    @action(detail=True, methods=["post"])
    def reenviar(self, request, pk=None):
        """Reenvía el PDF de la factura al correo de contacto del cliente."""
```

## Trampas

Las cuatro fallan **en silencio**: no hay error, no hay warning, simplemente el
dato sale mal o no sale.

**1. `@extend_schema` debe ir ARRIBA de `@action`.** DRF hace `func.kwargs = kwargs`
— asignación, no merge. Si `extend_schema` corre primero, `@action` pisa lo que
dejó y la anotación desaparece sin avisar.

**2. Nunca pongas `extensions=` en un `@extend_schema` aplicado a la clase de un
ViewSet.** Ese decorador hace `f.schema = ExtendedSchema()` global, así que el
mismo `x-registry` queda estampado en `list`, `retrieve`, `create`… Cinco
operaciones distintas declarando las mismas dependencias y los mismos efectos: no
es documentación incompleta, es documentación falsa. Para ViewSets, siempre
`@extend_schema_view`.

**3. El docstring de clase de un ViewSet se propaga a todas las acciones.**
`get_doc()` recorre el MRO, así que un docstring largo en la clase se convierte en
la `description` de las cinco operaciones. Regla: docstring de clase de ViewSet =
una línea de overview, y la descripción real va por acción.

**4. Si no pasas `summary`, no hay summary.** No se infiere del docstring ni del
nombre del método.

## Comprobación previa

```
python -c "import inspect, drf_spectacular.utils as u; print('extensions' in inspect.signature(u.extend_schema).parameters)"
```

Si imprime `False`, la versión instalada es anterior a 0.22 y no soporta
extensiones: detente y pide subir drf-spectacular. El mínimo que declara el
paquete (`>=0.27`) lo cubre, pero el entorno del consumidor puede estar
desalineado.

## Re-ejecución: no pisar lo escrito a mano

Antes de escribir sobre una vista que ya tiene `@extend_schema`:

1. Lee el `x-registry` existente.
2. Rellena solo lo ausente o cuyo valor sea `TODO`.
3. Para cualquier campo ya poblado que cambiarías, **muestra el diff y pregunta**.
   Que un texto suene mejor no justifica reemplazar lo que escribió alguien que
   conoce el dominio.
4. **Fusiona el decorador existente**, añadiéndole kwargs. Nunca apiles un segundo
   `@extend_schema`.
5. Una dependencia declarada que ya no aparece en el código **no se borra**: se
   reporta como discrepancia y se pregunta. Puede que el código cambiara, o puede
   que tu rastreo no llegara.

No añadas marcadores tipo `"generado-por"`. Mienten en cuanto alguien edita el
campo a mano, y entonces son peor que no tenerlos.

Criterio de aceptación: **una segunda ejecución sin cambios de código produce cero
ediciones.**
