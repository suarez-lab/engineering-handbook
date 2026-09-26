# Pausa un servicio de Cloud Run escribiendo una anotación — y guarda el valor que estás pisando

[English](README.md) · **Español**

`cloud-run` · `cost` · `2026`

## Contexto

Una cartera de servicios de Cloud Run repartidos por varios proyectos GCP, la mayoría de
los cuales solo necesitan estar accesibles en horario laboral. Queríamos dos cosas desde
un panel de administración interno: un botón manual de pausa/reanudación, y un apagado
nocturno programado.

La restricción que lo condicionó todo: una org policy bloquea `gcloud run deploy`, porque
ese comando emite una llamada a `setIamPolicy` como parte de su flujo normal. Cualquier
cosa que redespliegue el servicio para cambiar su escalado queda descartada. También queda
descartado meter el SDK completo de GCP en una app Next.js cuyo bundle preferimos mantener
pequeño.

## Qué falló

**El primer intento escalaba los servicios a cero y no sabía devolverlos a su sitio.**
Pausar era fácil: poner el máximo de instancias a `0`. Reanudar implicaba elegir un número,
y el código elegía un valor por defecto. Servicios que estaban deliberadamente limitados a
2 volvieron a 10; uno limitado a 1 volvió pudiendo abrirse en abanico. Nada dio error. La
notificación fue la factura.

**El segundo intento, usando la API Knative, devolvía 404 sobre un servicio que
visiblemente existía.**

```
Requested entity was not found
```

Ni un error de permisos, ni una errata en el nombre del servicio — la URL estaba mal de
dos formas a la vez, y cada una por separado produce el mismo 404.

**El tercer intento funcionaba desde un script de prueba y fallaba desde Cloud Scheduler**,
con Scheduler rechazando la configuración del job en vez de fallar la petición en tiempo
de ejecución.

## Por qué

**El bug de la reanudación** es una escritura que falta. Pausar es destructivo:
`maxScale = 0` pisa el único registro de cuál era el valor anterior. Si no persistes el
original antes de pisarlo, la información ha desaparecido, y «reanudar» solo puede
significar «adivinar».

**El 404** es la forma de la API de Cloud Run con sabor Knative, que difiere de la forma
`v1` en dos aspectos que la gente arrastra por costumbre:

- La **región va en el nombre de host**, no en la ruta: `my-region-run.googleapis.com`.
  Un host sin región resuelve, autentica y reporta el servicio como no encontrado.
- El **namespace es el número de proyecto**, no el ID del proyecto. Un ID de proyecto en
  ese hueco también autentica y también reporta el servicio como no encontrado.

Dos errores independientes con un síntoma idéntico: por eso esto cuesta una tarde.

**El rechazo de Scheduler** es un hueco de métodos soportados: los targets HTTP de Cloud
Scheduler admiten `DELETE`, `GET`, `HEAD`, `POST` y `PUT` — no `PATCH`. Cualquier diseño
que tire de una actualización parcial desde un cron hay que reconvertirlo en un `POST`
contra un endpoint propio, que a su vez ejecuta el `PUT` aguas arriba.

## El patrón

**Persiste antes de pisar.** La máquina de estados es simétrica y el orden de escritura
importa — el valor guardado debe ser durable *antes* de la llamada destructiva:

```
pausa:      GET maxScale actual → WRITE power_state/<id> → PUT maxScale = 0
reanudar:   READ power_state/<id> → PUT maxScale = original → DELETE power_state/<id>
```

**Cambia una anotación, nada más.** El `PUT` toca solo
`spec.template.metadata.annotations`, así que nunca llama a `setIamPolicy` y se queda
dentro de la org policy:

```ts
const host = `https://${region}-run.googleapis.com`;
const url =
  `${host}/apis/serving.knative.dev/v1/namespaces/${projectNumber}/services/${serviceName}`;
//        la región en el host  ^^^^^^        NÚMERO de proyecto ^^^^^^^^^^^^^

const svc = await (await fetch(url, { headers: auth })).json();

svc.spec.template.metadata.annotations["autoscaling.knative.dev/maxScale"] = "0";
delete svc.status;          // campo de solo lectura; si lo dejas, el PUT falla

await fetch(url, { method: "PUT", headers: auth, body: JSON.stringify(svc) });
```

**Coge el token del metadata server**, no de un fichero de clave — que además resulta ser
la única opción cuando una org policy bloquea la creación de claves de cuenta de servicio:

```ts
const r = await fetch(
  "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token",
  { headers: { "Metadata-Flavor": "Google" }, cache: "no-store" },
);
const { access_token } = await r.json();
```

**Dale al runtime el rol más estrecho que funcione.** Escribir el objeto del servicio
necesita `roles/run.developer`; solo la administración entre proyectos necesita
`roles/run.admin`:

```bash
# la service account de runtime del servicio que apagas y enciendes
SA="$(gcloud run services describe my-service --region=my-region \
  --format='value(spec.template.spec.serviceAccountName)')"
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:${SA}" \
  --role="roles/run.developer" --condition=None
```

**Para el apagado programado, dos crons contra tu propio endpoint.** Expresa el horario en
UTC y convierte una sola vez, en un comentario, en vez de confiar en una cadena de zona
horaria que no vas a releer:

```bash
# El horario laboral local es UTC−4: bajar a las 22:00 locales, subir a las 07:00 locales.
gcloud scheduler jobs create http my-service-scale-down \
  --schedule="0 2 * * *" --time-zone="UTC" \
  --uri="https://my-service.example/internal/scaling?mode=down" \
  --http-method=POST --message-body='{}'

gcloud scheduler jobs create http my-service-scale-up \
  --schedule="0 11 * * *" --time-zone="UTC" \
  --uri="https://my-service.example/internal/scaling?mode=up" \
  --http-method=POST --message-body='{}'
```

Para la variante programada, las anotaciones que hay que cambiar son `minScale` (`"0"`
abajo, `"1"` arriba) y `run.googleapis.com/cpu-throttling` (`"true"` abajo, `"false"`
arriba) — ese par es lo que apaga una instancia caliente, en vez de limitar una fría.

**Tres reglas de UI**, porque este control es destructivo y cabe en un clic:

1. Confirmación explícita, nombrando el servicio y el proyecto GCP en el diálogo.
2. Una acción cada vez — deshabilitar todos los botones mientras haya una petición en curso.
3. Modela `unknown` como estado de primera clase. Si la lectura del escalado actual falló,
   el botón queda deshabilitado. Nunca actúes sobre un valor que no has podido leer.

## Cómo verificarlo

**Comprueba tu URL antes de culpar a los permisos.** Si esto devuelve el servicio, la forma
es correcta; si da 404, una de las dos sustituciones está mal:

```bash
PROJECT_NUMBER=$(gcloud projects describe my-project --format='value(projectNumber)')
TOKEN=$(gcloud auth print-access-token)

curl -s -o /dev/null -w '%{http_code}\n' \
  -H "Authorization: Bearer $TOKEN" \
  "https://my-region-run.googleapis.com/apis/serving.knative.dev/v1/namespaces/${PROJECT_NUMBER}/services/my-service"
```

`200` significa que host y namespace son correctos. `404` con un token válido significa
que no lo son — revisa primero el prefijo de región en el host, es el más frecuente de los
dos.

**Demuestra que reanudar restaura el original, no un valor por defecto.** Lee el valor,
pausa, reanuda, lee otra vez:

```bash
scale() { gcloud run services describe my-service --region my-region \
  --format="value(spec.template.metadata.annotations['autoscaling.knative.dev/maxScale'])"; }

scale                       # p. ej. 2
# ...pausar desde el panel...
scale                       # 0
# ...reanudar desde el panel...
scale                       # tiene que volver a ser 2, no un valor por defecto
```

Si la tercera lectura no coincide con la primera, falta el paso de persistir-antes-de-pisar,
o está compitiendo con el `PUT`.

**Confirma que no estás chocando con la política de IAM.** Una pausa que falla con
`PERMISSION_DENIED` sobre `setIamPolicy` en vez de sobre `run.services.update` significa
que algo del camino sigue redesplegando en lugar de parchear la anotación.

## Véase también

- [Un Service de Cloud Run que responde antes de terminar no es una ejecución durable](../cloudrun-job-vs-service/README.es.md)
- [Cloud Run pierde el estado en memoria en cada reinicio](../cloudrun-sync-status-persistence/README.es.md) —
  relevante si tu panel cachea en memoria el estado pausado/en marcha.
