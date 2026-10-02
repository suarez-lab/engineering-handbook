# Un secreto guardado con un salto de línea final pasa todas las comprobaciones que escribiste y falla en producción

[English](README.md) · **Español**

`secret-manager` · `correctness` · `2026`

## Contexto

Un servicio cuya credencial se genera con una herramienta de línea de comandos y se guarda
en un gestor de secretos, y que la aplicación lee como variable de entorno. Una
comprobación de token, un callback firmado o un flujo de login compara el valor guardado
con lo que envía un cliente.

## Qué falló

**Todos los clientes recibían `401` — y las comprobaciones hechas a mano decían que el
secreto era correcto.** Pasó una segunda vez en otro servicio: un inicio de sesión OAuth
fallaba y perdimos minutos comparando hashes del secreto en dos sitios que parecían idénticos.

## Por qué

Las herramientas de línea de comandos terminan su salida con un salto de línea. Encadenado
al gestor de secretos, ese salto se guarda como parte del valor:

```bash
openssl rand -hex 32 | gcloud secrets versions add my-secret --data-file=-
#                      ^ el valor guardado son 65 bytes: 64 caracteres más "\n"
```

La aplicación lee 65 bytes y los compara con los 64 que tiene el cliente. Nunca coinciden.

Lo que lo ocultó fue el propio shell. **La sustitución de comandos elimina los saltos de
línea finales**, así que cualquier comprobación escrita como `$(...)` quita en silencio
justo el byte que está mal:

```bash
# ambos lados pierden el salto, los hashes coinciden — y el secreto guardado sigue roto
[ "$(gcloud secrets versions access latest --secret=my-secret | sha256sum)" = \
  "$(printf '%s' "$EXPECTED" | sha256sum)" ] && echo "parece correcto"
```

La verificación era estructuralmente incapaz de ver el defecto que debía encontrar.

## El patrón

**Escribe el valor sin el salto de línea.**

```bash
openssl rand -hex 32 | tr -d '\n' | gcloud secrets versions add my-secret --data-file=-

# o, para un valor que ya tienes en una variable
printf '%s' "$VALUE" | gcloud secrets versions add my-secret --data-file=-
```

**Compara bytes crudos, nunca el resultado de un `$(...)`.** Mira el último byte, o cuenta:

```bash
gcloud secrets versions access latest --secret=my-secret | xxd | tail -n 1
# un "0a" al final es el salto de línea

gcloud secrets versions access latest --secret=my-secret | wc -c
# se esperan 64 para un valor hex de 32 bytes; 65 significa que hay un salto guardado
```

**Decide a propósito si la aplicación recorta.** Recortar al leer oculta esta clase de bug,
lo cual es cómodo y también es como dejas de notar que el valor guardado está mal. O recortas
en todas partes y lo documentas, o no recortas en ninguna y mantienes exacto el valor guardado.

## Cómo verificarlo

Para cada secreto que generes con un script, ejecuta la comprobación `wc -c` de arriba y
compárala con la longitud que esperas. Una diferencia de exactamente un byte es este bug.

## Véase también

- [`secrets-flag-replaces-the-list`](../secrets-flag-replaces-the-list/README.es.md) — la otra forma en que un despliegue deja un secreto mal sin ningún error.
