# Dependencias salientes

`depende-de` declara a qué servicios llama un endpoint, leído del código. Es la
contraparte estática de lo que el runtime observa: el hub cruza ambas y responde
dos preguntas que por separado no puede.

- Declarado y **no** observado → camino muerto, o rama que nadie ejerce en este
  entorno.
- Observado y **no** declarado → dependencia que se coló sin quedar documentada.

Por eso importa que lo declarado sea honesto: una dependencia inventada produce
una alerta falsa, y una omitida deja pasar un cambio que rompe.

## Qué rastrear

Desde la vista, hacia dentro, dos o tres saltos:

| Patrón | Qué extraer |
|---|---|
| `requests.get(...)`, `httpx.post(...)` directos | método + URL |
| Cliente envuelto (`crm_client.obtener_cliente(id)`) | entra al cliente y saca el método + URL reales |
| `session = requests.Session()` con `base_url` | la base más el path de cada uso |
| SDK de terceros (`stripe.Charge.create`) | el servicio; la ruta concreta suele no ser visible |

No cuentan como dependencia saliente: consultas a la base de datos, caché,
lectura de archivos, ni tasks de Celery encoladas (esas ya van en
`requiere.efectos`).

## Forma de cada entrada

```python
{
    "servicio": "crm",
    "llamada": "GET /api/customers/{}/",
    "origen": "billing/services/crm.py:42",
}
```

- **`servicio`** — nombre corto y estable. Si la URL sale de
  `settings.CRM_URL`, el nombre del setting es la mejor pista. Si es un host
  literal (`crm.internal`), úsalo tal cual; el hub ya resuelve host → servicio
  con las base URLs del manifest.
- **`llamada`** — verbo y path.
- **`origen`** — `archivo:línea` del punto de llamada real, no de la vista. Es lo
  que permite ir a verlo.

## URLs que no son literales

Casi nunca lo son. La regla: **escribe el patrón, nunca un valor de ejemplo.**

```python
requests.get(f"{settings.CRM_URL}/api/customers/{cid}/")
# -> "GET /api/customers/{}/"
```

El `{}` no es un capricho de formato: es exactamente el placeholder que produce
`normalize_path` en el runtime (`registry_client/normalize.py`), así que lo
declarado y lo observado quedan en la misma forma y el hub puede compararlos sin
traducir.

Si el path entero se arma en tiempo de ejecución y no puedes determinarlo:

```python
{"servicio": "crm", "llamada": "GET (path dinámico)", "origen": "..."}
```

Declara el servicio, que sí conoces, y sé explícito sobre lo que no. Un
`"llamada": "GET /api/algo/"` inventado es peor que admitir el hueco.

## Base URL desde settings

Cuando la base salga de un setting, mira si el proyecto declara
`SPECTACULAR_SETTINGS["SERVERS"]`. Ahí es donde el hub lee las base URLs para
resolver `host` → servicio. Si el servicio destino no está en ningún `SERVERS` del
ecosistema, el hub lo tratará como dependencia externa — lo cual es correcto y
también es dato útil, pero conviene mencionarlo al usuario si era un servicio
interno que simplemente no está registrado todavía.
