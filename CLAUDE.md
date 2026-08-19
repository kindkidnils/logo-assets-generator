# Logo Assets Generator

Single-page browser tool om een compleet, geordend logo-asset-pack te genereren per klant/project: digitale formaten (SVG/PNG/JPG/WebP in meerdere formaten en kleurversies) én print-klare bestanden (300dpi PNG + echte vector-CMYK-EPS), verpakt in één ZIP.

Eerder bekend als "LogoKit" (herbouwd/herbenoemd tijdens de skill-ontwikkeling — zie git-historiek als die bewaard is, anders begin bij deze versie).

## Doel

Vervangt het handmatig genereren van een logo-pakket voor een klant of intern project: favicon-achtige formaten, transparante PNG's, print-EPS met correcte CMYK — allemaal in één keer, consistent benoemd, in een voorspelbare mapstructuur.

## Stack

Vanilla HTML/CSS/JS + Canvas API, gebouwd op `assets/template.html` uit de `tkk-tool-template`-skill (spoor A). Eén bestand, geen build step. Externe dependency: JSZip (CDN) voor het samenstellen van de ZIP, pdf.js (CDN, dynamic import) enkel wanneer een PDF geüpload wordt.

## Kernfunctionaliteit

1. **Klantnaam** (stap 01) — vormt de basis van elke bestandsnaam in de export, nooit de naam van het geüploade bronbestand.
2. **Logo's uploaden** (SVG, PDF of PNG/JPG) — elk logo krijgt een vrij intypbare rol (met suggesties via `<datalist>`, geen vaste dropdown) die de mapnaam bepaalt. Een "+"-tegel naast bestaande logo's maakt duidelijk dat je er meer kan toevoegen.
3. **Merkkleuren** (naam + hex + exacte CMYK + optioneel Pantone) — de CMYK-waarden worden letterlijk overgenomen, nooit automatisch geschat vanuit hex (browsers kunnen dat niet betrouwbaar zonder ICC-profiel).
4. **Kleurversies per asset** — dynamisch: elke toegevoegde merkkleur plus zwart/wit worden automatisch aangeboden als aan te vinken variant.
5. **Pakket-opties** — SVG/PNG/JPG/WebP voor digitaal, plus een print-map met 300dpi PNG en een EPS per vlakke kleurversie.

## Vector-EPS: hoe de CMYK-precisie werkt

Browsers kunnen RGB niet betrouwbaar naar CMYK omrekenen (geen ICC-profiel) — dus in plaats van te proberen te converteren, wordt de vorm van het logo (silhouet) apart vastgelegd en met PostScript's `setcmykcolor` ingekleurd met de CMYK-waarde die de gebruiker letterlijk heeft ingevuld. Twee routes, afhankelijk van het bronformaat:

- **SVG- of PDF-bron**: de vectorpaden worden zelf omgezet naar PostScript-paden (`moveto`/`lineto`/`curveto`) — een echt, oneindig schaalbaar vectorbestand. Voor PDF gebeurt dit via pdf.js' operator-list (de al-uitgepakte tekencommando's), niet via een eigen PDF-parser.
- **Enkel PNG/JPG-bron** (geen vectorbron): valt terug op een hogeresolutie rastermasker (1500px) ingekleurd via `imagemask` — nog steeds de exacte CMYK-waarde, maar geen oneindig schaalbaar vectorpad.

Getest met Ghostscript: de gerenderde CMYK-waarden kwamen exact overeen met de ingevoerde waarden (tot op 8-bit-afrondingsniveau).

Diezelfde PDF-vectorpaden (via `collectPdfPaintOps()`) voeden ook de **digitale SVG-export** (`buildVectorSVGFromPDF()`) — een PDF-bron levert dus, net als een SVG-bron, een echte `01_SVG`-map op in plaats van enkel raster-formaten. De functie schrijft dezelfde paden als SVG `<path d="…">` i.p.v. PostScript, met `fill="#000000"` als placeholder; de bestaande `recolorSvgText()` herkleurt 'm daarna per variant, exact zoals bij een geüploade SVG.

## Export-boomstructuur i.p.v. logo-preview

Het resultaatvlak rechts toont geen render van het logo (checkerboard-preview), maar een live meebewegende boomweergave van de exacte mapstructuur en bestandsnamen die in de ZIP terechtkomen. Reden: een render van één kleurvariant zei weinig over wat er werkelijk gegenereerd zou worden (meerdere logo's, meerdere varianten, digitaal + print); de boomstructuur beantwoordt direct de vraag "wat zit er straks in dat bestand?". Gebouwd via `buildExportTree()`, die exact dezelfde naamgevingslogica hergebruikt als de eigenlijke export (`exportZip()`) — beide moeten synchroon blijven bij wijzigingen.

## Bestandsnaamgeving

Patroon: `{klantnaam}-{rol}-{kleurversie}-{digitaal|print}-{detail}.{ext}`, bv. `tigron-primary-logo-groen-digitaal-0512px.png` of `tigron-primary-logo-zwart-print.eps`.

- Rol staat **altijd** in de bestandsnaam (niet enkel de mapnaam) — een bestand moet ook begrijpelijk zijn als het uit zijn map wordt gehaald.
- `digitaal`/`print` staat expliciet in elke bestandsnaam — voorkomt dat een klant een digitaal (RGB, lage/schermresolutie) bestand per ongeluk voor drukwerk gebruikt, ook zonder de mapstructuur-context.
- PNG-formaten zijn opgepad tot 4 cijfers (`0256px`, `0512px`, `1024px`, `2048px`) zodat ze alfabetisch (zoals Finder/Verkenner standaard sorteert) ook numeriek correct sorteren — zonder opvulling sorteren "1024"/"2048" vóór "256"/"512".
- Bekende open vraag, nog niet beslist: bestandsnamen zijn nu Nederlandstalig (`digitaal`, `print`, `op-wit`). Voor internationale klanten kan Engels een betere keuze zijn — bewust nog niet omgezet, wacht op een beslissing.

## Bewust niet (v1)

- Geen automatisch gegenereerde LEESMIJ-bestanden of merkkleuren-documentatie in de export — enkel de effectieve assets (zie SKILL.md).
- Geen automatische CMYK-conversie vanuit hex — enkel wat de gebruiker zelf invult, plus een optionele "indicatieve schatting"-knop is bewust géén onderdeel meer (verwijderd op vraag, om verwarring met een echte waarde te vermijden).
- Geen echte vectorisatie van raster-bronnen (PNG/JPG) — enkel SVG/PDF-bronnen krijgen een vector-EPS.

## Getest

Volledige exportflow herhaaldelijk end-to-end getest in een gesimuleerde DOM-omgeving (jsdom + node-canvas): klantnaam, kleuren toevoegen/bewerken, meerdere logo's met vrije rol, alle pakket-opties, en de gegenereerde ZIP-inhoud vergeleken met de live boomweergave (moeten exact overeenkomen). EPS-output apart geverifieerd met Ghostscript (vorm + exacte CMYK-kleur).

## Toegankelijkheid

Volgt de WCAG-tabel uit SKILL.md: 24×24px icoon-knoppen (niet 17×17px), zichtbare focus-states, `aria-label` op elk icoon-only element, `for`/`id`-gekoppelde labels, geen `opacity` voor disabled-states (vaste gedempte kleuren + `pointer-events: none`), `prefers-reduced-motion` gerespecteerd, betekenisvolle `alt`-teksten.

## Deployment

Repo: [`kindkidnils/logo-assets-generator`](https://github.com/kindkidnils/logo-assets-generator) — gehost via GitHub Pages op de `main`-branch, root. Enkelvoudig HTML-bestand (`index.html`), geen build step.
