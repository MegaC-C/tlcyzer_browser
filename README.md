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
| Beleuchtungsmodell | 2D-Polynom 4. Grades, 15 Koeffizienten, kleinste Quadrate auf 1/4-Auflösung |
| Segmentierung | Schwellwert auf dem Residuum, Connected Components (8er-Nachbarschaft), Filter nach Fläche und Seitenverhältnis |
| Fleckmitte | intensitätsgewichteter Schwerpunkt |
| Integral | Summe der obersten 15 % der Pixelwerte im Kreis |
| Quantifizierung | lineare Regression Integral → Prozent über die Referenzflecken |

### Bewusste Abweichungen

* **Schwelle.** Das Paper schwellt am Mittelwert des Residuums. Da der Mittelwert der Kleinste-Quadrate-Residuen
  konstruktionsbedingt bei null liegt, wird hier `Mittel + k·σ` mit einstellbarem `k` (Vorgabe 2,0) verwendet.
  Auf das Integral wirkt sich das kaum aus, weil ohnehin nur die obersten 15 % der Pixel summiert werden.
* **Robuster Hintergrund (optional, aus).** Zweiter Fit unter Ausschluss der Fleckpixel. Genauer, aber nicht Teil
  der validierten Originalkette — für Vergleichbarkeit mit der Publikation ausgeschaltet lassen.
* **Koordinaten** sind auf [−1, 1] normiert (numerisch stabiler, mathematisch dasselbe Modell).
* Randberührende Komponenten (Plattenkante, Bleistiftmarkierungen) werden verworfen.

## Hosting

Alles ist statisch, es genügt ein beliebiger Static-Host.

**GitHub Pages**

```bash
git init && git add . && git commit -m "TLCyzer Web"
git branch -M main
git remote add origin git@github.com:<nutzer>/<repo>.git
git push -u origin main
```

Dann *Settings → Pages → Source: Deploy from a branch → main / (root)*. Nach ein bis zwei Minuten liegt die App
unter `https://<nutzer>.github.io/<repo>/`. HTTPS ist aktiv, damit funktionieren Kamera und Service Worker.
Alle Pfade sind relativ, ein Unterverzeichnis ist also kein Problem.

Gleichwertig kostenlos: Cloudflare Pages, Netlify, Vercel, Codeberg Pages, GitLab Pages.

## Lizenz

Die Original-App (github.com/TLCyzer/tlcyzer, Kotlin/Rust) steht unter GPL-3.0. Dieser Code ist eine eigenständige
Neuimplementierung nach der Verfahrensbeschreibung im Open-Access-Paper (CC BY 4.0) und übernimmt keinen Quellcode.
Um dem Geist des Projekts zu folgen, wird er ebenfalls unter **GPL-3.0** veröffentlicht — bei einer Veröffentlichung
`LICENSE` ergänzen und die Herkunft der Methode nennen.

## Hinweis

Screening-Werkzeug, kein Arzneibuchverfahren. Die Autoren empfehlen ausdrücklich, die Methode nur unter
wissenschaftlicher Aufsicht einzusetzen; auffällige Proben gehören in ein Labor mit kompendialer Analytik.
