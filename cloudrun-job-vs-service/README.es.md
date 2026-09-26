# Un Service de Cloud Run que responde antes de terminar no es una ejecución durable

[English](README.md) · **Español**

`cloud-run` · `cloud-scheduler` · `data loss` · `2026`

## Contexto

Trabajo batch programado sobre Google Cloud Run: exportaciones nocturnas, scrapers, ETL,
generación de informes. El código es un script — arranca, hace el trabajo, escribe un
fichero de salida y sale. La pregunta es qué primitiva de Cloud Run lo ejecuta, y la
respuesta no es obvia porque tanto un **Service** como un **Job** parecen funcionar a la
primera.

Aplica a cualquier cosa disparada por Cloud Scheduler que deba *completarse*, frente a
cualquier cosa disparada por un usuario que deba *responder*.

## Qué falló

Tres síntomas, en el orden en que nos los encontramos.

**1. El script desplegado como Service nunca arrancó.**

```
container failed to start and listen on the port defined by the PORT environment variable
```

Engañoso, porque el script corría perfectamente: hacía el trabajo y llamaba a `exit(0)`.
Un Service de Cloud Run interpreta un proceso que deja de escuchar como un health check
fallido. La salida limpia *es* el fallo.

**2. Reescrito como Service HTTP que devolvía `202 Accepted` y seguía en una promesa en
segundo plano, el trabajo se truncaba en silencio.**

Sin error. Sin línea de log. El fichero de salida en el almacenamiento de objetos se
escribía, pero incompleto — parcial en unas ejecuciones, al día en otras. Parecía una API
externa inestable.

**3. Cuando por fin funcionó, Cloud Scheduler seguía reportando el job como fallido.**

El historial de ejecuciones del Scheduler mostraba fallos. Los logs del servicio mostraban
`200 OK` en absolutamente todas las invocaciones durante una semana. Los dos decían la
verdad.

## Por qué

**El síntoma 1** es un desajuste de primitiva. Un Service es un servidor: su contrato es
*escucha en `PORT` y sigue escuchando*. Un Job es un proceso: su contrato es *corre hasta
el final y sal*. Desplegar código batch como Service invierte la condición de éxito.

**El síntoma 2** es el throttling de CPU. Por defecto, la CPU de un Service de Cloud Run
se recorta una vez enviada la respuesta. Una promesa que sigue viva después de
`res.status(202).send()` no tiene garantizada una porción de CPU — puede quedar pausada a
mitad de una lectura o de una subida y no reanudarse nunca, porque desde el punto de vista
de la plataforma la petición ya terminó. El trabajo no es durable; es una continuación
best-effort. Está bien para telemetría fire-and-forget y está mal para cualquier cosa cuya
ausencia notarías.

**El síntoma 3** son dos timeouts independientes. El `attempt-deadline` de Cloud Scheduler
y el `timeoutSeconds` del Service no tienen relación. Scheduler corta la conexión HTTP y
marca su propio intento como fallido en *su* deadline, sin importar que el servicio siga
corriendo y termine bien bajo su propio timeout, más alto. El resultado es una falsa
alarma en la capa de observabilidad: el trabajo está bien, el panel no.

Hemos visto el síntoma 3 dos veces, en sistemas sin relación entre sí. Las dos veces el
runtime real había crecido por encima de un deadline fijado cuando la carga era menor —
una vez porque una API de geocodificación de terceros exigía throttling por petición, otra
porque un timeout de lectura contra un proveedor externo más un cold start empujaron la
ejecución justo por encima de la línea. Ningún lint ni gate de despliegue lo detecta.

Un detalle que cuesta una hora al diagnosticar el síntoma 3: las Cloud Functions de 2ª
generación corren sobre Cloud Run. Consulta sus logs con
`resource.type="cloud_run_revision"`. El filtro antiguo `resource.type="cloud_function"`
es solo de 1ª generación y devuelve un conjunto vacío **sin error**, que se lee como «no
hay logs, luego nunca se ejecutó».

## El patrón

Elige la primitiva por la forma del trabajo, no por lo que ya tengas desplegado:

| | Cloud Run **Service** | Cloud Run **Job** |
|---|---|---|
| Carga | Servidor HTTP / API | Script batch |
| Disparo | Petición HTTP | Manual, Scheduler, Pub/Sub |
| Duración | Indefinida, escala a cero | Acotada (máx. 24h) |
| Debe escuchar en `PORT` | Sí | No |
| Señal de éxito | Sigue sirviendo | Sale con `0` |

Despliega el trabajo batch como Job:

```bash
gcloud run jobs create my-job \
  --image "$IMAGE_URL" \
  --project my-project --region my-region \
  --task-timeout 900 --max-retries 1 --memory 2Gi
```

Si de verdad necesitas un Service — porque la ruta batch comparte código con una API —
entonces haz que el handler haga `await` del trabajo y responda al final. Un parámetro
`?wait=true` explícito deja el contrato a la vista en el propio target del Scheduler:

```ts
app.post("/tasks/export", async (req, res) => {
  const durable = req.query.wait === "true";
  if (!durable) {
    // Solo es seguro si una cola o un Job ya asumió la propiedad del trabajo.
    res.status(202).json({ accepted: true });
    return;
  }
  const result = await runExport();   // la CPU está garantizada mientras la petición sigue abierta
  res.status(200).json(result);       // responder solo cuando el trabajo ya está en disco
});
```

Y luego dimensiona los deadlines de forma que el de fuera sea el mayor:

```
attempt-deadline de Cloud Scheduler  >  timeoutSeconds del Service  >  peor caso de runtime
```

De aquí salen tres reglas, cada una de las cuales nos costó un incidente en producción:

- **Un Job no hereda nada del Service que tiene al lado.** Si el Service lleva
  `--vpc-connector`, el Job necesita el suyo, explícito. Sin él obtienes `ETIMEDOUT` al
  conectar con una base de datos privada mientras el Service, en el mismo proyecto,
  funciona perfectamente.
- **`describe && update || create` debe repetir todos los flags en la rama `update`**, no
  solo `--image`. Secretos, conector VPC, memoria — todo lo que falte en `update`
  desaparece del Job en el siguiente despliegue.
- **Un Job de Node.js tiene que cerrar sus pools de conexiones.** El trabajo termina, los
  logs dicen «hecho», y la ejecución se reporta igualmente como fallida con
  `exit code: 0 and message: The configured timeout was reached`, porque un pool abierto
  mantiene vivo el event loop.

```ts
async function main() {
  await doWork();
  await pool.end();                 // sin esto el proceso no sale nunca
}
main().catch(async (err) => {
  await pool.end().catch(() => {});
  process.exit(1);
});
```

## Cómo verificarlo

**¿Están tus deadlines de Scheduler realmente por encima de tus runtimes?** Para cada job
programado, compara el deadline configurado con el p99 de las ejecuciones reales:

```bash
gcloud scheduler jobs describe my-scheduler-job \
  --location my-region --format="value(attemptDeadline)"

gcloud run services describe my-service \
  --region my-region --format="value(spec.template.spec.timeoutSeconds)"
```

Después lee los logs del propio servicio para esa misma ventana — no el estado del
Scheduler:

```bash
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="my-service"' \
  --freshness=7d --format="value(httpRequest.status, httpRequest.latency)"
```

Si el Scheduler dice fallido y esto dice `200` con una latencia por encima del attempt
deadline, tienes el síntoma 3 y no hay nada roto salvo las alertas.

**¿Hay algún handler haciendo trabajo durable después de responder?** Este grep encuentra
la forma:

```bash
grep -rn "res.status(202)\|res.sendStatus(202)" src/
```

Para cada coincidencia, mira si algo posterior a esa línea escribe en una base de datos o
en un almacén de objetos. Si lo hace, y ninguna cola ni Job asumió antes la propiedad del
trabajo, no es durable.

**¿Tu Job sale?** Ejecútalo y busca una finalización que se reporte como timeout:

```bash
gcloud run jobs execute my-job --region my-region --wait
```

Una línea de log diciendo que el trabajo se completó, seguida de un fallo por timeout, es
un handle abierto — casi siempre un pool de conexiones.

## Véase también

- [Cloud Run pierde el estado en memoria en cada reinicio](../cloudrun-sync-status-persistence/README.es.md) —
  la misma propiedad de «escala a cero» que estrangula tu promesa en segundo plano borra
  también tus variables de estado.
- [Escalar un servicio de Cloud Run a cero sin tocar IAM](../cloudrun-power-control/README.es.md)
