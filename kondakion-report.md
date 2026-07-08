# Session report: `kondakion.tex` pitch-mark notation

This report documents the development of [`kondakion.tex`](kondakion.tex),
a plain-TeX file that typesets a hymn text with hand-drawn pitch marks (slanted
strokes above syllables) and underlines (for held notes). It covers all work up
to and including the `\finalskip` spacing before the hymn's final line.

## Goal

Typeset a hymn/canticle text with marks above individual syllables indicating
rising or falling pitch, plus a way to mark syllables that are sung on a longer
note. Each pitch mark is one or more **45-degree slanted strokes**: upward for
rising pitch, downward for falling, with one/two/three strokes for increasing
emphasis. The text started as a single line and grew into a multi-paragraph
piece with hanging-indented continuation lines.

## Final state

- **Format:** plain TeX, compiled with `pdftex`, matching the repo's
  `kummer.tex`/`cv.tex` style (no `\documentclass`, no packages, ends with
  `\bye`).
- **Font:** Helvetica 18 pt (`\font\rm=phvr8t scaled 1800`).
- **Paper:** A4 (`\pdfpagewidth=210mm \pdfpageheight=297mm`).
- **Layout:** uniform leading (every line the same height), and a hanging indent
  (first line of each paragraph flush left, continuation lines indented).
- **Tracked PDF:** unlike the other documents, `kondakion.pdf` is committed
  to the repo (it was added intentionally, overriding the `*.pdf` gitignore
  rule), and is rebuilt and committed alongside each source change.

### Macros

| Macro | Meaning |
|-------|---------|
| `\rise{syll}` | one upward stroke (rising pitch) |
| `\risetwo{syll}` | two upward strokes |
| `\risethree{syll}` | three upward strokes |
| `\fall{syll}` | one downward stroke (falling pitch) |
| `\falltwo{syll}` | two downward strokes |
| `\fallthree{syll}` | three downward strokes |
| `\hold{syll}` | underline the syllable (sung on a longer note) |
| `\finalskip` | vertical space that sets off the hymn's final line |

The six stroke macros share one core, `\risemark{strokes}{syllable}`, which
measures the syllable's width, overlays the strokes with an `\rlap` +
`\pdfliteral`, and prints the syllable. `\hold` is independent but composes with
the marks — e.g. `\rise{\hold{heer}}` puts a stroke above and an underline below
the same syllable.

### Tunable parameters

| Parameter | Value | Controls |
|-----------|-------|----------|
| `\risescale` | `0.9` | stroke **size** (length); `1.8` is proportional to 18 pt, `0.9` is half |
| `\risethick` | `1.44` | stroke **line width** (bp), independent of size |
| `\riseheight` | measured | fixed height of the marks above the baseline (cap/ascender height of the font) |
| `\holddepth` | measured | how far below the baseline the underline sits (descender depth + 1.5 pt) |
| `\holdthick` | `1.44pt` | underline thickness, independent of size |
| `\holdgap` | `4pt` | how much narrower than its syllable each underline is drawn (centered), so adjacent underlines don't join |

## Design and implementation

- **Why a pdf literal.** A diagonal line cannot be made from `\hrule`/`\vrule`
  (those are axis-aligned only), so the strokes are drawn with a `\pdfliteral`.
  This is the one engine dependency (pdfTeX); everything else is portable plain
  TeX.
- **Exact 45 degrees.** Each stroke is a `(+5,+5)` (rising) or `(+5,-5)`
  (falling) big-point vector. Strokes are defined at the 10 pt design size and
  scaled by `\risescale` via a `cm` matrix. Falling strokes are the mirror of
  rising ones — obtained by swapping the two `y`-values of each stroke.
- **Centering.** A group of one/two/three strokes is laid out symmetrically
  about `x = 0` and placed with `\kern0.5\dimen0` (half the syllable width), so
  it stays horizontally centered over the syllable regardless of stroke count.
- **Fixed mark height.** Marks are raised by a fixed `\riseheight` (measured
  once from the font, `\hbox{Ahg}`), not by each syllable's own height, so every
  mark sits the same distance above the baseline and they line up horizontally.
- **Size/thickness decoupling.** The `cm` matrix scales the stroke geometry, but
  the stroke operator `S` runs *after* `Q` restores the CTM, so the pen width
  `\risethick` is applied unscaled. The PDF current path survives `Q` (only the
  graphics state is popped), so the subpaths built under the scale are still
  stroked. This lets size and thickness be changed independently. (A single `S`
  after `Q` strokes all subpaths of a multi-stroke mark.)
- **Underlines.** `\hold` draws a rule a fixed `\holddepth` below the baseline
  (just under the descenders, so underlines line up across syllables), of
  thickness `\holdthick`. Being below the baseline, it never fights the marks
  above. The rule is drawn `\holdgap` narrower than the syllable and centered,
  so two adjacent held syllables leave a small gap between their underlines
  instead of joining.
- **Uniform leading.** The line height is set once, from the mark geometry:
  `\baselineskip` = mark reach (`\riseheight + 8bp*\risescale`) + underline reach
  (`\holddepth + \holdthick`) + a small gap. `\lineskiplimit=-\maxdimen` forces
  TeX to always use `\baselineskip`, so every pair of baselines is exactly the
  same distance apart whether or not a line carries marks. `\topskip` reserves
  room for the first line's marks under the top margin.
- **Hanging indent.** `\everypar={\hangindent=1.5em \hangafter=1 }` keeps the
  first line of each paragraph flush left (with `\parindent=0pt`) and indents
  every following wrapped line. It is set in `\everypar` because `\hangindent`
  and `\hangafter` reset after each paragraph; this also covers paragraphs that
  start with a mark, since those enter horizontal mode via `\leavevmode`, which
  still fires `\everypar`.
- **Final line.** Rather than a special symbol, the hymn's last line is set off
  with `\finalskip` — a small `\vskip` (4 pt) placed, in vertical mode, between
  the preceding `\par` and the final line. It adds to the fixed `\baselineskip`,
  so the last line gets a little extra air without disturbing the uniform
  leading elsewhere.

## How it developed (chronological)

1. **First version (LaTeX + TikZ).** A `\rise` macro placed a single 45-degree
   line above the `heer` syllable of "kondakion".
2. **Ported to plain TeX.** Reimplemented `\rise` with `\pdfliteral` (since
   `\hrule`/`\vrule` can't draw diagonals), keeping the file pure plain TeX.
3. **Centering fix.** The stroke initially sat left of center; centered it over
   the syllable with a symmetric path about `x = 0`.
4. **Two/three-stroke variants + shared core.** Refactored the box logic into
   `\risemark`; added `\risetwo`/`\risethree`.
5. **Falling variants.** Added `\fall`/`\falltwo`/`\fallthree` as mirror images.
6. **Helvetica 18 pt** and a `\risescale` factor (`cm` matrix) so strokes and
   line width scale with the font.
7. **Leading first attempt.** Added an invisible per-mark strut so lines with
   marks reserved vertical room.
8. **A4 paper size.**
9. **Fixed mark height.** Introduced `\riseheight` so all marks align at the
   same distance above the baseline.
10. **Uniform leading.** Replaced the per-mark strut with a single fixed
    `\baselineskip` (+ `\lineskiplimit`/`\topskip`) so every line is the same
    height.
11. **`\hold` underlines** for held syllables, with the leading extended to
    reserve room for them.
12. **Smaller marks + size/thickness decoupling.** Halved the mark size
    (`\risescale` 1.8 -> 0.9) and separated line width (`\risethick`) from size
    via the `cm`/`Q`/`w`/`S` reordering; the underline thickness `\holdthick`
    was likewise decoupled and thickened.
13. **Doubled stroke thickness** (`\risethick` 0.72 -> 1.44) and added the
    **hanging indent** for continuation lines.
14. **Underline gap.** Made each `\hold` rule `\holdgap` narrower than its
    syllable and centered (`\holdgap` = 4 pt), so the underlines of adjacent
    held syllables stay separate.
15. **Final-line marking.** Explored an enlarged asterisk (bold, then regular)
    for the hymn's last line, then settled instead on `\finalskip` — a little
    vertical space before the final line, with no special symbol.

## Bugs found and fixed

1. **Stroke off-center.** The first stroke sat left of the syllable; fixed by
   drawing it symmetrically about the center and kerning to the midpoint.
2. **Marks landing on a separate line above the text.** A mark at the *start* of
   a line ran in vertical mode, so its boxes were added to the vertical list —
   the em-dash/stroke ended up on their own line above the text. **Fix:**
   `\leavevmode` in `\risemark` (and `\hold`) forces horizontal mode first.
3. **`\lower` in vertical mode** (during an abandoned `\risedash` helper): raised
   `! You can't use \lower in vertical mode` — this is what exposed bug #2; the
   `\leavevmode` fix superseded it.
4. **Marks not reserving vertical space.** The `\pdfliteral` strokes have no
   height to TeX, so the next line up could collide with them. First solved with
   a per-mark invisible strut (`4210d09`), later superseded by the global
   uniform-leading approach (`8644033`).
5. **Uneven leading.** The per-mark strut only lifted *marked* lines. **Fix:**
   set the line height once, globally, so all lines match.

## Verification

Every change was checked by compiling with `pdftex` and rendering the PDF to an
image to confirm the marks were correctly shaped, sized, centered, and spaced,
that marks and underlines composed, that the leading stayed uniform, and that
the hanging indent applied only to continuation lines. `pdfinfo` confirmed A4.

## Commit history

Source (`kondakion.tex`), newest last:

- `2b7851e` add kondakion hymn line with rising-pitch mark
- `7169e1d` add pitch-stroke variants, falling marks, Helvetica 18pt
- `e065118` fix pitch marks at start of line, use several in one line
  (`\leavevmode`)
- `4210d09` reserve line height for the pitch strokes (per-mark strut)
- `ab6275a` align pitch marks at a fixed height above the baseline (also adds A4)
- `8644033` give every line the same height for even leading (global leading)
- `5be0675` add `\hold` to underline held syllables
- `4c2994e` halve mark size and decouple stroke/underline thickness
- `4faa43f` indent continuation lines; thicken marks
- `86dc4d3` space adjacent held-syllable underlines (`\holdgap`); update report
- `c7b8243` bolder final-line marker (`\endmark`); add two rise marks
- _(committed together with this report revision)_ replace `\endmark` with
  `\finalskip` (vertical space before the final line)

This report: `d8d4755` added the original version; later revisions brought it up
to date through `c7b8243`, and the present revision adds the `\finalskip` change
listed above.

## Notes / possible next steps

- `kondakion.pdf` **is** tracked (added in an earlier commit, overriding the
  `*.pdf` gitignore), so it is rebuilt and committed alongside each source
  change.
- The two mark dimensions are independent knobs: `\risescale` (size/length) and
  `\risethick` (line width); the underline has `\holddepth` (position),
  `\holdthick` (thickness), and `\holdgap` (inset). Changing size no longer
  disturbs thickness.
- The pitch-stroke **size** (`\risescale`) is not yet settled and is expected to
  be revisited; thickness is independent, so tuning size won't disturb it.
- The hanging-indent amount is `1.5em` (font-relative); adjust as desired.
- The final line is set off by `\finalskip` (a `\vskip`); adjust the amount
  (currently 4 pt) to taste.
- `\riseheight`/`\holdthick`/leading are all derived from the font at setup, so
  changing the font size mostly self-adjusts; `\risescale` and `\risethick` are
  the hand-set values to revisit for a new size.
