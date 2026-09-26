# Un listener en tiempo real es una suscripción, por usuario, a las escrituras de todos los demás

[English](README.md) · **Español**

`firestore` · `cost` · `2026`

## Contexto

Un panel interno sobre Firestore. De varios cientos a unos pocos miles de usuarios,
todos autenticados, todos abriendo más o menos las mismas pantallas a la misma hora
laboral. Una de esas pantallas es una vista agregada — un mapa de calor, un ranking, un
panel de contadores — construida de la forma evidente: `onSnapshot` sobre una colección
con un `limit`, para que la vista se actualice sola según llegan documentos.

Si tu sistema tiene esa forma, esto te aplica. Si tu concurrencia se mide en decenas,
no te aplica, y deberías quedarte con el código simple.

## Qué falló

No saltó nada. Ni error de cuota, ni 429, ni página de estado en amarillo. El contador
de lecturas del panel de facturación, sin más, crecía a un ritmo que nadie sabía
explicar desde las cifras de tráfico, y la vista agregada se volvía más lenta según
avanzaba el día.

La parte engañosa: perfilar una sesión suelta hacía que la vista pareciera barata. Un
usuario que la abría leía los documentos que permitía el `limit` y nada más. El coste no
estaba en la sesión; estaba en el producto de sesiones por escrituras, y ese producto no
aparece en ninguna traza individual.

La forma del código que lo provocaba:

```javascript
// aggregate view — updates itself, therefore "free"
onSnapshot(
  query(collection(db, 'events'), orderBy('createdAt', 'desc'), limit(200)),
  (snap) => render(aggregate(snap.docs))
);
```

## Por qué

`onSnapshot` factura en dos momentos, y el segundo es el que se olvida.

1. **Al engancharse**, el listener recibe el conjunto de resultados completo. Eso es una
   lectura por documento devuelto, por listener. Una vista con `limit(200)` abierta en
   1000 clientes son 200000 lecturas, y vuelven a ser 200000 lecturas cada vez que esos
   clientes recargan.

2. **En cada escritura posterior que encaje con el query**, el documento modificado se
   entrega a *todos* los listeners enganchados, y cada entrega es una lectura. Un job de
   fondo que añade 60 documentos con 1000 listeners enganchados factura 60000 lecturas
   por un lote de 60 escrituras.

Así que el coste no lo gobierna cuántos documentos tienes ni cuántos usuarios tienes.
Lo gobierna su producto — `documentos_modificados × listeners_enganchados` — y ese
término solo existe en tiempo de ejecución, que es por lo que nunca aparece en una
revisión de código.

Lo segundo que conviene nombrar: casi ninguna vista agregada necesita tiempo real de
verdad. Un mapa de calor con cuatro horas de retraso sigue siendo un mapa de calor. Un
ticker de precios no. El `onSnapshot` se eligió porque era el código más corto, no
porque el producto pidiera actualizaciones en vivo.

## El patrón

Tres reglas, ordenadas por cuánto ahorran.

**1. Precalcula el agregado en un único documento.** Un job programado lee la colección
una vez, en el servidor, y escribe un documento pequeño. Cada cliente hace un `getDoc`.
Las lecturas pasan de `listeners × documentos` a `listeners × 1`.

```typescript
// server — runs on a schedule, not per request
export async function computeSnapshot() {
  const since = Timestamp.fromMillis(Date.now() - 72 * 60 * 60 * 1000);
  const snap = await db.collection('events')
    .where('createdAt', '>=', since)
    .orderBy('createdAt', 'desc')
    .get();

  const byBucket = new Map<string, number>();
  for (const doc of snap.docs) {
    const d = doc.data();
    if (d.kind !== 'SEARCH') continue;          // extra filters in memory:
    const key = (d.bucket ?? '').trim();        // one index, not a combinatorial set
    if (key) byBucket.set(key, (byBucket.get(key) ?? 0) + 1);
  }

  await db.doc('aggregates/heatmap').set({
    buckets: [...byBucket].map(([bucket, count]) => ({ bucket, count })),
    total: snap.size,
    generatedAt: FieldValue.serverTimestamp(),
    windowHours: 72,
  });
}
```

**2. Separa la suscripción por tipo de vista, no por página.** Tiempo real y agregado
son contratos distintos y no deben compartir listener. La vista de lista — filtrada,
paginada, propia de cada usuario — se queda con `onSnapshot`. La vista agregada lee el
documento precalculado y no engancha nada.

**3. Haz explícita la obsolescencia, con fallback.** Un documento precalculado que deja
de regenerarse en silencio es peor que la versión cara, porque está mal y además calla.
Comprueba la edad al leer y cae al fallback en lugar de pintar datos viejos como
frescos.

```javascript
const doc = await getDoc(docRef(db, 'aggregates', 'heatmap'));
const ageMs = Date.now() - (doc.data()?.generatedAt?.toMillis() ?? 0);

// job runs every 4h; allow one missed run, then stop trusting it
if (!doc.exists() || ageMs > 5 * 60 * 60 * 1000) return renderFromLiveQuery();
renderFromSnapshot(doc.data());
```

La ventana de tolerancia debe ser estrictamente mayor que el intervalo del job y
estrictamente menor que dos intervalos. Igual al intervalo, y cualquier ejecución lenta
dispara el fallback; sin límite, y un job muerto pasa desapercibido una semana.

## Cómo verificarlo

No hace falta una prueba de carga. Dos comprobaciones, ambas en menos de cinco minutos.

**Localiza la exposición.** Cualquier listener con un `limit` grande sobre una colección
compartida es candidato:

```bash
grep -rn "onSnapshot" src/ | grep -v "\.test\."
# then, for each hit: is the query user-specific, or does every user attach the same one?
```

Un listener cuyo query no contiene ningún identificador de usuario lo engancha todo el
mundo, y su coste se multiplica.

**Observa el fan-out directamente.** Registra los cambios entregados por snapshot, abre
la vista en dos pestañas y escribe un documento que encaje:

```javascript
onSnapshot(q, (snap) => {
  console.log('delivered docs:', snap.docChanges().length, 'from cache:', snap.metadata.fromCache);
});
```

Una escritura, dos pestañas, dos entregas, dos lecturas facturadas. Multiplícalo por tu
concurrencia real y por tu ritmo real de escritura — ese producto es la cifra que hay
que comparar contra la versión precalculada, que es una lectura por cliente y carga de
página, sea cual sea el ritmo de escritura.

## Véase también

- El mismo razonamiento de fan-out aplica a cualquier suscripción por push que se
  facture por entrega, no solo a Firestore.
- Un job programado que regenera un documento compartido necesita su propio seguro
  contra solapamientos: dos ejecuciones simultáneas del mismo job calcularán y
  escribirán dos veces tan contentas. Un documento de lock transaccional con TTL es el
  arreglo más pequeño; un leer-luego-escribir sin transacción no es un arreglo en
  absoluto.
