# Un modelo que responde correctamente dentro de un bloque de código markdown te tumba el endpoint igual

[English](README.md) · **Español**

`gemini` · `outage` · `2026`

## Contexto

Una app de consumo del sector del juego (España, 2026) con un flujo de captura por foto: el
usuario fotografía un documento en papel con el móvil, la imagen viaja en base64 a una función
serverless, un modelo multimodal la lee y devuelve campos estructurados, y la app muestra un
formulario pre-rellenado para que el usuario confirme.

Esta entrada aplica a cualquier ruta de «OCR con LLM» donde la respuesta del modelo se parsea
como JSON y el resultado del parseo decide el estado HTTP.

## Qué falló

Algunas fotos producían un **error de red** en la app. No un mensaje de validación, no un «no
hemos podido leer esto» — un fallo genérico, sin formulario en pantalla y sin forma de que el
usuario metiera los datos a mano. El flujo se quedaba en un callejón sin salida.

Lo engañoso: el modelo había hecho su trabajo. La respuesta cruda registrada en los logs era
JSON bien formado con todos los campos correctamente extraídos. Solo que venía envuelta así:

````
```json
{"date": "16/07/2026", "amount": "12.50", "error": null}
```
````

`json.loads()` sobre esa cadena lanza excepción. La excepción se propagó, la función devolvió
500, y el manejador del `fetch` del frontend hizo lo que hace con un 500: mostrar un error de
red.

Se apilaron dos defectos distintos. El modelo ignora ocasionalmente el «sin markdown» — ese es
recuperable. El segundo no: **el endpoint trató un resultado de calidad de contenido como un
fallo de transporte**, y un fallo de transporte no tiene ruta de fallback en un frontend.

## Por qué

Que un modelo instruido formatee JSON dentro de un bloque de código no es un mal
funcionamiento. El código en bloques es, con enorme diferencia, la forma dominante en la que
aparece JSON en sus datos de entrenamiento, así que el bloque es la continuación de alta
probabilidad aunque el prompt diga lo contrario. Puedes bajar esa probabilidad con prompting;
no puedes llevarla a cero. Trata el bloque como salida esperada, no como un bug.

El error de fondo es una confusión de categorías en el contrato de respuesta. Aquí hay dos
tipos de «fallo» y necesitan tratamientos opuestos:

- **Fallo técnico** — el modelo no responde, la credencial está mal, la función revienta. El
  cliente debe reintentar o avisar de una caída. `5xx` es lo correcto.
- **Fallo de extracción** — la foto está borrosa, el documento no es el esperado, un campo es
  genuinamente ilegible. Esto es un *resultado normal y esperado* de ponerle una cámara a
  cualquier persona. El cliente debe caer a entrada manual. `200` con un flag es lo correcto.

Devolver `5xx` para el segundo tipo destruye el fallback, porque la rama de error del frontend
está escrita para problemas de red, no para «pídele al usuario que lo teclee».

## El patrón

```python
import json, re

PROMPT = """Extract the fields from this document as strict JSON.
Respond with the JSON object ONLY: no extra text, no markdown, no code fences.

Required shape:
{"date": "DD/MM/YYYY", "amount": "X.XX", "error": null}

Rules:
- If a field cannot be read with certainty, use null for that field.
- If the image is not the expected document, set "error" to a short description.
"""

FENCE = re.compile(r"^```(?:json)?\s*|\s*```$")

def parse_model_output(raw: str) -> dict:
    """Never raises. Returns extracted fields, or a dict carrying an error."""
    try:
        return json.loads(FENCE.sub("", raw.strip()))
    except json.JSONDecodeError as e:
        return {"error": f"unparseable model output: {e}"}

def extract(image_b64: str, mime_type: str) -> dict:
    try:
        raw = model.generate_content([PROMPT, {"inline_data": {
            "mime_type": mime_type, "data": image_b64}}]).text
        logger.info("model raw response: %s", raw)   # keep this: failures are unreproducible
        return parse_model_output(raw)
    except Exception as e:                            # noqa: BLE001 - boundary
        return {"error": f"extraction failed: {e}"}

def handler(request):
    data = extract(request.json["image_b64"], request.json["mime_type"])
    if data.get("error"):
        # Content outcome, not transport failure. 200, with an explicit fallback flag.
        return json.dumps({"ok": False, "use_manual_form": True,
                           "error": data["error"]}), 200
    return json.dumps({"ok": True, "data": data}), 200
```

Tres reglas que trascienden este incidente:

- **Limpia el bloque de código, siempre.** Una regex. No es un problema que se resuelva solo
  con el prompt.
- **El estado HTTP refleja el transporte, no el contenido.** Un modelo que declina extraer te
  ha respondido. `200` más un flag legible por máquina mantiene alcanzable la rama de fallback
  del cliente.
- **Muestra siempre un formulario de confirmación.** La salida es una lectura probabilística
  de una foto de móvil. Pre-rellénalo, deja que el usuario corrija, nunca lo escribas
  directamente en tu base de datos.

Y registra el texto crudo de la respuesta. Cuando un usuario informa de que «no leyó mi
documento», la imagen ya no suele estar y la ejecución no es reproducible; la cadena cruda es
la única evidencia que vas a tener.

## Cómo verificarlo

Salta el modelo. Dale a tu parser las formas que el modelo produce de verdad:

```python
cases = [
    '{"date": "16/07/2026", "error": null}',                  # clean
    '```json\n{"date": "16/07/2026", "error": null}\n```',    # fenced
    '```\n{"date": "16/07/2026", "error": null}\n```',        # fenced, no language
    'Here is the JSON:\n```json\n{"date": null}\n```',        # preamble
    'I cannot read this image.',                              # refusal
]
for raw in cases:
    out = parse_model_output(raw)      # must never raise
    assert "date" in out or "error" in out
```

Los dos últimos son los que importan: ninguno puede lanzar excepción, y ambos tienen que
volver como error de extracción, nunca como excepción.

Después comprueba el contrato de punta a punta con una entrada deliberadamente ilegible:

```bash
curl -s -o /dev/null -w '%{http_code}\n' -X POST "$ENDPOINT" \
  -H 'content-type: application/json' \
  -d '{"image_b64":"aGVsbG8=","mime_type":"image/jpeg"}'
```

Si eso imprime algo que no sea `200`, tus usuarios no tienen fallback manual.

## Véase también

- [Tres esquemas describen una misma respuesta, y el más estricto gana en silencio](../gemini-zod-schema-pipeline/README.es.md)
  — el siguiente fallo del mismo pipeline: el modelo extrae un campo correctamente y tu
  validador lo tira sin decir nada.
