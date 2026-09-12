---
name: card-deck
description: Build or extend a print-ready physical card deck (front/back cards on A4, optional fold-and-glue tuck box) in the style of Gapplings — a single self-contained HTML file, no build step. Use when asked to create a new card deck, add/edit cards, design a tuck box, recolour a deck, or export a deck to PDF for this project or a sibling a11y-intuition-lab project.
---

# Card Deck Authoring

How to build a print-ready card deck like Gapplings: a single
self-contained HTML file that renders a print layout in the browser
(9 cards per A4 sheet, front + back, optional fold-and-glue box), which you
then export to PDF. Follow it top to bottom for a new deck, or jump to a
section to extend an existing one.

Pitfalls worth knowing before you start (details in their sections below):
front/back pages that must be **interleaved**, not grouped, or automatic
duplex printing pairs the wrong sides together (§9); a Chrome headless
`--print-to-pdf` bug that silently scales/pads the *entire* PDF once a
document crosses a page-count threshold (§12); an icon-cropping algorithm
for extracting individual icons from a flattened reference mockup (§7);
exact paper-fraction math for tiling A5/A6/A7 inserts on an A4 sheet (§11);
and the difference between "page margin" and "cards inset from the edge",
which are not the same thing (§11).

## 1. What you're building

One HTML file (e.g. `index.html`) with:

- A `<style>` block defining the print geometry and card look.
- A `<script>` block with plain data objects (one per card) and small
  template functions that turn data into HTML.
- No build step, no framework, no external JS. Fonts load from Google Fonts;
  everything else is local (`icons/*.png`).
- Print output: several `.page` divs, each exactly one A4 sheet, laid out
  edge-to-edge with `page-break-after:always` so the browser (or a headless
  print) paginates them 1:1. Front and back pages must be **interleaved**
  in that page sequence, one pair per physical sheet — see §9, and don't
  skip it even for a one-page deck extension, it's the single most common
  way a deck silently breaks double-sided printing.

The user opens the file and prints (`Ctrl/Cmd+P → Margins: None → Background
graphics: On → Paper: A4`), or you export a PDF for them (§12).

## 2. Base skeleton

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>My Deck — Print Cards</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Poppins:wght@600;700;800&family=Nunito+Sans:wght@400;600;700;800&display=swap" rel="stylesheet">
<style>
  :root{
    --accent:#1c516b;       /* primary brand colour: borders, titles */
    --accent-dark:#103c52;  /* darker shade: body headings, questions */
    --accent-fill:#557a90;  /* muted mid-tone, used for icon art fills */
    --cream:#ffffff;        /* card background */
    --card-w:63mm;
    --card-h:88mm;
  }
  *{box-sizing:border-box;}
  html,body{margin:0;padding:0;background:#e9e9e9;}
  body{font-family:'Nunito Sans','Segoe UI',Arial,sans-serif;}

  .toolbar{
    position:sticky;top:0;z-index:10;
    background:#222;color:#fff;padding:10px 16px;
    font:14px/1.4 system-ui,sans-serif;
    display:flex;gap:16px;align-items:center;flex-wrap:wrap;
  }
  @media print{ .toolbar{display:none !important;} }

  @page{ size:A4; margin:0; }

  .page{
    position:relative;
    width:210mm;height:297mm;
    background:var(--cream);
    margin:12px auto;              /* screen only: visual gap between sheets */
    box-shadow:0 0 8px rgba(0,0,0,.25);
    overflow:hidden;
    page-break-after:always;
  }
  @media print{ .page{ margin:0; box-shadow:none; } }

  .page-label{                     /* screen-only reference caption */
    position:absolute; top:6mm; left:0; width:100%;
    text-align:center; color:#999; font-size:8pt; letter-spacing:.5px;
  }
  @media print{ .page-label{ display:none; } }
</style>
</head>
<body>
<div class="toolbar">
  <b>My Deck — print</b>
  <span>Ctrl/Cmd+P → Margins: <b>None</b> → Background graphics: <b>On</b> → Paper: <b>A4</b></span>
</div>
<div id="app"></div>
<script>
  // card data + render functions go here (§4–§6)
  // document.getElementById('app').innerHTML = ... (§4)
</script>
</body>
</html>
```

Everything below extends this skeleton.

## 3. The 3×3 grid (9 cards per A4 sheet)

Standard mini-card size is **63×88 mm** (3 columns × 3 rows tiles an A4 sheet
with generous outer margin). CSS Grid, no gap — cards touch edge to edge and
a single dashed outline marks the whole block for cutting:

```css
.grid{
  position:absolute;
  top:16.5mm; left:10.5mm;
  width:189mm; height:264mm;   /* 3×63mm, 3×88mm */
  display:grid;
  grid-template-columns:repeat(3, 63mm);
  grid-template-rows:repeat(3, 88mm);
}
.grid::before{                 /* one cut guide around the whole 3×3 block */
  content:""; position:absolute; inset:0;
  outline:.35pt dashed #bbb; pointer-events:none;
}
```

`.grid::before` is **screen + print** by default — it's a real cut guide
someone follows with scissors, not a debug aid, so don't wrap it in
`@media print{ display:none }` unless the user explicitly wants a guide-free
final print.

Render a page of 9 card-data objects with:

```js
function renderPage(cards, label){
  return `<div class="page">
    <div class="page-label">${label}</div>
    <div class="grid">${cards.map(cardHTML).join('')}</div>
  </div>`;
}
```

## 4. Card anatomy

Every front card is `title-zone` → `icon-zone` → `body-zone`, stacked in a
flex column:

```css
.card{
  width:var(--card-w); height:var(--card-h);
  border:1.3pt solid var(--accent);
  border-radius:4mm;
  padding:4mm 3.6mm 3.8mm;
  display:flex; flex-direction:column; align-items:center;
  text-align:center;
  background:var(--cream);
  overflow:hidden;
  position:relative;
}
.title-zone{ min-height:11mm; display:flex; flex-direction:column; align-items:center; justify-content:flex-start; }
.title{ font-family:'Poppins',sans-serif; font-weight:800; font-size:12.5pt; letter-spacing:.2px; color:var(--accent-dark); text-transform:uppercase; line-height:1.15; }
.subtitle{ font-family:'Poppins',sans-serif; font-weight:600; font-style:italic; font-size:7.6pt; color:var(--accent); margin-top:.6mm; }

.icon-zone{ height:24mm; width:100%; display:flex; align-items:center; justify-content:center; margin:1.5mm 0 1mm; flex-shrink:0; }
.icon-zone img{ max-height:100%; max-width:100%; object-fit:contain; display:block; }

.body-zone{ flex:1; width:100%; display:flex; flex-direction:column; align-items:center; justify-content:center; gap:2mm; }
.p{ font-size:8.3pt; line-height:1.32; color:#2a2a2a; }              /* plain body paragraph */
.q{ font-family:'Nunito Sans',sans-serif; font-weight:800; font-size:8.6pt; line-height:1.28; color:var(--accent-dark); } /* bold question / takeaway */
.note{ font-size:7.6pt; line-height:1.3; color:#555; }               /* small footnote */

.divider{ display:flex; align-items:center; gap:2mm; width:70%; margin:.5mm 0; }
.divider .line{ flex:1; height:1pt; background:var(--accent); opacity:.5; }
.divider .dot{ width:1.6mm; height:1.6mm; border-radius:50%; background:var(--accent); }
```

`object-fit:contain` on the icon means you never have to pre-size icon PNGs
exactly — drop in an image of roughly the right aspect ratio and it centers
itself in the 24mm-tall zone.

## 5. Card types

One render function switches on a `type` field. Add new types the same way
— don't create a new component system, just add a branch:

```js
function cardHTML(c){
  let inner = `<div class="title-zone">
      <div class="title">${c.title}</div>
      ${c.subtitle ? `<div class="subtitle">${c.subtitle}</div>` : ''}
    </div>
    <div class="icon-zone">${iconImg(c.icon)}</div>`;

  if(c.type==='perspective'){        // body + bold question + small note
    inner += `<div class="body-zone">
      <div class="p">${c.body}</div>
      <div class="q">${c.question}</div>
      <div class="note">${c.note}</div>
    </div>`;
  } else if(c.type==='intro'){       // multi-paragraph + bold footer
    inner += `<div class="body-zone">
      ${c.paragraphs.map(p=>`<div class="p">${p}</div>`).join('')}
      <div class="q">${c.footer}</div>
    </div>`;
  } else if(c.type==='principle'){   // one paragraph + divider + bold footer
    inner += `<div class="body-zone">
      <div class="p">${c.body}</div>
      <div class="divider"><span class="line"></span><span class="dot"></span><span class="line"></span></div>
      <div class="q">${c.footer}</div>
    </div>`;
  } else if(c.type==='action'){      // multi-paragraph + bold question
    inner += `<div class="body-zone">
      ${c.paragraphs.map(p=>`<div class="p">${p}</div>`).join('')}
      <div class="q">${c.question}</div>
    </div>`;
  } else if(c.type==='blank'){       // fill-in-the-blank template card
    inner += `<div class="body-zone">
      ${c.fields.map(f=>`<div class="field">
          <span class="flabel">${f.label}</span>
          ${Array(f.lines).fill('<span class="fline"></span>').join('')}
        </div>`).join('')}
    </div>`;
  } else if(c.type==='back'){        // repeating card-back design
    inner += `<div class="divider"><span class="line"></span><span class="dot"></span><span class="line"></span></div>
    <div class="tagline">${c.tagline.join('<br>')}</div>`;
  }
  return `<div class="card ${c.type==='blank'?'blank':''} ${c.type==='back'?'back-card':''}">${inner}</div>`;
}

function iconImg(name){ return `<img src="icons/${name}" alt="">`; }
```

Card data is a plain array of objects — no schema validation, just be
consistent within a type:

```js
const page1 = [
  { type:'perspective', title:'Vision', icon:'vision.png',
    body:'Explore situations where visual information is difficult or impossible to perceive.',
    question:'What does the design require people to see?',
    note:'Think about colour, contrast, size, layout and visual cues.' },
  // … 8 more, 9 cards fills one page
];
```

`blank` cards (fill-in-your-own) are a nice thing to include near the end of
a deck — they cost nothing and make the deck extensible by its users.

## 6. Colour system

Three CSS variables carry the whole palette: `--accent` (borders, titles,
dividers), `--accent-dark` (darker text, questions), `--accent-fill` (a
muted mid-tone used when generating/recolouring icon art). Pick a palette by
setting three hex values — nothing else in the CSS references colour
directly except a few greys (`#2a2a2a` body text, `#555` notes, `#999`
guides).

**To recolour an existing deck's icon photos/renders** (not just the CSS),
hue-rotate the PNGs in bulk rather than regenerating them — it's fast and
keeps every icon's shading/texture intact:

```python
from PIL import Image
import numpy as np

DELTA_DEG = 140  # degrees to rotate hue by
DELTA_255 = DELTA_DEG / 360.0 * 255.0

for path in icon_paths:
    im = Image.open(path).convert('RGBA')
    arr = np.array(im)
    rgb, alpha = arr[:,:,:3], arr[:,:,3]
    hsv = np.array(Image.fromarray(rgb, 'RGB').convert('HSV')).astype(np.int16)
    hsv[:,:,0] = (hsv[:,:,0] + DELTA_255).astype(np.int16) % 256
    new_rgb = np.array(Image.fromarray(hsv.astype(np.uint8), 'HSV').convert('RGB'))
    Image.fromarray(np.dstack([new_rgb, alpha]), 'RGBA').save(path)
```

Compute the same rotation on your three hex CSS variables (via
`colorsys.rgb_to_hsv` / `hsv_to_rgb`) so the CSS chrome and the icon art
shift together and stay in family. Always back up originals first — a
session earlier in this project got this exactly right by copying
`icons/*.png` to a scratch dir before mutating them in place.

## 7. Sourcing and cropping icons

Icons should be **tightly cropped around their own content** — no title
text, no body text, no card border — so `object-fit:contain` can center them
cleanly regardless of each card's exact aspect ratio.

If you're extracting icons from a flattened reference image (a mockup, a
screenshot of "here's roughly what I want") rather than clean individual
assets, don't eyeball fixed crop boxes per card — icon size/position varies
card to card and a fixed box will slice through some of them. Instead:

1. Detect the card grid in the source image by scanning for the border/ink
   colour (`diff = |pixel - background| > threshold`, then look for long
   contiguous runs to find each card's left/right/top/bottom edges).
2. Within each card, scan row-by-row ink density (`mask.sum(axis=1)`) from
   just below the title downward. A **true content gap** reads as several
   consecutive rows at or near zero; distinguish it from anti-aliasing noise
   with a minimum run length (start at ~5 rows, only go lower if a real gap
   turns out to be that short) and a per-row pixel-count floor (not just
   "any nonzero pixel" — stray JPEG/paper-texture noise triggers false
   positives at `>0`).
3. Take the **first honest gap** as end-of-title, and the **next gap** after
   that as end-of-icon. Multi-line titles need a taller skip before the
   first gap; check each card, don't assume one offset fits all.
4. Autocrop to the tight bounding box of non-background pixels inside that
   band, add a few px of padding, save.

This is inherently a bit fiddly — budget for **one visual QA pass**: render
a contact sheet (all cropped icons pasted into one grid image) and actually
look at it before wiring the crops into card data. A crop that clips the
edge of a token/glyph is the single most common mistake here, and it is
easy to miss at a glance — check every one at full size at least once.

## 8. The card back

Two supported patterns:

**A. Plain repeating back** (this project's default) — a small icon,
title, divider, tagline, no border (`border-color:transparent`) so a few mm
of duplex misregistration during printing never shows as a mismatched
outline:

```css
.back-card{ border-color:transparent; }
.back-card .icon-zone{ height:30mm; }
.back-card .tagline{ color:var(--accent-dark); font-size:8.6pt; font-weight:700; line-height:1.6; margin-top:1mm; }
```

**B. Full-bleed photographic/textured back** — when the back design is a
busy image that should fill the card completely, give it real print bleed
so duplex misregistration never shows white paper at the trimmed edge:

```css
.bleed-card{ width:var(--card-w); height:var(--card-h); position:relative; }
.bleed-card img{
  position:absolute; left:-5mm; top:-5mm; width:73mm; height:98mm; /* card + 5mm each side */
  object-fit:cover; display:block;
}
```

Since the 9 cards in the grid sit edge-to-edge with zero gap, this
deliberately **overflows into the neighbouring cells** — harmless because
every card on a repeating back page is identical, so the overlap is
invisible; it's material that gets trimmed away regardless.

If your only source art is exactly card-sized with no extra bleed material
around the edges, don't sample outward into a blank inter-card gap in the
*source* image (you'll bake a visible pale ring into the "bleed"). Instead
generate the bleed by **edge-extending** the tight crop:

```python
import numpy as np
arr = np.array(tight_crop)                     # exact card-size crop, no gap included
pad_x, pad_y = 24, 24                           # ≈5mm at this crop's px/mm scale
bled = np.pad(arr, ((pad_y,pad_y),(pad_x,pad_x),(0,0)), mode='edge')
```

`mode='edge'` repeats the outermost row/column of real pixels outward,
which reads as a natural continuation of the texture instead of a hard
edge or a smear of background colour.

## 9. Page order for duplex printing (front/back must alternate)

Automatic double-sided ("duplex") printing — whether the user ticks
"Two-sided" in the print dialog or duplex-prints a PDF you handed them —
pairs **consecutive pages** as the two sides of one physical sheet: document
pages 1 & 2 print on sheet 1 (page 1 on side A, page 2 on side B), pages 3 &
4 on sheet 2, and so on. The document's page order *is* the sheet-pairing —
there is no separate "this page is a back" flag the printer reads.

That means the `document.getElementById('app').innerHTML = ...` assembly
must **alternate** front, back, front, back, ... — one full pair per sheet —
never all fronts followed by all backs, even when every card shares one
identical repeating back design (§8A). If you render the fronts as a block
and the (single, reused) back as a block after it, two front pages get
duplexed onto the same sheet back-to-back, and the trailing back page ends
up paired with whatever follows it (nothing, or an unrelated page like the
box template) — the deck comes out of the printer with fronts on both sides
of some sheets and stray backs elsewhere, and it isn't obvious from looking
at any single page in a browser or PDF viewer, only from an actual
double-sided print.

```js
// WRONG — breaks automatic duplex printing: fronts, then backs, grouped
document.getElementById('app').innerHTML =
  renderPage(front1, 'Front 1') +
  renderPage(front2, 'Front 2') +
  renderPage(backPage, 'Back');          // one back page, tacked on the end

// RIGHT — interleaved: front, back, front, back — one pair per sheet
document.getElementById('app').innerHTML =
  renderPage(front1, 'Front 1') +
  renderPage(backPage, 'Back (sheet 1)') +
  renderPage(front2, 'Front 2') +
  renderPage(backPage, 'Back (sheet 2)');
```

Even with one repeating back design, call its render function again for
each sheet rather than rendering it once — the fix is to repeat the call,
not to add a flag.

**If each card's back is unique** (not a repeating design), the back page's
own cell order also has to line up with that sheet's front page after the
physical flip, which depends on which edge the printer's duplex mode flips
on (long-edge flip mirrors left-right; short-edge flip mirrors top-to-bottom
— confirm which the user's setup uses, long-edge is the common default) —
mirror the back page's cell order accordingly so cell N's back lands behind
front cell N. For the common **repeating identical back** this never
matters, since every cell is the same; skip it.

**Standalone pages that never pair with a card page** (the fold-and-glue box
template, §10) go after all the interleaved front/back pairs, so they start
a fresh sheet. That's fine printing single-sided with a blank reverse — but
only works cleanly if it lands on an odd page number in the final document;
if the front/back pairs before it are already even in count (they will be,
since they're pairs), it will.

## 10. The fold-and-glue box template

A tuck box sized to the deck's card footprint, printed as its own A4 page
(a dieline: solid lines = cut, dashed = fold, one shaded panel = glue tab).
The technique to reuse:

- Compute the net as **one outer cut path** (a single closed SVG `<path>`
  tracing the whole silhouette, including flap notches and the tuck-flap
  thumb curve) plus a **separate list of internal fold lines** (each drawn
  as its own dashed `<line>`). Don't try to render the fold lines as part of
  the same path as the cut lines — mixing solid/dashed within one stroked
  path is awkward; two passes over the same geometry is simpler and matches
  how real dielines are authored.
- Box footprint: width/depth from the card's width + a couple mm clearance;
  depth from the expected stack thickness (~0.3–0.35mm per card at typical
  card stock) plus slack.
- One glue tab is fine and far more reliable than a no-glue interlocking
  base for a hand-cut box — don't over-engineer a friction-fit bottom.
- Legend (small inline SVG swatches + labels for cut/fold/glue) and a short
  numbered assembly instructions block are worth the space — this is a
  physical craft object, not just a picture.

## 11. Non-standard sizes on the same A4 sheet (A6/A7 inserts, coins, etc.)

Sometimes a deck needs an oddball insert — an A6 or A7 sized explainer card,
a card meant to have something physical glued to it — printed on the same
A4 workflow. Two things matter here:

**Keep the page size at A4.** Don't try to mix page orientations/sizes in
one document (see the print-to-pdf caveat in §12) — instead centre or tile
the smaller artwork on a normal 210×297mm `.page`, same as every other page.

**Compute exact paper-fraction tiles, don't guess.** A4 = 2×A5 = 4×A6 =
8×A7, and each halving alternates which side gets cut:

| Size | mm | fits on one A4 as |
|---|---|---|
| A5 | 148×210 | 2 (1×2 or 2×1) |
| A6 | 105×148 | 4 (2×2) |
| A7 | 74×105 | 8 (2×4, cells **105×74.25mm** landscape) |

If your card's natural content layout doesn't match the orientation the
tiling wants (e.g. a wide landscape card needs to go into a tall narrow
cell), **rotate the rendered card 90°** rather than redesigning the layout:

```css
.cell{ position:relative; width:105mm; height:148mm; }  /* the tile the paper math wants */
.cell .card{                                             /* your card, unchanged, e.g. 148×105 */
  position:absolute; left:50%; top:50%;
  transform:translate(-50%,-50%) rotate(90deg);
}
```
`translate(-50%,-50%)` centers the card on the cell first; `rotate(90deg)`
then spins it around its own centre, so its 148×105 footprint becomes a
105×148 visual footprint that exactly fits the cell. After cutting, rotate
the physical piece back — this is a standard imposition trick, not a hack.

Check the arithmetic before you build: multiply cells × cell-size and
confirm it equals 210×297 (or A4 minus your intended margin) **before**
writing the CSS, the same way you'd check a cut list before touching wood.

**Cut-guide vs. content edge.** If cards should bleed to the true sheet
edge (no blank margin) but the guide *lines* between them shouldn't run
into the last few mm of paper a printer can't reliably mark, size the two
independently: full-bleed card/cell geometry, but the dashed divider lines
inset ~5mm from the outer sheet edge on both ends:

```css
.grid{ position:absolute; top:0; left:0; width:210mm; height:297mm; } /* cards: edge to edge */
.divider-v{ position:absolute; left:105mm; top:5mm; width:0; height:287mm; border-left:.35pt dashed #bbb; }
.divider-h{ position:absolute; top:74.25mm; left:5mm; height:0; width:200mm; border-top:.35pt dashed #bbb; }
```
"5mm margin around the page" and "cards inset 5mm from the edge" are **not
the same thing** — confirm which one is meant before implementing either.
When only one side of a front/back pair needs the cut guide at all (you cut
once, through both layers, using whichever side you're looking at), draw it
on one side's page only.

**Bordered cards (not the bleed-art case above) need a position-aware
border, not a separate divider.** The dashed-divider technique just above
is for card art with no border of its own. If the card design *is*
bordered — the default `.card{ border:1.3pt solid var(--accent); }` from
§4 — and the grid tiles full-bleed to the true sheet edge (no page margin,
as this section's math wants), don't also draw a divider: the card's own
border already runs along every shared edge between cards, doubled up with
its neighbour's border, which is the cut guide. The problem is the *outer*
perimeter of that border — the sides facing the physical sheet edge rather
than another card — which sits inside the ~5mm most printers can't mark, so
it prints clipped or missing while the inner lines print fine. Since the
sheet edge doesn't need a cut line anyway (the paper edge already separates
it), the fix is to omit border on exactly those outward-facing sides, per
cell, based on its row/column position — not to shrink the grid to make
room for a margin. For an R-row × C-col grid in row-major DOM order:

```css
.grid > .card:nth-child(-n+C)  { border-top:none; }      /* top row */
.grid > .card:nth-child(n+K)   { border-bottom:none; }   /* bottom row, K = (R-1)*C + 1 */
.grid > .card:nth-child(Cn+1)  { border-left:none; }     /* left column */
.grid > .card:nth-child(Cn)    { border-right:none; }    /* right column */
```
Substitute the actual numbers for `C`, `R`, `K` (e.g. a 2×2 grid is
`nth-child(-n+2)`, `nth-child(n+3)`, `nth-child(2n+1)`, `nth-child(2n)`).
Corner cells match two of these rules at once and correctly end up with
only their two inward-facing sides bordered. Border-radius on a card like
this reads oddly once two adjacent sides have no border to round into —
drop `border-radius` (square corners) on decks that use this technique.

**Temporary QA outline.** While iterating on any of this, a plain
`outline:.3mm solid red;` on the card element is the fastest way to see
exactly where each card's true edge falls relative to bleed art, dashed
guides, and the sheet edge. `outline` (not `border`) so it doesn't perturb
layout, and note that it can get visually covered by a *later* sibling's
overlapping bleed content if there is any — put it on a piece that doesn't
overlap, or accept it only shows on the near edges. Remove it (or the whole
temporary rule) once the geometry is confirmed; say so explicitly when you
add it and again when you remove it.

## 12. Exporting to PDF

```bash
google-chrome --headless --disable-gpu --no-sandbox \
  --print-to-pdf=out.pdf --print-to-pdf-no-header --no-pdf-header-footer \
  --run-all-compositor-stages-before-draw \
  "file:///absolute/path/to/index.html"
```

Verify page count and paper size before shipping:

```bash
pdfinfo out.pdf | grep -E "Pages|Page size"
```

**Known bug — verify every time, not just once:** this specific
combination of a heavy multi-page document (custom fonts, many images,
absolutely-positioned SVG content) and Chrome's headless
`--print-to-pdf` can silently start scaling the *entire* PDF down and
padding it with blank space (bottom-right) once the document crosses a
page-count threshold (observed at 9+ pages in this project; the exact
trigger is document-weight-dependent, not a fixed page count in general).
Every individual `.page` looked correct on screen and the PDF's own
`/MediaBox` still reported A4 — the bug only shows up when you actually
open/rasterize the PDF. Detect it cheaply:

```bash
pdftoppm -f 1 -l 1 -png -r 80 out.pdf check
python3 -c "
from PIL import Image
import numpy as np
im = Image.open('check-1.png')            # or check-01.png if 10+ pages
arr = np.array(im.convert('RGB'))
white = arr.min(axis=2)
print('right edge used:', np.where((white>250).sum(axis=0)>0)[0].max(), '/', im.width)
print('bottom edge used:', np.where((white>250).sum(axis=1)>0)[0].max(), '/', im.height)
"
```
Both numbers should be within a pixel or two of the image's full
width/height. If they're both around ~83% of it, the bug has hit.

**Fix:** never let a single `--print-to-pdf` call cover more pages than the
threshold that broke for this document. Split the render into two (or more)
HTML variants — same file, just changing which `renderPage(...)` calls feed
`document.getElementById('app').innerHTML` — print each separately, then
concatenate:

```bash
pdfunite part1.pdf part2.pdf final.pdf
```
Re-run the pixel check above on the merged PDF's first *and* last page
before calling it done — a bad merge order or a leftover truncated part is
just as easy to ship unnoticed as the original bug.

## 13. Workflow checklist for a new or edited deck

1. Write/edit card data (§5) and any new CSS the card type needs (§4).
2. Screenshot-check in a real browser before touching the PDF:
   `google-chrome --headless --disable-gpu --no-sandbox --window-size=1050,<tall> --screenshot=out.png "file://…"`,
   then crop/view the region you changed. Cheaper and faster to iterate on
   than round-tripping through PDF export every time.
3. Check the final `innerHTML` assembly interleaves front/back pages (§9) —
   this is easy to get right while a deck has one front+one back page and
   easy to silently break the moment it grows past that, so re-check it
   whenever the page count changes, not just on the first build.
4. Only once the screenshot looks right, export to PDF (§12) and verify
   page count + the full-bleed pixel check.
5. If icons came from a reference image rather than clean assets, do the
   contact-sheet QA pass (§7) before wiring them in.
6. Never leave a temporary QA aid (red outline, debug grid lines) in a PDF
   you hand to the user without saying so — and follow up to remove it.

## If a request needs something the guide doesn't cover

Extend the guide's own patterns (add a new `type` branch in §5, add a new
CSS variable in §6) rather than restructuring what's there — one render
function that switches on a `type` field, plain data objects, three CSS
variables for the whole palette. If it's a genuinely new reusable pattern
(a new back style, a new insert size), add it back into this file as a new
subsection so the next deck benefits too.
