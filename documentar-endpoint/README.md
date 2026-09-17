# documentar-endpoint

Skill de Claude Code para documentar endpoints DRF **en el código** — docstring +
`@extend_schema` con la extensión `x-registry` — de forma que drf-spectacular
arrastre esa capa semántica al OpenAPI y el manifest la lleve al hub de
`django-api-registry`.

El hub ya sabe **cuánto** tráfico hace cada endpoint y **a quién** llama: eso lo mide
el runtime con OpenTelemetry. Lo que no sabe es **para qué sirve**. Eso es lo que
escribe esta skill.

La fuente de verdad es el código: no genera fichas `.md` ni `.yaml` paralelas,
porque se desincronizan.

## Origen

Viene empaquetada dentro de `django-api-registry==0.1.0`, en
`registry_client/skills/documentar-endpoint/`. Esta es una copia extraída del
paquete instalado, publicada aparte para poder instalarla como skill sin depender
de tener el paquete en el entorno.

## Instalación

Instrucciones completas en el [README de la raíz](../README.md). En corto, desde un
clon del repo:

```bash
# skill personal — disponible en todos tus proyectos
cp -r documentar-endpoint ~/.claude/skills/

# o skill de un proyecto — versionada con el repo que la usa
cp -r documentar-endpoint /ruta/al/proyecto/.claude/skills/
```

Lo que tiene que quedar en `skills/documentar-endpoint/` es `SKILL.md` y la carpeta
`references/`. Claude Code la descubre por el frontmatter de `SKILL.md`. Reinicia la
sesión y compruébala con `/documentar-endpoint`.

## Uso

Se dispara sola cuando pides documentar un endpoint, anotar una vista, rellenar
`extend_schema`, preparar el manifest o auditar qué endpoints están sin documentar.
También la puedes invocar directo:

```
/documentar-endpoint apis/helpdesks/views/
```

El objetivo puede ser un archivo, una ruta HTTP o una app.

## Qué hace, en orden

1. **Inventario real** — arranca por `python manage.py spectacular --file -`, no por
   `urls.py`, para listar las operaciones que existen de verdad y cuáles carecen de
   `summary`/`description`.
2. **Resolver el objetivo a vistas** — mapea operaciones a `APIView`, `ViewSet`,
   `@api_view` y `@action`, y confirma el lote antes de tocar nada.
3. **Inferir del código** — permisos, serializers, llamadas salientes.
4. **Entrevistar por lotes** — una sola tanda de preguntas para todo el grupo, no
   una por campo.
5. **Escribir la anotación** — docstring + `@extend_schema` con `x-registry`.
6. **Verificar** — `spectacular --fail-on-warn` y, si el manifest está conectado,
   `registry_dump --print`.
7. **Reportar** — tabla de operación · documentada · campos sin respuesta · vistas
   fuera de alcance.

## Reglas que la skill no negocia

- Infiere lo que el código demuestra; pregunta lo que no. Nunca inventa el propósito
  de negocio ni lo deja en blanco en silencio.
- Nunca pisa texto escrito a mano: al re-ejecutar sobre una vista ya anotada rellena
  solo lo vacío o marcado `TODO`.
- No pregunta lo que ya infirió.
- No documenta consumidores ni criticidad: el hub los deduce del runtime.
- **Máximo 8 operaciones por ejecución** — un diff de cuarenta endpoints no lo revisa
  nadie, y documentación sin revisar es el problema que esto viene a resolver.

## Aviso que conviene leer antes de anotar

`x-registry` lleva nombres de servicios internos, settings de URLs, precondiciones de
negocio y efectos secundarios. Si el proyecto sirve `/api/schema/` sin autenticación,
todo eso queda expuesto. La skill revisa
`SPECTACULAR_SETTINGS["SERVE_PERMISSIONS"]` y avisa **antes** de anotar, no después.

## Estructura

```
SKILL.md                      — el flujo y las reglas
references/descubrimiento.md  — qué inferir de cada vista
references/dependencias.md    — rastreo de llamadas salientes
references/entrevista.md      — qué preguntar y cómo
references/anotacion.md       — contrato de anotación y sus cuatro trampas silenciosas
```

## Requisitos

- Proyecto Django con DRF y **drf-spectacular** configurado. Si
  `manage.py spectacular` falla, la skill se detiene y lo dice en vez de seguir a
  ciegas.
- `django-api-registry` solo hace falta para el paso del manifest
  (`registry_dump`); la anotación funciona sin él.
