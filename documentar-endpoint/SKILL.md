---
name: documentar-endpoint
description: "Documenta endpoints DRF directamente en el código — docstring + @extend_schema con la extensión x-registry — para que el OpenAPI que llega al hub de django-api-registry explique para qué sirve cada endpoint, qué requiere y de qué depende. Infiere del código lo que puede y pregunta solo lo que no. Úsalo cuando pidan: documentar un endpoint, anotar una vista, describir para qué sirve un endpoint, rellenar extend_schema, preparar el manifest, o auditar qué endpoints están sin documentar. Disparadores: documentar endpoint, anotar vista, extend_schema, x-registry, OpenAPI, drf-spectacular, registry_dump, manifest, para qué sirve este endpoint."
metadata:
  paquete: django-api-registry
  version: "0.1.0"
  idioma: es
---

# Documentar endpoints para el hub

El hub de `django-api-registry` ya sabe **cuánto** tráfico hace cada endpoint y **a
quién** llama: eso lo mide el runtime con OpenTelemetry. Lo que no sabe es **para
qué sirve**. Este skill escribe esa capa semántica en el código de las vistas, de
forma que drf-spectacular la arrastre al OpenAPI y el manifest la lleve al hub.

La fuente de verdad es **el código**. No generes fichas `.md` ni `.yaml` paralelas:
se desincronizan. El OpenAPI generado ya es el artefacto estructurado.

## Reglas que no se negocian

1. **Infiere lo que el código demuestra. Pregunta lo que no.** Nunca inventes el
   propósito de negocio de un endpoint, y nunca lo dejes en blanco en silencio.
2. **Nunca pises texto escrito a mano.** Al re-ejecutar sobre una vista ya
   anotada, rellena solo lo vacío o marcado `TODO`; para el resto, muestra el
   diff y pregunta.
3. **No preguntes lo que ya inferiste.** Si `permission_classes` dice la
   autenticación, no preguntes por la autenticación.
4. **No documentes consumidores ni criticidad.** El hub los deduce del runtime;
   escribirlos a mano es duplicar un dato que envejece mal.
5. **Máximo 8 operaciones por ejecución.** No es arbitrario: documentar cuarenta
   endpoints de una vez produce un diff que nadie revisa, y documentación sin
   revisar es exactamente el problema que esto viene a resolver. Si el objetivo
   tiene más, haz las primeras 8 y dilo.

## Flujo

### 1. Inventario real

Arranca por el schema, no por `urls.py`:

```
python manage.py spectacular --file -
```

Eso lista las operaciones que **existen de verdad** y cuáles carecen de
`summary`/`description`. Si el comando falla, el proyecto no tiene
drf-spectacular configurado: detente y dilo, no sigas a ciegas.

Mira también el bloque `info` del schema. Es la ficha del servicio en el hub, y
`info.contact` es el equipo dueño por defecto de todos los endpoints que estás por
anotar. Si viene vacío, anótalo para el reporte final: se arregla en
`SPECTACULAR_SETTINGS` y no bloquea nada. Los dos niveles, en
`references/anotacion.md`.

Comprueba también que la versión instalada soporta extensiones (una línea, en
`references/anotacion.md`).

### 2. Resolver el objetivo a vistas

El usuario invoca con un archivo, una ruta HTTP o una app. Mapea las operaciones
del schema a sus vistas (`APIView`, `ViewSet`, `@api_view`, `@action`) leyendo
`urls.py` y los routers.

Presenta la lista de operaciones detectadas con su estado (`sin anotar` /
`parcial` / `anotada`) y confirma el lote antes de tocar nada.

Si encuentras vistas Django planas (no DRF), **avisa que no saldrán en el
OpenAPI** y no las anotes: el hub no las verá aunque el runtime sí las reporte.

### 3. Inferir del código

Lee la vista y lo que llama. Ver `references/descubrimiento.md` para qué señal
alimenta qué campo, y `references/dependencias.md` para rastrear las llamadas
salientes.

### 4. Entrevistar por lotes

Una sola tanda de preguntas para todo el grupo de endpoints, no una por campo.
Ver `references/entrevista.md`.

### 5. Escribir la anotación

Contrato exacto en `references/anotacion.md` — **léelo antes de escribir**. Tiene
cuatro trampas que fallan en silencio, sin error ni warning: el orden de
`@extend_schema` respecto a `@action`, `extensions` en la clase de un ViewSet, el
docstring de clase que se propaga a todas las acciones, y el `summary` que nunca
se infiere.

Añade el import de `extend_schema`/`extend_schema_view` si falta.

### 6. Verificar

```
python manage.py spectacular --fail-on-warn --file -
```

`--fail-on-warn` atrapa justo lo que falla callado: extensiones mal prefijadas y
nombres de acción inexistentes en `extend_schema_view`. Confirma que `x-registry`
aparece bajo las operaciones que tocaste.

Si el servicio ya tiene el manifest conectado, cierra el circuito:

```
python manage.py registry_dump --print
```

### 7. Reportar

Tabla final: operación · documentada · campos que quedaron sin respuesta · vistas
fuera de alcance. Sin emojis.

Si el bloque `info` venía incompleto (paso 1), dilo aquí con el ajuste concreto
que falta en `SPECTACULAR_SETTINGS`. Es una línea de configuración que le da
nombre y dueño a todo el servicio en el catálogo; cuesta menos que documentar un
endpoint y rinde más.

## Antes de terminar: ¿el schema es público?

`x-registry` lleva nombres de servicios internos, settings de URLs, precondiciones
de negocio y efectos secundarios. Si el proyecto sirve `/api/schema/` sin
autenticación, todo eso queda expuesto.

Comprueba `SPECTACULAR_SETTINGS["SERVE_PERMISSIONS"]` y la ruta del schema en
`urls.py`. Si está abierto, **dilo antes de anotar**, no después: es más fácil
decidir el modelo de exposición con cero endpoints anotados que con cuarenta.

## Referencias

| Tema | Archivo | Cárgalo cuando |
|---|---|---|
| Qué inferir de cada vista | `references/descubrimiento.md` | Paso 3, siempre |
| Llamadas salientes | `references/dependencias.md` | Paso 3, si la vista llama a otros servicios |
| Qué preguntar y cómo | `references/entrevista.md` | Paso 4, siempre |
| Contrato de anotación y sus trampas | `references/anotacion.md` | Paso 5, siempre |

<!-- django-api-registry 0.1.0 -->
