# TLCyzer Web

Quantitative Auswertung von Dünnschichtchromatogrammen (GPHF-Minilab-Verfahren) im Browser.
Reine Client-Anwendung: kein Server, kein Upload, kein Build-Schritt. Nach dem ersten Aufruf läuft sie offline.

## Verwendung

1. `index.html` im Browser öffnen (oder die gehostete Seite aufrufen).
2. Foto der UV-beleuchteten Platte laden — idealerweise aus der im Paper beschriebenen lichtdichten Holzbox.
3. Die vier Plattenecken setzen, zuschneiden.
4. Erkannte Flecken prüfen, Referenzen mit ihrem Sollgehalt markieren.
5. Gehalt berechnen, Ergebnis als JSON/CSV/PNG exportieren.

## Umgesetzte Pipeline

Nach Hauk et al., *Sci Rep* **12**, 13433 (2022), doi:10.1038/s41598-022-17527-y, Abschnitt „TLC imaging application":

| Schritt | Umsetzung |
|---|---|
| Plattenerkennung | Ecken manuell oder per Helligkeitsschwelle vorgeschlagen (Original: Hough-Transformation) |
| Entzerrung | Homographie aus 4 Punkten, bilineare Abtastung |
| Graustufen | `Y = 0.2126R + 0.7152G + 0.0722B` auf linearisiertem sRGB |
| Beleuchtungsmodell | 2D-Polynom 4. Grades, 15 Koeffizienten, kleinste Quadrate auf jedem 16. Pixel |
| Segmentierung | Schwellwert auf dem Residuum, morphologisches Öffnen, Connected Components, Filter nach Fläche und Seitenverhältnis |
| Fleckmitte | intensitätsgewichteter Schwerpunkt |
| Radius | größter Abstand vom Schwerpunkt zu einem Fleckpixel, plus 12 % |
| Integral | Summe der obersten 15 % der Pixelwerte im Kreis |
| Quantifizierung | lineare Regression Integral → Prozent über die Referenzflecken |

Die Termreihenfolge des Polynoms entspricht Gleichung (2) der Publikation, die Feldnamen im
JSON-Export (`agentName`, `x`, `y`, `radius`, `integrationValue`, `percentage`) denen der
Original-App. Das Paper verlangt ausdrücklich **lineare** RGB-Kanäle für die Graustufen; das ist
hier so umgesetzt, während die Original-App auf gammabehafteten Werten rechnet.

### Schwellwert

Das Paper schwellt am Mittelwert des hintergrundbereinigten Bildes. Dessen Rust-Umsetzung rechnet in
`u8` und schneidet die Subtraktion bei null ab, wodurch der Mittelwert zwischen Untergrund und Fleck
zu liegen kommt. Beide Regeln stehen zur Wahl:

* **Mittel + k·σ** (Vorgabe, `k = 2,0`) — arbeitet unabhängig vom Rauschniveau des Untergrunds.
* **Mittelwert (wie Paper)** — dieselbe Regel wie in der Publikation, mit dem Schnitt bei null vor
  der Mittelwertbildung.

Die Vorgabe fiel auf `k·σ`, weil die Paper-Regel bei rauscharmem Untergrund kippt: Dort liegt der
Mittelwert weit unter dem Fleckniveau, die Schwelle erfasst dann einen großen Teil der Platte und
die Flecken verschmelzen. In einer Testreihe über mehrere Rauschniveaus lieferte `k·σ` durchgehend
die richtige Fleckzahl und Abweichungen von 0,3 bis 3,7 Prozentpunkten, die Paper-Regel 3,0 bis 4,9
Prozentpunkte und auf der rauschfreien Platte gar kein verwertbares Ergebnis.

### Weitere bewusste Abweichungen

* **Morphologisches Öffnen.** Wie im Original, Radius standardmäßig 0,75 % der größten Bildkante,
  einstellbar. Entfernt Rauschinseln und trennt schwach verbundene Flecken.
* **Robuster Hintergrund (optional, aus).** Zweiter Fit unter Ausschluss der Fleckpixel; im Test
  sinkt der Fehler des Beleuchtungsmodells dadurch um etwa den Faktor 4. Nicht Teil der validierten
  Originalkette — für Vergleichbarkeit mit der Publikation ausgeschaltet lassen.
* **Koordinaten** sind auf [−1, 1] normiert. Mathematisch dasselbe Modell, numerisch erheblich
  gutartiger: Die Spanne der Diagonalelemente im 15×15-System sinkt von rund 10²⁴ auf 25.
* **Connected Components** mit 8er-Nachbarschaft (Original: 4er).
* **Fleckradius** deckt den ganzen Fleck ab, wie im Paper beschrieben. Die Original-App nimmt
  stattdessen den kleinsten Abstand zur Bounding-Box-Ecke und damit einen kleineren Kreis.
* **Integration** über den Kreis, wie im Paper. Die Original-App integriert über das umschließende
  Quadrat.
* Randberührende Komponenten (Plattenkante, Bleistiftmarkierungen) werden verworfen.

## Hosting

Alles ist statisch, es genügt ein beliebiger Static-Host.

**GitHub Pages**

```bash
git push -u origin main
```

Dann *Settings → Pages → Source: Deploy from a branch → main / (root)*. Nach ein bis zwei Minuten liegt die App
unter `https://<nutzer>.github.io/<repo>/`. HTTPS ist aktiv, damit funktionieren Kamera und Service Worker.
Alle Pfade sind relativ, ein Unterverzeichnis ist also kein Problem.

Gleichwertig kostenlos: Cloudflare Pages, Netlify, Vercel, Codeberg Pages, GitLab Pages.

Nach einer Änderung an den ausgelieferten Dateien die Version in `sw.js` (`CACHE`) hochzählen, damit
die alten Einträge verworfen werden. Seitenaufrufe laufen ohnehin zuerst über das Netz, eine still
veraltete Fassung sollte damit nicht entstehen.

## Lizenz

Die Original-App (github.com/TLCyzer/tlcyzer, Kotlin/Rust) steht unter GPL-3.0. Dieser Code ist eine eigenständige
Neuimplementierung nach der Verfahrensbeschreibung im Open-Access-Paper (CC BY 4.0) und übernimmt keinen Quellcode.
Um dem Geist des Projekts zu folgen, steht er ebenfalls unter **GPL-3.0**; der Lizenztext liegt als `LICENSE` bei.

## Hinweis

Screening-Werkzeug, kein Arzneibuchverfahren. Die Autoren empfehlen ausdrücklich, die Methode nur unter
wissenschaftlicher Aufsicht einzusetzen; auffällige Proben gehören in ein Labor mit kompendialer Analytik.
