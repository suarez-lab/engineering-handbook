# Tres esquemas describen una misma respuesta, y el más estricto gana en silencio

[English](README.md) · **Español**

`gemini` · `correctness` · `2026`

## Contexto

Un asistente del sector inmobiliario (Latinoamérica, 2026) que responde consultas con datos
estructurados: el modelo devuelve un objeto JSON bajo decodificación restringida, el backend
lo valida con un esquema Zod, y el frontend renderiza el objeto validado.

Esta entrada aplica a cualquier pipeline donde la respuesta de un LLM pasa por un validador en
tiempo de ejecución antes de llegar a la interfaz — es decir, si lo estás haciendo bien, a
todos.

## Qué falló

Se añadió un campo nuevo a las respuestas del asistente: un enlace a cada anuncio. El trabajo
parecía trivial — ampliar el prompt, ampliar el esquema de respuesta del modelo, desplegar.

En la interfaz el campo llegaba `undefined`. Siempre.

Lo engañoso: **todos los componentes reportaban éxito.** La salida cruda del modelo contenía
el campo, correctamente relleno. No se lanzaba ninguna excepción. No se registraba ningún
error de validación. No aparecía nada en el rastreador de errores. La petición devolvía `200`
con un cuerpo bien formado que simplemente no tenía el campo dentro.

La primera hipótesis fue el modelo — problema de prompt, de decodificación, de caché. No era
ninguno. El valor se generaba y, acto seguido, lo borraba nuestro propio código, a propósito y
en silencio.

## Por qué

Un esquema de respuesta no es un artefacto. Son tres, mantenidos en tres sitios distintos, y
tienen que coincidir:

1. **El `responseSchema` del modelo** — restringe lo que el modelo puede emitir.
2. **El esquema del validador** (Zod, Pydantic, el que uses) — decide qué sobrevive hasta la
   aplicación.
3. **El ejemplo JSON dentro del prompt** — en la práctica, la señal más fuerte sobre la forma
   de la salida. Un campo descrito en prosa pero ausente del ejemplo a menudo simplemente no
   se produce.

Aquí, los artefactos 1 y 3 tenían el campo y el artefacto 2 no. Y el comportamiento por
defecto de la mayoría de validadores de objetos es **descartar** las claves desconocidas, no
rechazarlas. Es un valor por defecto razonable — es lo que hace seguros a los validadores
frente a campos inyectados — pero significa que una clave no declarada no produce error, ni
aviso, ni rastro. Es un borrado silencioso.

Cada artefacto falla de forma distinta, y por eso el síntoma confunde:

| Falta en | Síntoma |
|---|---|
| Ejemplo del prompt | El modelo suele omitir el campo; la salida parece «incompleta» |
| Esquema de respuesta del modelo | El modelo puede emitirlo, o tener prohibido emitirlo |
| Esquema del validador | El modelo lo emite bien; desaparece entre backend e interfaz |

Una variante de la misma clase: un campo tipado `NUMBER` en el esquema del modelo cuando los
valores reales son identificadores alfanuméricos de fuentes de terceros (`ABC1011102482`). El
modelo devuelve un string, el validador espera un número, y se rechaza la respuesta *entera* —
un fallo ruidoso con la misma raíz: un tipo declarado en un sitio que no cuadra con los datos
de otro.

## El patrón

Declara los tres juntos, en un mismo fichero, para que no se pueda añadir un campo a uno y
olvidarlo en los otros:

```ts
import { z } from 'zod';
import { SchemaType } from '@google/generative-ai';

// 1. What the model is allowed to emit.
export const MODEL_SCHEMA = {
  type: SchemaType.OBJECT,
  properties: {
    id:          { type: SchemaType.STRING },   // STRING accepts numeric-looking ids too
    title:       { type: SchemaType.STRING },
    listing_url: { type: SchemaType.STRING },
  },
};

// 2. What survives into the application.
export const ResponseSchema = z.object({
  id:          z.union([z.number(), z.string()]),   // external ids are not always numeric
  title:       z.string(),
  listing_url: z.string().optional(),               // optional, not absent
});

// 3. What the prompt shows. Real-looking values, never empty ones.
export const PROMPT_EXAMPLE = JSON.stringify({
  id: 'ABC1011102482',
  title: 'Two-bedroom apartment',
  listing_url: 'https://example.com/listing/123',
}, null, 2);
```

Dos reglas que conviene interiorizar:

- **Un campo nuevo son tres ediciones, nunca una.** Si tu checklist tiene una línea, está mal.
- **Usa `STRING` en el esquema del modelo para cualquier identificador que venga de fuera de
  tu sistema.** Los ids externos son opacos. `STRING` acepta las dos formas; `NUMBER` rechaza
  media realidad y se lleva por delante la respuesta completa.

## Cómo verificarlo

Los tres artefactos viven en el mismo fichero, así que compara sus conjuntos de claves
directamente — sin llamar al modelo, sin red, ejecutable como test unitario:

```ts
const modelKeys  = new Set(Object.keys(MODEL_SCHEMA.properties));
const zodKeys    = new Set(Object.keys(ResponseSchema.shape));
const promptKeys = new Set(Object.keys(JSON.parse(PROMPT_EXAMPLE)));

const diff = (a: Set<string>, b: Set<string>) => [...a].filter(k => !b.has(k));

assert.deepEqual(diff(modelKeys, zodKeys),    [], 'model emits fields the validator drops');
assert.deepEqual(diff(zodKeys, modelKeys),    [], 'validator expects fields the model cannot emit');
assert.deepEqual(diff(modelKeys, promptKeys), [], 'schema declares fields the prompt never shows');
```

Añádelo una vez y todo campo futuro queda comprobado gratis. Si ahora mismo estás persiguiendo
un campo que desaparece, la versión de sesenta segundos es registrar la salida cruda del
modelo junto al objeto validado y comparar las claves — la diferencia entre ambos es tu
respuesta, y señala al validador, no al modelo.

## Véase también

- [Un modelo que responde correctamente dentro de un bloque de código markdown te tumba el endpoint igual](../gemini-vision-extractor/README.es.md)
  — el fallo anterior del mismo pipeline, donde la salida del modelo ni siquiera llega al
  validador.
