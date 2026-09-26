# Un `bottom` negativo empuja el pie fijo *hacia dentro* del contenido, no fuera de la página

[English](README.md) · **Español**

`chrome-headless` · `correctness` · `2026`

## Contexto

Generamos PDFs a partir de HTML con Chrome headless — sin wkhtmltopdf, sin WeasyPrint,
sin ningún runtime extra que instalar o fijar:

```bash
chrome --headless --disable-gpu --no-pdf-header-footer \
       --print-to-pdf=out.pdf --print-to-pdf-no-header "file://report.html"
```

El documento necesitaba un pie en todas las páginas. `@page` le reservaba un margen
inferior.

## Qué falló

El pie se imprimía *encima* del texto del cuerpo, en todas las páginas. Ni una vez suelta,
ni en un salto de página — en todas.

Lo que parecía: un problema de apilamiento CSS o de z-index. Lo que era: el pie estaba
colocado correctamente, pero para un sistema de coordenadas que habíamos supuesto mal.

La regla culpable:

```css
@page { size: A4; margin: 22mm 18mm 26mm 18mm; }
.footer {
  position: fixed;
  bottom: -18mm;   /* "push it down into the margin" */
  left: 0;
  right: 0;
}
```

El desplazamiento negativo es la intuición de la que hay que desconfiar. Se lee como
*mueve esto por debajo del área de contenido, al blanco que he reservado* — y eso no es
lo que hace.

## Por qué

Chrome calcula la caja imprimible a partir del margen de `@page`. Los desplazamientos de
`position: fixed` se resuelven **dentro de esa caja**, no contra la hoja física.

Así que el margen no es espacio vacío al que puedas llegar poniéndote en negativo. Está
fuera del sistema de coordenadas, sin más. Un `bottom` negativo no viaja hacia el borde
del papel; viaja hacia arriba, de vuelta al flujo del contenido — que es exactamente de
donde salía el solape.

Al margen reservado se llega quedándose **en positivo y por debajo del margen**.
`bottom: 8mm` con un margen inferior de `26mm` cae dentro de la banda reservada, y sobra
sitio.

El mismo razonamiento en el eje horizontal: `left: 0` alinea con la hoja, no con la
columna de texto, así que el pie queda más ancho que el contenido que tiene encima.
Iguálalo al margen horizontal de `@page`.

## El patrón

```css
@page { size: A4; margin: 22mm 18mm 26mm 18mm; }

.footer {
  position: fixed;
  bottom: 8mm;     /* positive, and ≤ the @page bottom margin */
  left: 18mm;      /* = the @page horizontal margin, not 0 */
  right: 18mm;
  text-align: center;
  font-size: 8pt;
  border-top: 1px solid #ddd;
  padding-top: 6px;
}
```

Generalizado, para cualquier elemento `position: fixed` pensado para repetirse en todas
las páginas — pie, cabecera, marca de agua:

- El desplazamiento (`top` / `bottom`) es **positivo** y **≤ el margen de `@page` de ese lado**.
- `left` / `right` replican el margen horizontal de `@page`, salvo que quieras
  deliberadamente que el elemento ocupe toda la hoja física.

## Cómo verificarlo

Dos minutos, sin necesidad de ningún proyecto:

```bash
cat > /tmp/probe.html <<'HTML'
<style>
  @page { size: A4; margin: 22mm 18mm 26mm 18mm; }
  .footer { position: fixed; bottom: 8mm; left: 18mm; right: 18mm;
            border-top: 1px solid #000; font: 8pt sans-serif; }
  p { font: 11pt serif; }
</style>
<div class="footer">footer — page bottom</div>
<p>line</p><p>line</p><!-- repeat until it spans 3+ pages -->
HTML

chrome --headless --disable-gpu --no-pdf-header-footer \
       --print-to-pdf=/tmp/probe.pdf --print-to-pdf-no-header file:///tmp/probe.html
```

Abre el PDF y mira **la última página**, no la primera. Un pie que solapa suele verse
bien en la página 1, donde el contenido no llega abajo del todo.

Después cambia `bottom: 8mm` por `bottom: -18mm` y regenera. Si el pie aterriza sobre tu
texto, has reproducido el bug y el arreglo en la misma sentada.

## Véase también

- Los márgenes de `@page` definen la caja imprimible — casi todas las sorpresas de
  print-CSS se remontan a olvidar que cualquier otra caja se mide dentro de ella.
- [Cinco deploys fallaron con un mensaje de error cierto, preciso y que apuntaba al sitio equivocado](../systematic-debugging/README.es.md)
  — el mismo tipo de error: se aceptó una explicación plausible sin reproducirla en el motor que
  de verdad renderiza.
- [El modelo es una cuarta ruta de código, y tu formateador no se ejecuta en ella](../llm-output-field-normalization/README.es.md)
  — los supuestos de maquetación se rompen con contenido que no escribiste tú, en impresión igual
  que en una UI.
