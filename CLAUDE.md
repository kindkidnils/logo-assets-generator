# Logo Assets Generator

Single-page browser tool om een compleet, geordend logo-asset-pack te genereren per klant/project: digitale formaten (SVG, PNG, JPG en WebP per kleurversie) én print-klare vectorbestanden (EPS en PDF met exacte CMYK), verpakt in één ZIP. Elk bestand is bijgesneden tot de rand van het logo.

Eerder bekend als "LogoKit" (herbouwd/herbenoemd tijdens de skill-ontwikkeling — zie git-historiek als die bewaard is, anders begin bij deze versie).

## Doel

Vervangt het handmatig samenstellen van een logo-pakket voor een klant of intern project: transparante PNG's, JPG's op een achtergrond, en print-EPS/PDF met correcte CMYK — allemaal in één keer, consistent benoemd, in een voorspelbare mapstructuur.

## Stack

Vanilla HTML/CSS/JS + Canvas API, gebouwd op `assets/template.html` uit de `tkk-tool-template`-skill (spoor A). Eén bestand, geen build step. Externe dependency: JSZip (CDN) voor het samenstellen van de ZIP, pdf.js (CDN, dynamic import) enkel wanneer een PDF geüpload wordt.

## Kernfunctionaliteit

1. **Klantnaam** (stap 01) — vormt de basis van elke bestandsnaam in de export, nooit de naam van het geüploade bronbestand.
2. **Logo's uploaden** (enkel SVG of PDF — vectorbronnen, zie "Bewust niet") — elk logo krijgt een vrij intypbare rol (met suggesties via `<datalist>`, geen vaste dropdown) die de mapnaam bepaalt. Een "+"-tegel naast bestaande logo's maakt duidelijk dat je er meer kan toevoegen.
3. **Merkkleuren** (naam + hex + exacte CMYK + optioneel Pantone) — de CMYK-waarden worden letterlijk overgenomen, nooit automatisch geschat vanuit hex (browsers kunnen dat niet betrouwbaar zonder ICC-profiel).
4. **Kleurversies per asset** — dynamisch: elke toegevoegde merkkleur plus zwart/wit worden automatisch aangeboden als aan te vinken variant. Zodra er een meerkleurig logo is, komt daar de variant **Origineel** bij (zie "Meerkleurige logo's").
5. **Pakket-opties** — per kleurversie één SVG, één PNG (3000 px), JPG's per achtergrond en één WebP voor digitaal, plus een print-map met een EPS en een PDF (vector, exacte CMYK).

## Vector-EPS en -PDF: hoe de CMYK-precisie werkt

Browsers kunnen RGB niet betrouwbaar naar CMYK omrekenen (geen ICC-profiel) — dus in plaats van te proberen te converteren, wordt de vorm van het logo (silhouet) apart vastgelegd en met PostScript's `setcmykcolor` ingekleurd met de CMYK-waarde die de gebruiker letterlijk heeft ingevuld. Twee routes:

- **Normaal geval**: de vectorpaden van de geüploade SVG/PDF worden zelf omgezet naar PostScript-paden (`moveto`/`lineto`/`curveto`) — een echt, oneindig schaalbaar vectorbestand. Beide brontypes lopen via `vectorDrawing()`, dat de paden één keer opbouwt; `buildVectorEPS()` en `buildVectorPDF()` verpakken die als EPS (PostScript-operatoren) of PDF (dezelfde operatoren onder hun PDF-naam: `m`/`l`/`c`/`k` …). Een EPS en PDF van dezelfde versie renderen pixel-identiek: een PDF wordt bij upload al omgezet naar een SVG (`logo.svgFromPdf`, zie hieronder), via pdf.js' operator-list (de al-uitgepakte tekencommando's), niet via een eigen PDF-parser.
- **Fallback bij parse-mislukking**: als de vectorpaden van een specifiek logo niet uitgelezen kunnen worden (bv. tekst die niet naar paden is omgezet, een verloop, of een clip-path — zie de beperkingen bij `collectPdfPaintOps`/`collectPaintOps`), krijgt dat logo geen PDF en valt de EPS terug op een hogeresolutie rastermasker (1500px) ingekleurd via `imagemask` — nog steeds de exacte CMYK-waarde, maar geen oneindig schaalbaar vectorpad, en (als enige bestand) niet bijgesneden tot het logo. Dit is een foutopvang, geen normaal exportpad — sinds de upload-stap enkel nog SVG/PDF aanvaardt (zie "Bewust niet") is er geen raster-bronbestand meer dat hier standaard doorheen zou moeten.

Getest met Ghostscript: de gerenderde CMYK-waarden kwamen exact overeen met de ingevoerde waarden (tot op 8-bit-afrondingsniveau).

Diezelfde PDF-vectorpaden (via `collectPdfPaintOps()`) voeden ook de **digitale SVG-export**: bij upload schrijft `pdfOpsToSvg()` ze één keer als SVG `<path d="…">` met per vorm de bronkleur uit de PDF (`logo.svgFromPdf`). Vanaf dan wordt een PDF-bron behandeld als een geüploade SVG — `recolorSvgText()` voor de monochrome versies, de kleurkoppeling voor "Origineel".

## Meerkleurige logo's

Elke vorm krijgt bij upload zijn effectieve vulkleur als `data-tkk-fill`-attribuut (`annotateSvgColors()` → `logo.colorSvg`). Voor een SVG berekent de browser die zelf via `getComputedStyle` (het logo hangt daarvoor heel even onzichtbaar in de pagina), zodat ook kleuren via `<style>`-klassen en overerving van een `<g>` kloppen. Voor een PDF komt de kleur uit pdf.js' `setFillRGBColor`-operatoren. Twee of meer verschillende kleuren = meerkleurig logo.

- **Koppeling**: elke gevonden kleur wordt gekoppeld aan de dichtstbijzijnde merkkleur uit stap 03 (RGB-afstand ≤ `COLOR_MATCH_TOLERANCE` = 40; ruim, omdat pdf.js CMYK naar RGB omzet en die hex dus zelden exact gelijk is aan de officiële). Per kleur handmatig te overschrijven in stap 02, ook bewust op "Niet gekoppeld" — een handmatige keuze wordt nooit door de automatische koppeling overschreven, tenzij die merkkleur verwijderd wordt. Een ontbrekende kleur kan je met één klik naar het formulier van stap 03 sturen (hex al ingevuld).
- **Variant "Origineel"** (enkel voor meerkleurige logo's): digitaal krijgt elke vorm de **hex** van zijn merkkleur (`originalSvgFor()`, en daaruit ook PNG/JPG/WebP — zonder tint), print krijgt een EPS met per vorm de **exacte CMYK** in tekenvolgorde, zodat overlappingen kloppen (`buildVectorEPS()` met een kleurfunctie i.p.v. één CMYK).
- **Geen EPS zonder volledige koppeling**: zolang één kleur niet gekoppeld is, komt er geen `origineel-print.eps` (ook niet in de boomweergave). Het rastermasker-vangnet kent maar één kleur, dus voor "Origineel" is er geen fallback.

## Passend formaat (geen vierkant, geen witruimte)

Elk bestand wordt bijgesneden tot de rand van de getekende vormen, zodat een logo in een lay-out exact op zijn rand uitgelijnd kan worden. Vroeger kwam alles in een vierkant met marge, wat bij een breed logo (bv. 2000×2000 voor een woordmerk) vooral lege ruimte opleverde.

- **Begrenzing** (`logoBBox()` → `logo.bbox`, één keer bij upload): de exacte omtrek van alle vulvormen, met bij curves de werkelijke uitersten van de bézier in plaats van de controlepunten (`opsBBox()`). Lijnen (strokes) tellen mee met de halve lijndikte. De begrenzing blijft binnen de originele viewBox, zodat vormen buiten het artboard (die in het bronbestand onzichtbaar waren) niet opduiken.
- **SVG**: dezelfde inhoud, met `viewBox`/`width`/`height` op de begrenzing (`cropSvg()`). Uitzondering: `print-vector-bron.svg` en `print-bron.pdf` zijn ongewijzigde kopieën van het bronbestand.
- **PNG/JPG/WebP**: gerenderd vanuit de bijgesneden vector-SVG, rechtstreeks op de doelgrootte (`renderRaster()`), dus ook voor PDF-bronnen scherp — vroeger werd daar een tussenbeeld van 2000px opgeschaald. De maten (`PNG_SIZE` 3000, `JPG_SIZE` 2000, `WEBP_SIZE` 2048) gelden voor de **langste zijde**; de andere zijde volgt uit de verhouding, afgerond op hele pixels. `preserveAspectRatio="none"` vermijdt dat die afronding een halve pixel witruimte aan de randen laat.
- **EPS en PDF**: BoundingBox (EPS) of MediaBox/TrimBox (PDF) = de begrenzing, geschaald naar 720 pt op de langste zijde. De EPS-BoundingBox is naar boven afgerond op hele punten (de HiResBoundingBox en de PDF-boxen zijn exact).

Getest met een volledige export (Comova-SVG + Tigron-PDF, 54 bestanden): elk raster-, EPS- en PDF-bestand raakt het logo aan alle vier de randen, EPS en PDF gerenderd met Ghostscript (exacte CMYK, EPS en PDF pixel-identiek); de PDF opent ook correct in de PDF-engine van macOS.

## JPG-achtergronden

Een JPG heeft geen transparantie: een logo met te weinig contrast tegenover de achtergrond gaf een effen of onbruikbaar vlak (wit op wit, zwart op zwart, een donker logo op zwart). Zulke combinaties worden overgeslagen (`jpgBgsFor()`), ook in de boomweergave en de schatting. Criterium: WCAG-contrastverhouding < 2.0 tussen logokleur en achtergrond (`MIN_JPG_CONTRAST`). Daaronder vallen bv. donkergroen op zwart (≈ 1.5) en geel op wit (≈ 1.35); oranje op wit (≈ 2.8) blijft. Voor "Origineel" moet élke kleur van het logo de drempel halen — anders valt een deel van het logo weg (bv. de donkergroene tekst van Comova op zwart).

## State shape

```js
logo = {
  id, name, role, type: 'svg' | 'pdf',
  img,            // rasterbeeld voor de thumbnail en het rastermasker-vangnet van de EPS
  svgText,        // SVG-bron (enkel type 'svg')
  pdfBytes,       // PDF-bron (enkel type 'pdf')
  svgFromPdf,     // vectorpaden uit de PDF als SVG, met bronkleuren (enkel type 'pdf')
  colorSvg,       // bron-SVG met data-tkk-fill per vorm
  colors,         // ['#083213', '#fd6f3b'] — unieke vulkleuren in tekenvolgorde
  colorMap,       // { '#083213': 'color-3', '#fd6f3b': null } — bronhex → merkkleur-id
  colorMapManual, // Set van bronhexen die de gebruiker zelf koppelde
  bbox,           // { x, y, w, h } — rand van het logo in viewBox-coördinaten (passend formaat)
}
```

## Export-boomstructuur i.p.v. logo-preview

Het resultaatvlak rechts toont geen render van het logo (checkerboard-preview), maar een live meebewegende boomweergave van de exacte mapstructuur en bestandsnamen die in de ZIP terechtkomen. Reden: een render van één kleurvariant zei weinig over wat er werkelijk gegenereerd zou worden (meerdere logo's, meerdere varianten, digitaal + print); de boomstructuur beantwoordt direct de vraag "wat zit er straks in dat bestand?". Gebouwd via `buildExportTree()`, die exact dezelfde naamgevingslogica hergebruikt als de eigenlijke export (`exportZip()`) — beide moeten synchroon blijven bij wijzigingen.

## Bestandsnaamgeving

Patroon: `{klantnaam}-{rol}-{kleurversie}-{digitaal|print}-{detail}.{ext}`, bv. `tigron-primary-logo-groen-digitaal-0512px.png` of `tigron-primary-logo-zwart-print.eps`.

- Rol staat **altijd** in de bestandsnaam (niet enkel de mapnaam) — een bestand moet ook begrijpelijk zijn als het uit zijn map wordt gehaald.
- `digitaal`/`print` staat expliciet in elke bestandsnaam — voorkomt dat een klant een digitaal (RGB, lage/schermresolutie) bestand per ongeluk voor drukwerk gebruikt, ook zonder de mapstructuur-context.
- Eén PNG per kleurversie, zonder maat in de naam (`…-digitaal.png`, 3000 px langste zijde). Vroeger waren er vier maten (256–2048 px) plus een "print"-PNG; dat is bewust afgeslankt — zie "Bewust niet".
- Bekende open vraag, nog niet beslist: bestandsnamen zijn nu Nederlandstalig (`digitaal`, `print`, `op-wit`). Voor internationale klanten kan Engels een betere keuze zijn — bewust nog niet omgezet, wacht op een beslissing.

## Bewust niet (v1)

- Geen automatisch gegenereerde LEESMIJ-bestanden of merkkleuren-documentatie in de export — enkel de effectieve assets (zie SKILL.md).
- Geen automatische CMYK-conversie vanuit hex — enkel wat de gebruiker zelf invult, plus een optionele "indicatieve schatting"-knop is bewust géén onderdeel meer (verwijderd op vraag, om verwarring met een echte waarde te vermijden).
- **Monochrome versies van een meerkleurig logo worden niet aangepast**: alles wordt één kleur, dus overlappende vormen smelten samen (bv. twee overlappende bladeren worden één vlak). Bewust aanvaard — wie een officiële mono-versie uit het merkenboek heeft, uploadt die als apart logo met een eigen rol.
- **Geen CMYK uitlezen uit een PDF**: pdf.js zet CMYK-kleuren om naar RGB voor we ze zien, dus de originele CMYK uit een CMYK-PDF gaat verloren. Wel mogelijk met een eigen content-stream-parser, maar bewust niet gedaan: de CMYK komt altijd uit stap 03.
- **Geen reeks PNG-maten en geen print-PNG**: vroeger 256/512/1024/2048 px plus een 3000 px "print"-PNG per kleurversie. Die maten kwamen uit de wereld van iconen en favicons, die een vierkant met marge vragen — iets wat dit pakket sinds het passend formaat niet meer maakt. Web, presentaties en Office verkleinen zelf, dus één PNG van 3000 px dekt alle toepassingen. De print-PNG was bovendien RGB in een map die "CMYK_Print" heet; voor drukwerk zijn er de EPS en de PDF. Resultaat: ±45 → ±30 bestanden per logo.
- **Geen icoonpakket** (vierkant, voor favicons en profielfoto's in vaste maten): bewust uitgesteld; als het er komt, dan als apart, optioneel pakket voor het beeldmerk, niet als extra maten bij elk logo.
- **Geen PNG/JPG als upload-bron** (bewust verwijderd op vraag) — stap 02 aanvaardt enkel SVG en PDF. Elk ander bestandstype geeft nu een duidelijke foutmelding i.p.v. stilzwijgend een raster-logo aan te maken. Reden: een raster-bron levert nooit een echte vector-EPS of -SVG op (enkel het rastermasker-fallback, zie "Vector-EPS"), wat verwarrend was als iemand per ongeluk een PNG uploadde en een matige print-kwaliteit kreeg zonder dat duidelijk was waarom. PNG/JPG blijven wel gewoon geldige **export**-formaten (stap 05) — het gaat enkel om wat als bron aanvaard wordt.

## Getest

Volledige exportflow herhaaldelijk end-to-end getest in een gesimuleerde DOM-omgeving (jsdom + node-canvas): klantnaam, kleuren toevoegen/bewerken, meerdere logo's met vrije rol, alle pakket-opties, en de gegenereerde ZIP-inhoud vergeleken met de live boomweergave (moeten exact overeenkomen). EPS-output apart geverifieerd met Ghostscript (vorm + exacte CMYK-kleur).

## Toegankelijkheid

Volgt de WCAG-tabel uit SKILL.md: 24×24px icoon-knoppen (niet 17×17px), zichtbare focus-states, `aria-label` op elk icoon-only element, `for`/`id`-gekoppelde labels, geen `opacity` voor disabled-states (vaste gedempte kleuren + `pointer-events: none`), `prefers-reduced-motion` gerespecteerd, betekenisvolle `alt`-teksten.

Thumbnails zijn echte `<button>`-elementen; de verwijderknop staat ernaast in `.thumb-wrap`, niet erin. Een knop binnen een element dat zelf als knop werkt, is ongeldig, en in de praktijk ving de thumbnail Enter op, waardoor de verwijderknop met het toetsenbord niet werkte. De verwijderknop wordt verborgen via `opacity` (niet `display: none`) en verschijnt bij hover en focus. Na selecteren of verwijderen zet `focusLogoThumb()` de focus terug in de thumbnail-rij, zodat die niet naar het begin van de pagina springt.

## Deployment

Repo: [`kindkidnils/logo-assets-generator`](https://github.com/kindkidnils/logo-assets-generator) — gehost via GitHub Pages op de `main`-branch, root. Enkelvoudig HTML-bestand (`index.html`), geen build step.
