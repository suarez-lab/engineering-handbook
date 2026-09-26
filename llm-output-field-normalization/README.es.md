# El modelo es una cuarta ruta de código, y tu formateador no se ejecuta en ella

[English](README.md) · **Español**

`llm` · `correctness` · `2026`

## Contexto

Un servicio que responde preguntas de usuario sobre un catálogo. Los registros se traen
de una fuente de datos, se inyectan en el prompt como contexto, y el modelo devuelve
JSON estructurado — una lista de recomendaciones, cada una con un id y unos cuantos
campos de presentación.

Algunos de esos campos de presentación no le corresponde inventarlos al modelo. Un
título, un precio formateado, una etiqueta canónica: se derivan del registro mediante una
función determinista que ya existe en el código y que ya se usa en todas partes.

## Qué falló

Los títulos de la respuesta venían como cadenas crudas de origen —
`"north-district·apartment·sale"` — en vez de la forma legible que produce el formateador,
`"Apartment in North District"`.

Lo que lo hacía confuso: el formateador era demostrablemente correcto, estaba cubierto
por tests unitarios y se llamaba desde todos los sitios que construyen un título.
Auditamos tres rutas de llamada y las tres estaban bien. El bug sobrevivió a un deploy
que lo «arreglaba», porque el arreglo se aplicó a código que nunca fue el origen de esa
cadena.

La forma cruda era la cabecera unida por delimitadores que la fuente de datos de origen
guarda como clave de índice. Aparecía literalmente en el contexto del prompt. Y el valor
de ejemplo del esquema de respuesta, escrito para ilustrar el campo, resultaba parecerse
a ella:

```json
{ "title": "Area·Type·Operation" }
```

Así que el esquema no estaba describiendo el campo. Estaba demostrando el formato
equivocado, justo en el sitio al que el modelo más atención presta.

## Por qué

No había tres rutas de código que construyeran un título. Había cuatro. La cuarta es el
modelo, y el modelo no llama a tus funciones.

Cuando un valor aparece en el contexto del prompt con una forma que satisface de manera
plausible un campo del esquema, generar ese campo copiándolo es la continuación más
barata disponible. Al modelo no le cuesta nada y es localmente coherente con todo lo que
puede ver — incluido un ejemplo de esquema que se parece más a la cadena cruda que a la
salida pretendida.

El fallo es silencioso por construcción. La salida es JSON válido, pasa la validación de
esquema, el campo es una cadena no vacía y del tipo correcto. Nada aguas abajo tiene
forma alguna de saber que esa cadena en concreto se copió en lugar de derivarse. Las
instrucciones en el prompt reducen la frecuencia; no la llevan a cero, y un campo con
fuente canónica no debería tener frecuencia ninguna.

## El patrón

Decide, campo a campo, de quién es — y hazlo cumplir en el código, no en el prompt.

| Campo | Dueño |
|---|---|
| Derivable de datos canónicos (título, precio, etiqueta, URL) | Tu código. El modelo no debe producirlo. |
| Requiere inferencia (resumen, justificación, ranking) | El modelo. No hay fuente canónica. |

**Lo mejor: no pedir el campo siquiera.** Quítalo del esquema de respuesta y constrúyelo
después. Un campo que el modelo no puede emitir es un campo que no puede equivocar.

**Cuando el esquema tiene que conservarlo**, sobrescríbelo de forma determinista después
de validar y antes de que la respuesta salga del servicio:

```ts
// after schema validation, before returning to the caller
const byId = new Map(records.map(r => [r.id, r]));

for (const item of response.items) {
  // the model returns ids as strings even when the source type is numeric
  const id = typeof item.id === 'string' ? Number(item.id) : item.id;
  const record = byId.get(id) ?? byId.get(item.id as never);
  if (record) item.title = buildTitle(record);   // canonical wins, always
}
```

Dos detalles que sostienen la estructura:

- **Coacciona el tipo del id.** Un modelo al que le das ids numéricos los devolverá
  entrecomillados con frecuencia. Un `Map` indexado por números falla entonces en todas
  las búsquedas, y la pasada de normalización se convierte en un no-op que falla
  exactamente igual de silencioso que el bug que venía a arreglar.
- **No te saltes el caso sin coincidencia.** Si el id no resuelve a ningún registro, eso
  es un problema que merece un log — puede que el modelo se haya inventado un elemento
  que ni siquiera está en el contexto.

Y arregla de paso el ejemplo del esquema: un valor de ejemplo es una demostración, no
documentación. Que enseñe la salida que quieres.

## Cómo verificarlo

Pasa el pipeline sobre un puñado de registros y comprueba igualdad contra la función
determinista — no una regex, no un «parece razonable»:

```ts
for (const item of response.items) {
  const record = byId.get(Number(item.id));
  expect(item.title).toBe(buildTitle(record));   // exact, or the model authored it
}
```

Para localizar la exposición en un servicio ya existente sin ejecutar nada, pregúntate
por cada campo de tu esquema de respuesta: *¿hay ya una función en este repositorio que
calcule esto?* Todo campo cuya respuesta sea sí, y que el modelo siga teniendo permitido
emitir, es este bug esperando la entrada adecuada.

La sonda en vivo más rápida: coge un registro cuya cadena cruda de origen se diferencie
a simple vista de su salida formateada, métela en el contexto e inspecciona el campo. Si
las dos formas son idénticas para todos los registros de tus datos de prueba, el test no
puede detectar este fallo en absoluto — cambia de datos.

## Véase también

- La misma pregunta de propiedad aplica a ids, monedas y fechas en la salida
  estructurada del modelo: cualquier cosa con representación canónica debería escribirla
  código que tenga el valor canónico, descartando la versión del modelo en lugar de
  validarla.
