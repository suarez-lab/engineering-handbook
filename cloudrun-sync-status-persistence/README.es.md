# Cloud Run pierde el estado en memoria en cada reinicio — tu endpoint de status está mintiendo

[English](README.md) · **Español**

`cloud-run` · `correctness` · `2026`

## Contexto

Un panel de administración con un indicador de «última sincronización». Un sync en segundo
plano escribe un JSON en el almacenamiento de objetos cada pocas horas; una variable a
nivel de módulo lleva la cuenta de si está `idle`, `running` o `error`, y de cuándo
terminó bien por última vez; un endpoint `GET /api/sync/<x>/status` devuelve esa variable.

Cualquier servicio de Cloud Run que escale a cero tiene esta forma en algún sitio,
normalmente escrita pronto, cuando el servicio nunca estaba inactivo el tiempo suficiente
para notarlo.

## Qué falló

El panel mostraba el sync como si no se hubiera ejecutado nunca. Marca de tiempo en
blanco, estado `idle`.

El sync sí se había ejecutado, a su hora, horas antes — el fichero de salida en el
almacenamiento de objetos estaba correcto y al día, con un `syncedAt` reciente dentro.
Los datos estaban bien. Lo único mal era el informe sobre los datos.

La parte engañosa es que esto parece un sync que falla de forma intermitente. Se reproduce
«aleatoriamente»: bien justo después de un deploy, en blanco a la mañana siguiente, bien
otra vez después de que alguien pulse por ahí. Dos ingenieros pueden discrepar sobre si el
bug existe, porque quien llega primero al endpoint paga el cold start y quien llega segundo
no.

## Por qué

Cloud Run termina las instancias cuando no hay tráfico, y las reemplaza en cada deploy.
Un `let` en el ámbito de módulo vive exactamente lo que vive la instancia.

Así que `idle` está sobrecargado. Significa dos cosas distintas que el código no puede
distinguir:

- *este sync no está corriendo ahora mismo, y aquí tienes cuándo terminó la última vez*, y
- *este proceso no tiene ni idea, porque arrancó hace treinta segundos*

Los dos se serializan al mismo JSON. El endpoint no está reportando el estado del sync;
está reportando la edad del contenedor.

El arreglo no es añadir una base de datos. El registro durable de cuándo terminó bien el
sync ya existe — es el fichero de salida que el propio sync escribió. Simplemente no se
estaba leyendo.

## El patrón

Cuando el estado en memoria diga `idle`, recurre al artefacto. Lee solo los primeros
bytes del fichero de salida: la marca de tiempo vive cerca del principio, y descargar una
exportación de varios megabytes para averiguar un campo es exactamente cómo un endpoint de
status se convierte en la ruta más lenta de la aplicación.

```ts
type SyncStatus = {
  state: "idle" | "running" | "error";
  syncedAt?: string;
};

async function resolveSyncStatus(
  memState: SyncStatus,
  objectName: string,
): Promise<SyncStatus> {
  // 'running' y 'error' solo se pueden saber en memoria — confía en ellos.
  if (memState.state !== "idle") return memState;

  try {
    const file = storage.bucket(process.env.OUTPUT_BUCKET!).file(objectName);
    const [exists] = await file.exists();
    if (!exists) return memState;

    // Lectura por rango: los primeros 200 bytes, no la exportación entera.
    const [head] = await file.download({ start: 0, end: 199 });
    const match = head.toString().match(/"syncedAt"\s*:\s*"([^"]+)"/);
    if (match) return { state: "idle", syncedAt: match[1] };
  } catch {
    // Que el almacenamiento no responda no es un fallo del sync — degrada a lo que sabemos.
  }
  return memState;
}
```

El handler pasa a ser async, que es el cambio que más se olvida:

```ts
// Antes — en blanco tras cualquier reinicio
app.get("/api/sync/reports/status", (req, res) => {
  res.json(getReportsSyncStatus());
});

// Después
app.get("/api/sync/reports/status", async (req, res) => {
  res.json(await resolveSyncStatus(getReportsSyncStatus(), "reports.json"));
});
```

Tres condiciones hacen que esto funcione, y las tres son fáciles de romper más adelante:

- **El sync tiene que escribir `syncedAt` cerca del principio de la salida.** Si un cambio
  futuro lo mueve detrás de un array grande, la lectura por rango deja de encontrarlo en
  silencio y vuelves a los blancos — sin ningún error.
- **Recurre al fichero de *salida*, no a un fichero de metadatos aparte.** Un fichero de
  metadatos puede escribirse cuando el sync arranca, o no escribirse cuando el sync muere
  a mitad; el fichero de salida existe solo si el sync produjo algo de verdad.
- **Si el estado ya vive en una base de datos, sáltate todo esto.** Firestore o Cloud SQL
  ya sobreviven a los reinicios. Este patrón es para estado cuyo único rastro durable es un
  fichero.

## Cómo verificarlo

Fuerza la condición en vez de esperarla. Desplegar una revisión nueva reemplaza la
instancia, que es el mismo evento que un escalado a cero:

```bash
gcloud run services update my-service --region my-region \
  --update-env-vars "CACHE_BUST=$(date +%s)"

curl -s https://my-service.example/api/sync/reports/status
```

Si la respuesta no trae `syncedAt` y el fichero de salida sí, estás expuesto:

```bash
gcloud storage cat gs://my-bucket/reports.json | head -c 200
```

Dos marcas de tiempo que coinciden significan que el fallback funciona. Una marca en el
fichero y un blanco en el endpoint es el bug, reproducido a voluntad en menos de un minuto.

Para encontrar todas sus apariciones en un código base, busca rutas de status que no sean
`async`:

```bash
grep -rn "sync.*status" src/ | grep -v "async"
```

## Véase también

- [Un Service de Cloud Run que responde antes de terminar no es una ejecución durable](../cloudrun-job-vs-service/README.es.md) —
  el mismo ciclo de vida de instancia, visto desde el lado de quien escribe en vez del de
  quien lee.
- [Escalar un servicio de Cloud Run a cero sin tocar IAM](../cloudrun-power-control/README.es.md)
