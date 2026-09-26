# Engineering Handbook

[English](README.md) · **Español**

Lecciones de operar sistemas en producción. Cada entrada se escribió **después** de un
incidente, de una sorpresa o de una factura — nunca como tutorial.

El formato es fijo a propósito: *contexto → qué falló → por qué → el patrón → cómo
verificarlo*. Si una entrada no puede responder a «cómo compruebo si tengo este problema
ahora mismo», no se publica.

Salen de plataformas de negocio asistidas por IA en Google Cloud: pipelines de mensajería,
control de coste de LLM, Firestore a una escala incómoda, la semántica menos evidente de
Cloud Run y las políticas de organización que, sin hacer ruido, deciden qué se te permite
desplegar.

## Índice

| Entrada | Lección |
|---|---|
| [`bounded-agent-loops`](bounded-agent-loops/README.es.md)<br><sub>`agent-loops` · `correctness` · `2026`</sub> | Un guard de deduplicación que solo mira el histórico no puede ver el duplicado que está creando ahora mismo |
| [`chrome-headless-pdf-fixed-footer`](chrome-headless-pdf-fixed-footer/README.es.md)<br><sub>`chrome-headless` · `correctness` · `2026`</sub> | Un `bottom` negativo empuja el pie fijo *hacia dentro* del contenido, no fuera de la página |
| [`cloudrun-job-vs-service`](cloudrun-job-vs-service/README.es.md)<br><sub>`cloud-run` · `cloud-scheduler` · `data loss` · `2026`</sub> | Un Service de Cloud Run que responde antes de terminar no es una ejecución durable |
| [`cloudrun-power-control`](cloudrun-power-control/README.es.md)<br><sub>`cloud-run` · `cost` · `2026`</sub> | Pausa un servicio de Cloud Run escribiendo una anotación — y guarda el valor que estás pisando |
| [`cloudrun-sync-status-persistence`](cloudrun-sync-status-persistence/README.es.md)<br><sub>`cloud-run` · `correctness` · `2026`</sub> | Cloud Run pierde el estado en memoria en cada reinicio — tu endpoint de status está mintiendo |
| [`firestore-collectiongroup-indexes`](firestore-collectiongroup-indexes/README.es.md)<br><sub>`firestore` · `correctness` · `2026`</sub> | Los informes collection-group fallan a gritos por el índice y en silencio por el limit |
| [`firestore-scalability`](firestore-scalability/README.es.md)<br><sub>`firestore` · `cost` · `2026`</sub> | Un listener en tiempo real es una suscripción, por usuario, a las escrituras de todos los demás |
| [`gemini-vision-extractor`](gemini-vision-extractor/README.es.md)<br><sub>`gemini` · `outage` · `2026`</sub> | Un modelo que responde correctamente dentro de un bloque de código markdown te tumba el endpoint igual |
| [`gemini-zod-schema-pipeline`](gemini-zod-schema-pipeline/README.es.md)<br><sub>`gemini` · `correctness` · `2026`</sub> | Tres esquemas describen una misma respuesta, y el más estricto gana en silencio |
| [`jest-promisify-mock-pattern`](jest-promisify-mock-pattern/README.es.md)<br><sub>`jest` · `node` · `correctness` · `2026`</sub> | `promisify` captura la referencia a la función al importar — tu mock llega tarde |
| [`llm-output-field-normalization`](llm-output-field-normalization/README.es.md)<br><sub>`llm` · `correctness` · `2026`</sub> | El modelo es una cuarta ruta de código, y tu formateador no se ejecuta en ella |
| [`systematic-debugging`](systematic-debugging/README.es.md)<br><sub>`node` · `outage` · `2026`</sub> | Cinco deploys fallaron con un mensaje de error cierto, preciso y que apuntaba al sitio equivocado |

---

<sub>Solo patrones genéricos. Sin código de cliente, credenciales ni identificadores.</sub>
