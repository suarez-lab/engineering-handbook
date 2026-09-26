# A negative `bottom` pushes a fixed footer *into* your content, not off the page

**English** · [Español](README.es.md)

`chrome-headless` · `correctness` · `2026`

## Context

We generate PDFs from HTML with Chrome headless — no wkhtmltopdf, no WeasyPrint,
no extra runtime to install or pin:

```bash
chrome --headless --disable-gpu --no-pdf-header-footer \
       --print-to-pdf=out.pdf --print-to-pdf-no-header "file://report.html"
```

The document needed a footer on every page. `@page` reserved a bottom margin for it.

## What failed

The footer printed *on top of* the body text, on every page. Not once, not at a page
break — everywhere.

What it looked like: a CSS stacking or z-index problem. What it was: the footer was
positioned correctly, for a coordinate system we had guessed wrong.

The offending rule:

```css
@page { size: A4; margin: 22mm 18mm 26mm 18mm; }
.footer {
  position: fixed;
  bottom: -18mm;   /* "push it down into the margin" */
  left: 0;
  right: 0;
}
```

The negative offset is the intuition to distrust. It reads as *move this below the
content area, into the white space I reserved* — and that is not what it does.

## Why

Chrome computes the printable box from the `@page` margin. `position: fixed` offsets
are resolved **inside that box**, not against the physical sheet.

So the margin is not empty space you can reach by going negative. It is outside the
coordinate system entirely. A negative `bottom` does not travel toward the paper edge;
it travels back up into the content flow — which is exactly where the overlap came from.

The reserved margin is addressed by staying **positive and smaller than the margin**.
`bottom: 8mm` with a `26mm` bottom margin sits in the reserved band, with room to spare.

Same reasoning for the horizontal axis: `left: 0` aligns to the sheet, not the text
column, so the footer hangs wider than the content above it. Match the `@page`
horizontal margin instead.

## The pattern

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

Generalised, for any `position: fixed` element meant to repeat on every page — footer,
header, watermark:

- The offset (`top` / `bottom`) is **positive** and **≤ the `@page` margin on that side**.
- `left` / `right` mirror the `@page` horizontal margin, unless you deliberately want
  the element to span the full physical sheet.

## How to verify

Two minutes, no project needed:

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

Open the PDF and check **the last page**, not the first. A footer that overlaps often
looks fine on page 1, where the content does not reach the bottom.

Then flip `bottom: 8mm` to `bottom: -18mm` and regenerate. If the footer lands on your
text, you have reproduced the bug and the fix in the same sitting.

## See also

- `@page` margins define the printable box — most print-CSS surprises trace back to
  forgetting that every other box is measured inside it.
- [Five deploys failed on an error message that was true, precise, and pointing at the wrong thing](../systematic-debugging/README.md)
  — same shape of mistake: a plausible explanation was accepted without reproducing it in the
  engine that actually renders.
- [The model is a fourth code path, and your formatter does not run on it](../llm-output-field-normalization/README.md)
  — layout assumptions break on content you did not author, in print exactly as in a UI.
