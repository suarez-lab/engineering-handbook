# Un guard de deduplicación que solo mira el histórico no puede ver el duplicado que está creando ahora mismo

[English](README.md) · **Español**

`agent-loops` · `correctness` · `2026`

## Contexto

Un bucle programado en una plataforma de financiación al consumo (Latinoamérica, 2026): en
cada ciclo lee el conjunto de cuentas que toca contactar, las pasa por un motor de reglas y
empuja las acciones resultantes a una cola de mensajería saliente.

Cada acción lleva una clave de idempotencia — algo como `D15_2026-07-16` — y, antes de
encolar, un guard hace la pregunta obvia: *¿esta clave ya se envió?* La responde consultando
el histórico persistido de mensajes enviados.

Esa es la forma de sistema a la que aplica esta entrada: **una sola vuelta del bucle puede
emitir más de una acción, y el guard contra la repetición vive en almacenamiento durable.**

## Qué falló

Algunos contactos recibieron el mismo recordatorio dos veces, con minutos de diferencia, el
mismo día.

La parte engañosa: **el guard funcionaba correctamente.** Estaba activo, se ejecutó sobre las
dos acciones y devolvió «no enviado previamente» las dos veces — con razón. No había registro
histórico que encontrar. Nada en los logs parecía un bypass, una carrera contra la base de
datos ni un reintento.

El primer instinto fue sospechar de la fuente de datos: una fila de cuenta duplicada, el
scheduler disparando dos veces, dos workers cogiendo el mismo lote. Se comprobaron las tres.
Las tres estaban limpias. La cola simplemente contenía dos filas con la misma clave de
idempotencia, escritas con milisegundos de diferencia, por el mismo proceso, en la misma
vuelta.

## Por qué

Dos reglas se solapaban en un valor frontera. Una disparaba con `days_overdue == 15`; otra,
añadida después para el seguimiento recurrente, disparaba con `days_overdue >= 15`. Las dos
usaban la misma plantilla, así que las dos derivaban la misma clave de idempotencia. Justo el
día 15 — y solo el día 15 — coincidían las dos.

El guard comparaba contra el histórico **persistido**. Las dos acciones nacían dentro de la
misma iteración del mismo bucle, antes de que ninguna se hubiera persistido. Ninguna estaba
en el histórico, porque el histórico se escribe *después* de que el bucle termine de decidir.

La deduplicación histórica y la intra-ciclo son problemas distintos. La primera pregunta
«¿alguna vuelta pasada ya hizo esto?»; la segunda pregunta «¿*esta* vuelta ya decidió hacer
esto?». Una consulta al almacén solo puede responder la primera. La segunda necesita estado
que viva dentro de la vuelta.

Al arreglar el primer defecto apareció un segundo. La reparación ingenua — guardar un set con
las claves vistas en el lote actual y saltarse las repetidas — descartaba trabajo legítimo en
silencio, porque la clave no es única entre entidades. Muchos contactos comparten
`D15_2026-07-16`. Deduplicar solo por clave descarta los mensajes de otras personas. **La
identidad usada para deduplicar tiene que ser compuesta: `(entidad, clave, canal)`.**

## El patrón

```js
// Two layers, and a composite identity in both.
async function planCycle(accounts, rules, history) {
  const planned = [];
  const seenThisCycle = new Set();          // layer 2: intra-cycle

  for (const account of accounts) {
    for (const rule of rules) {
      if (!rule.matches(account)) continue;

      const action = rule.buildAction(account);
      // Identity includes the entity. The key alone is shared across entities.
      const identity = `${action.entityId}|${action.idempotencyKey}|${action.channel}`;

      if (await history.alreadySent(identity)) continue;  // layer 1: historical
      if (seenThisCycle.has(identity)) continue;          // layer 2: intra-cycle

      seenThisCycle.add(identity);
      planned.push(action);
    }
  }
  return planned;
}
```

Tres reglas que generalizan más allá de este incidente:

- **Dos capas, siempre.** Una comprobación durable y una comprobación dentro de la vuelta.
  Cualquiera de las dos por su cuenta deja un agujero, y los agujeros están en lados opuestos.
- **Identidad compuesta en todas partes.** La misma cadena tiene que usarla la comprobación
  histórica, el set de la vuelta y el merge que persiste el lote. Si cualquiera de los tres
  usa una identidad más estrecha, o está dejando pasar duplicados o se está comiendo trabajo
  válido.
- **Sospechar del solape de reglas antes que de los datos.** Los duplicados en un valor
  frontera (`== N` junto a `>= N`) son un problema de diseño de predicados, no de ingesta.
  Revisa los predicados primero; es una comprobación de cinco minutos y aquí era la respuesta.

## Cómo verificarlo

No hace falta el bucle de producción. Corre una vuelta sobre una entrada sintética construida
para caer en la frontera, y comprueba el plan:

```js
const accounts = [{ entityId: 'a1', daysOverdue: 15 }, { entityId: 'a2', daysOverdue: 15 }];
const rules = [
  { matches: a => a.daysOverdue === 15,  buildAction: a => act(a) },
  { matches: a => a.daysOverdue >= 15,   buildAction: a => act(a) },
];
const act = a => ({ entityId: a.entityId, idempotencyKey: 'D15_2026-07-16', channel: 'wa' });

const planned = await planCycle(accounts, rules, { alreadySent: async () => false });

// Exactly one action per entity — not two, and not one in total.
assert.equal(planned.length, 2);
assert.equal(new Set(planned.map(p => p.entityId)).size, 2);
```

Importan las dos comprobaciones. `length === 2` caza el bug original de duplicados; la de
entidades distintas caza el arreglo demasiado entusiasta que deduplica solo por clave. Un test
que solo mire la primera pasará tan contento sobre un bucle que ha empezado a tragarse los
mensajes de otras personas.

Contra una cola viva, el equivalente es una consulta de agrupación sobre las filas encoladas
recientemente por `(entidad, clave, canal)` con `count > 1`. Si devuelve algo, falta la
segunda capa o su identidad es demasiado estrecha.

## Véase también

- [Bucle autónomo acotado](../../reference-architectures/bounded-autonomous-loop.es.md) — el
  contrato general del que este incidente es una instancia. El invariante 5 (idempotencia y
  exclusión mutua) y el invariante 2 (observación fresca) son los dos que este bucle violó.
  Lee la arquitectura para la forma; lee esta entrada para ver cómo se ve la violación en un
  log.
- [Causa raíz primero](../systematic-debugging/README.es.md) — por qué «revisa la fuente de
  datos» era el primer movimiento equivocado aquí, y qué hacer en su lugar.
