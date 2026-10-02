# `--set-secrets` reemplaza la lista entera — un despliegue puede borrar en silencio todos los demás secretos

[English](README.md) · **Español**

`cloud-run` · `outage` · `2026`

## Contexto

Un servicio de Cloud Run que lee varios secretos de Secret Manager, expuestos al contenedor
como variables de entorno. El despliegue vive en un fichero de build, y cada cambio que
necesita un secreto más lo añade al comando de despliegue.

## Qué falló

**El despliegue terminó bien, el servicio arrancó y una funcionalidad dejó de funcionar días
después.** Un cambio necesitaba un secreto nuevo, así que el fichero de build pasó solo ese
secreto a `--set-secrets`. La revisión nueva arrancó sana: el contenedor inició y respondió
a su health check. Nada en la salida del despliegue sugería que faltara algo.

Los secretos que *no* estaban en el comando desaparecieron de la revisión nueva. Los
caminos de código que los leen en el momento de la petición fallaron solo cuando alguien
usó ese camino. Lo descubrimos días después, por errores de usuario, no por un log de
despliegue.

## Por qué

La familia de flags `set` significa *reemplazar el conjunto entero*, no *añadir*:

| Flag | Comportamiento |
|---|---|
| `--set-secrets` | Los secretos listados pasan a ser los **únicos**. El resto se elimina. |
| `--update-secrets` | Los listados se añaden o cambian. El resto se conserva. |
| `--set-env-vars` | La misma trampa, para variables de entorno normales. |
| `--update-env-vars` | Fusiona. |

Como el contenedor arranca bien sin ellos, la plataforma no tiene nada de qué quejarse. Una
variable de entorno ausente es un error de aplicación, y la aplicación solo lo nota cuando
necesita el valor.

## El patrón

**Usa el flag que fusiona para los cambios incrementales.**

```bash
gcloud run services update my-service --region=my-region \
  --update-secrets=NEW_SECRET=new-secret-name:latest
```

**Si el pipeline tiene que usar `--set-secrets`, el fichero de build es la fuente de verdad
completa** — todos los secretos que el servicio necesita, cada vez, revisado como código:

```bash
gcloud run services update my-service --region=my-region \
  --set-secrets=SECRET_A=secret-a:latest,SECRET_B=secret-b:latest,SECRET_C=secret-c:latest
```

**Haz fallar el pipeline cuando desaparezca un nombre.** Compara los nombres antes y después:

```bash
names() {
  gcloud run services describe my-service --region=my-region --format=json \
    | jq -r '.spec.template.spec.containers[0].env[].name' | sort
}

names > /tmp/before.txt
# ... deploy ...
names > /tmp/after.txt
comm -23 /tmp/before.txt /tmp/after.txt   # nombres que existían y ya no
```

## Cómo verificarlo

Ejecuta la función `names` de arriba contra cualquier servicio que despliegues hoy y
compárala con la lista que la aplicación realmente lee. Cualquier nombre que el código use
y no aparezca en la salida es un fallo latente. Después busca `--set-secrets` y
`--set-env-vars` en tus ficheros de build y comprueba que cada uno lista todo lo que el
servicio necesita.

## Véase también

- [`cloudrun-job-vs-service`](../cloudrun-job-vs-service/README.es.md) — otro caso donde un despliegue parece sano y no lo está.
