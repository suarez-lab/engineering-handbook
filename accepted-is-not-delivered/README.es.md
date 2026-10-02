# Un `200` de una pasarela de mensajería no es entrega — valida el destinatario donde lo guardas

[English](README.md) · **Español**

`messaging` · `correctness` · `2026`

## Contexto

Un sistema que envía notificaciones a través de una pasarela de WhatsApp de terceros a
números de teléfono que vienen de configuración o de un formulario — no de un webhook
entrante. Los números los teclea una persona, en el formato que usa localmente.

## Qué falló

**Las notificaciones dejaron de llegar y, durante días, nada lo decía en ningún sitio.** La
aplicación registraba éxito en cada envío. La pasarela respondía `200`. No había error, ni
excepción, ni estado fallido. Los mensajes quedaban en estado `pending` de forma permanente
y el dueño del destinatario concluyó que el producto no funcionaba.

Lo engañoso: la consulta de contacto para el mismo número respondía `valid`.

## Por qué

Dos endpoints de la misma pasarela tratan la misma entrada de forma distinta:

- La **consulta de contacto** normaliza el número — añade el prefijo de país que falta — y
  lo da por válido, devolviendo el identificador normalizado.
- El endpoint de **envío** **no** normaliza. Ante un número en formato local, crea un chat
  con ese identificador malformado, acepta el mensaje y lo deja en cola para siempre.

Así, "válido" en la consulta y "aceptado" en el envío son ciertos a la vez, y ninguno
significa que el mensaje pueda llegar a un teléfono. El comportamiento de la aplicación es
idéntico en el caso bueno y en el malo, y eso es lo que lo vuelve silencioso.

La única señal fiable es la que la consulta *devuelve*: **el identificador normalizado es
distinto de la entrada.** La consulta nunca dice "inválido" para estos números, así que
esperar un estado que lo diga es esperar para siempre.

## El patrón

**Valida en el endpoint que guarda el número, no en el que lo usa.** Cuando llegas a enviar,
la persona que lo tecleó ya no está.

```ts
async function saveRecipient(raw: string) {
  const digits = raw.replace(/\D/g, "");
  const res = await gateway.checkContact(digits);   // la consulta normaliza
  const normalized = res.id.replace(/\D/g, "");

  if (normalized !== digits) {
    // la pasarela tuvo que corregirlo: guarda lo que devolvió, nunca lo tecleado
    return store(normalized);
  }
  return store(digits);
}
```

**Trata `pending` como un estado con plazo.** Todo lo que siga pendiente pasados unos
minutos es un fallo y necesita su propia alerta — ver
[alertar en el camino de respaldo](../../reference-architectures/alert-on-the-fallback-path.es.md).

```sql
-- mensajes aceptados por la pasarela pero nunca confirmados
SELECT id, recipient, created_at
FROM messages
WHERE status = 'pending' AND created_at < now() - interval '10 minutes';
```

## Cómo verificarlo

Toma cada número de destinatario que el sistema tenga guardado y pásalo por la consulta.
Cualquiera cuyo identificador devuelto difiera del guardado ya está fallando en silencio.
Después mira la antigüedad del mensaje `pending` más viejo — si se mide en días, llevas
tiempo perdiendo mensajes sin saberlo.

## Véase también

- [Alertar en el camino de respaldo](../../reference-architectures/alert-on-the-fallback-path.es.md)
