# Los informes collection-group fallan a gritos por el índice y en silencio por el limit

[English](README.md) · **Español**

`firestore` · `correctness` · `2026`

## Contexto

Una aplicación multi-tenant que guarda los registros de cada tenant bajo su propia ruta
de documento — `tenants/{tenantId}/deals/{dealId}` — y que necesita informes
cross-tenant: embudos, conteos de actividad, velocidad. La herramienta natural es
`collectionGroup('deals')` con un filtro de igualdad por `tenantId`.

Ahí esperan dos fallos distintos. Uno te para en la puerta. El otro te deja pasar y te
entrega un número equivocado.

## Qué falló

**El ruidoso.** Una query collection-group de un solo campo reventó en la primera
llamada:

```ts
db.collectionGroup('deals').where('tenantId', '==', t).get();
// FAILED_PRECONDITION: the query requires an index
```

La parte engañosa es lo que pasa después. Los índices compuestos ya existían y estaban
`READY`, el enlace de índice sugerido de la consola no resolvía el problema, y
`firebase deploy --only firestore:indexes` decía que todo había ido bien. El índice que
pedía la query no era un índice compuesto en absoluto.

Hay una variante peor del mismo error, porque el índice *sí* existe: un índice compuesto
collection-group desplegado con el campo de rango en `DESCENDING`, consultado con un
`.orderBy(field)` pelado — que por defecto es `ASCENDING`. Índice `READY`, campos que
coinciden, y `FAILED_PRECONDITION` igualmente.

**El callado.** Un informe que nunca lanzaba excepción, iba rápido y devolvía cifras
plausibles pero bajas. Sin error, sin warning, sin aviso de truncado:

```ts
const snap = await db.collectionGroup('deals')
  .where('tenantId', '==', t)
  .limit(5000)                                  // no date range in the query
  .get();
const rows = snap.docs.filter(d => d.createdAt >= from && d.createdAt <= to);
```

Para cualquier tenant por debajo de 5000 registros esto es correcto. Y sigue siendo
correcto en staging, en los tests y durante los primeros meses de producción. Después el
tenant más grande cruza el tope y el informe empieza a mentir — a ese en concreto, y a
nadie más.

## Por qué

**El índice.** Firestore indexa un campo automáticamente en scope `COLLECTION`. *No* lo
indexa automáticamente en scope `COLLECTION_GROUP`; eso viene desactivado por defecto, y
es un índice de campo único, así que ningún índice compuesto puede satisfacerlo. La
configuración de índices de campo único vive en otra superficie de API —
`fieldOverrides` — que las CLI de `gcloud` y `firebase` no exponen para
`queryScope=COLLECTION_GROUP`. De ahí: un índice realmente ausente que ninguna CLI puede
crear y que no arregla ninguna cantidad de despliegues de índices compuestos.

Declarar un `fieldOverride` además *reemplaza* la indexación automática de ese campo, así
que las entradas de scope `COLLECTION` que tenías gratis hay que volver a declararlas
junto a la que de verdad querías.

**La dirección de ordenación.** La dirección de un índice es un contrato exacto con
`.orderBy()`, no una pista. Si otra query creó el índice en `DESCENDING` primero, un
`.orderBy()` en `ASCENDING` se queda sin índice — aunque una persona que lea los dos
diría que son «el mismo índice».

**El tope silencioso.** El servidor aplica `.limit(n)` *antes* de que corra tu filtro en
memoria, y los documentos que devuelve son un subconjunto arbitrario, no los más
recientes salvo que hayas pedido un orden. Filtrar después del tope calcula, por tanto,
un agregado correcto sobre la población equivocada. Firestore no tiene motivo para
avisar: la query que le diste tuvo éxito exactamente como la especificaste.

## El patrón

**Mete dentro de la query todos los filtros que acoten la población.** El `.limit()`
deja de ser un filtro y pasa a ser un cortacircuitos que debe anunciarse al saltar.

```ts
const HARD_CAP = 50000;

let q = db.collectionGroup('deals').where('tenantId', '==', t);
if (status) q = q.where('status', '==', status);
if (from)   q = q.where('createdAt', '>=', Timestamp.fromDate(from));
if (to)     q = q.where('createdAt', '<=', Timestamp.fromDate(to));
if (from || to) q = q.orderBy('createdAt', 'desc');   // direction explicit — never the default
q = q.limit(HARD_CAP);

const snap = await q.get();
if (snap.size >= HARD_CAP) {
  console.warn(`[reports] hard cap hit (${snap.size}); result may be truncated`);
}
```

La excepción es un filtro de igualdad opcional que duplicaría tu combinatoria de índices
— uno con él, otro sin él. Una vez que el rango ha acotado el conjunto, ese filtra en
memoria: es exacto, barato y no cuesta ningún índice.

```ts
for (const doc of snap.docs) {
  if (!doc.ref.path.startsWith(`tenants/${t}/`)) continue;  // defensive: collection-group spans tenants
  const d = doc.data();
  if (pipelineId && d.pipelineId !== pipelineId) continue;
  // ...
}
```

Esa comprobación de ruta no es paranoia. Un collection group casa con *todas* las
subcolecciones con ese id, estén donde estén en la base de datos. Si alguna vez se
escribe un documento fuera de la ruta esperada, o falta `tenantId`, acaba en el informe
de otro tenant.

**Para el índice en sí**, elige por la forma de la query:

```bash
# Equality + range/order → composite, COLLECTION_GROUP scope. The CLI can do this.
# Equalities first, range/order last.
gcloud firestore indexes composite create \
  --collection-group=deals --query-scope=COLLECTION_GROUP \
  --field-config=field-path=tenantId,order=ascending \
  --field-config=field-path=createdAt,order=descending

# Single field, no range → fieldOverride via the REST API. The CLI cannot do this.
curl -X PATCH \
  "https://firestore.googleapis.com/v1/projects/PROJECT/databases/(default)/collectionGroups/deals/fields/tenantId?updateMask.fieldPaths=indexConfig" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"indexConfig":{"indexes":[
        {"queryScope":"COLLECTION","fields":[{"fieldPath":"tenantId","order":"ASCENDING"}]},
        {"queryScope":"COLLECTION","fields":[{"fieldPath":"tenantId","order":"DESCENDING"}]},
        {"queryScope":"COLLECTION","fields":[{"fieldPath":"tenantId","arrayConfig":"CONTAINS"}]},
        {"queryScope":"COLLECTION_GROUP","fields":[{"fieldPath":"tenantId","order":"ASCENDING"}]}
      ]}}'
```

Las tres entradas `COLLECTION` son la indexación automática que estás reemplazando.
Quítalas y rompes todas las queries normales sobre ese campo.

## Cómo verificarlo

**¿Hay algún informe filtrando después de un tope?** Esta es la búsqueda que encuentra el
bug silencioso, y es un solo comando:

```bash
grep -rn -A6 "collectionGroup(" src/ | grep -B3 "\.filter("
```

Lee cada coincidencia y hazte una sola pregunta: *¿puede esta colección superar alguna
vez el limit para un solo tenant?* Si la respuesta es sí, el informe ya está mal para ese
tenant, o lo estará.

**¿Es observable el truncado?** Baja temporalmente `HARD_CAP` por debajo del número de
documentos de un tenant conocido y lanza el informe. Si no se registra ningún warning y
quien llama no recibe ninguna marca de truncado, el cortacircuitos es decorativo —
arregla eso antes de tocar índices.

**¿La dirección del índice es la que supone tu código?** No la deduzcas del fichero de
configuración; lee lo que está desplegado de verdad:

```bash
gcloud firestore indexes composite list --format="table(name,queryScope,fields)"
```

Después haz que cada `.orderBy()` declare su dirección explícitamente, acorde con esa
salida.

Dos notas operativas ya que estás. `gcloud` y `firebase` mantienen almacenes de
credenciales separados — reautenticar uno no hace nada por el otro, y para todo lo
anterior te basta con `gcloud`. Y una creación de índice cuyo sondeo en el cliente se
muere puede haber tenido éxito perfectamente en el servidor: lista antes de reintentar.
Espera a `READY`, nunca a `CREATING`, antes de decirle a nadie que el informe funciona.

## Véase también

- Las vistas agregadas sobre colecciones compartidas tienen un modo de fallo de coste
  además de uno de corrección — precalcúlalas en vez de suscribir a cada cliente.
