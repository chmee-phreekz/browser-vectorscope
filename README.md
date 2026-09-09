# Vectorscope

Ein einzelnes, selbstständiges HTML/JS/CSS-Tool, das das Live-Bild der Webcam als Vektorskop, Waveform-Monitor und Chromatizitätsdiagramm darstellt – plus Pixel-Colorpicker mit Abgleich gegen eine 24-Felder-Farbtafel. Läuft komplett im Browser, kein Server, kein Build-Schritt, keine externen Abhängigkeiten außer zwei Google-Fonts.

## Schnellstart

1. `vectorscope.html` im Browser öffnen (am besten Chrome/Edge – siehe [Browser-Kompatibilität](#browser-kompatibilität)).
2. Kamerazugriff erlauben.
3. Fertig – das Bild läuft live, alle Instrumente aktualisieren sich fortlaufend.

Kein Internetzugang nötig außer für den einmaligen Font-Ladevorgang; die App selbst braucht keine Serververbindung.

## Funktionsübersicht

### Kamera-Panel (links oben)
- Live-Vorschau (16:9, intern 960×540), Kameraauswahl bei mehreren Geräten
- **Standbild** – friert das aktuelle Frame ein
- **Spiegeln** – horizontale Spiegelung der Anzeige
- **Weichzeichnen** – Gauß-Blur auf das Rohbild, um Sensorrauschen vor der Analyse räumlich zu mitteln
- **Zoom ×2** – digitaler Center-Crop-Zoom; wirkt konsistent auf Vorschau, Colorpicker, Maskierung und alle Messinstrumente, da diese alle aus demselben verarbeiteten Frame lesen
- **Maskierung** – Rechteck im Bild aufziehen: alles außerhalb wird schwarz maskiert und fließt so auch nicht mehr in die Analyse ein. Einfacher Klick pickt weiterhin eine Farbe.
- **Kameraeinstellungen** – baut sich dynamisch aus dem, was Browser und Kamera über die `MediaTrackCapabilities`-API tatsächlich melden (Auflösung, Bildrate, Weißabgleich-Modus, Belichtung, Fokus, Hardware-Zoom, Helligkeit/Kontrast/Sättigung, Torch …). Nicht unterstützte Werte werden schlicht nicht angezeigt.
- **Weißabgleich-Leisten** – zwei kompakte Relativanzeigen (siehe [Weißabgleich-Schätzung](#weißabgleich-schätzung))

### Vektorskop / Y-Waveform / Farbraum & Temp. (rechts oben, als Tabs)
- **Vektorskop** – Y′UV-Chrominanzebene mit Zielmarken (R/G/B/C/M/Yl), Hauttonlinie, wählbarem Farbprofil und Gain-Regler
- **Y-Waveform** – Helligkeitsverlauf über die Bildbreite
- **Farbraum & Temp.** – vereinfachtes CIE-xy-Diagramm: farblich akkurat eingefärbter sRGB-Arbeitsbereich, sRGB-Gamut-Dreieck, Planckscher Kurvenzug (Schwarzkörper) und D65-Weißpunkt

Alle drei Diagramme können optional die 24 Farbtafel-Felder und die aktuelle Picker-Position als Marker einblenden (Buttons dafür sitzen bei der Farbtafel bzw. im Vektorskop-Tab).

### Colorpicker + Farbtafel / RGB+Luma-Diagramm (unten)
- Klick ins Kamerabild pickt eine Farbe und **verfolgt die Bildposition live weiter** – der Wert aktualisiert sich jeden Frame neu, nicht nur beim Klick
- Anzeige in HEX, RGB, Lab, U/V
- Automatischer Abgleich mit der nächstliegenden der 24 Farbtafel-Felder (Metrik: siehe unten)
- Balkendiagramm „Jetzt vs. Ideal" für R, G, B und Luma (Y)

## Technische Details

### Y′UV-Farbprofile
Umschaltbar zwischen **Rec.601**, **Rec.709** (Standard), **Rec.2020** und einer **klassischen** analogen PAL/NTSC-Variante. Die Kamera liefert immer physisches sRGB – die Umschaltung ändert nur, mit welchen Luma-Koeffizienten (Kr/Kb) daraus Y′/U′/V′ berechnet wird. Für 601/709/2020 wird die normierte Pb/Pr-Form verwendet (generalisiert über alle drei Standards hinweg); „Klassisch" nutzt stattdessen die historischen Konstanten 0.492/0.877.

### Farbfeld-Matching
Der „nächste" Farbtafel-Wert wird **nicht** per RGB- oder Lab-Distanz bestimmt, sondern per euklidischem Abstand im Y′UV-Raum (Luma + Vektorskop-Ebene) zum aktuell gewählten Profil. Ändert sich das Profil, wird der aktuell gepickte Wert automatisch neu bewertet.

### Weißabgleich-Schätzung
1. Gepicktes RGB → CIE-xy-Chromatizität (über die sRGB/D65-XYZ-Matrix)
2. **Farbtemperatur (CCT):** McCamy-Näherung (1992) aus der xy-Chromatizität
3. **Tint (Grün/Magenta):** **Duv** – der senkrechte Abstand des Punkts von der Planckschen Kurve im CIE-1960-uv-Raum (ANSI C78.377), positiv = Richtung Grün, negativ = Richtung Magenta. Das ist derselbe Wert, den Lichtmessgeräte oft als „+3G"/„-2M" anzeigen.

Beide Werte sind **nur bei annähernd achromatischen (grauen/weißen) Farben** aussagekräftig – das Tool rechnet trotzdem für jede Farbe einen Wert aus, die Interpretation liegt beim Nutzer.

Dargestellt werden sie **relativ zu D65** (6504K, Duv 0) auf zwei symmetrischen Leisten statt als Absolutwert – siehe [Einstellbare Variablen](#einstellbare-variablen) für die Skalierung.

### Planckscher Kurvenzug
Näherung nach Kim et al. (2002), gezeichnet ab 2500K (darunter wird die Näherung sichtbar ungenau/geknickt).

### CIE-Diagramm-Einfärbung
Jeder Pixel im Diagramm wird über die inverse sRGB/D65-Matrix zurück nach sRGB gerechnet (bei Y=1). Farben außerhalb des sRGB-Gamuts werden verhältniserhaltend skaliert statt hart geclippt, damit der Übergang am Gamut-Rand weich bleibt.

## Einstellbare Variablen

Alle folgenden Werte sind als benannte Konstanten im `<script>`-Block kommentiert und lassen sich direkt anpassen:

| Variable | Standardwert | Wirkung |
|---|---|---|
| `CHROMA_COLOR_ALPHA` | `0.9` | Deckkraft der akkuraten Farbeinfärbung im CIE-Diagramm (0 = aus) |
| `MAX_CCT_DEVIATION_MIRED` | `100` | Wie viel Mired-Abweichung von 6504K die Warm/Kalt-Leiste bis zum Rand ausschlägt |
| `MAX_DUV_DEVIATION` | `0.02` | Wie viel Duv-Abweichung die Grün/Magenta-Leiste bis zum Rand ausschlägt |
| `BLUR_PX` | `3` | Radius des Weichzeichnen-Effekts in Pixeln |
| `ZOOM_FACTOR` | `2` | Vergrößerungsfaktor des Center-Crop-Zooms |
| `SAMPLE_W` / `SAMPLE_H` | `160` / `90` | Auflösung der internen Sampling-Canvas für Vektorskop/Waveform (Performance vs. Detailgrad) |
| `BAR_MAX_H` | `145` | Maximale Balkenhöhe im RGB+Luma-Diagramm (px) |

Kleinere Deviation-Werte machen die Weißabgleich-Leisten empfindlicher (feiner aufgelöst), größere Werte toleranter (mehr Abweichung bis zum Anschlag am Leistenrand).

## Bekannte Einschränkungen

- **Farbtiefe pro Kanal** ist über Web-Kamera-APIs nicht abfragbar oder einstellbar – Browser liefern grundsätzlich 8-Bit pro Kanal an den Canvas, unabhängig davon, was der Sensor intern kann.
- **`MediaTrackCapabilities`** wird nicht von allen Browsern gleich gut unterstützt (Safari z. B. eingeschränkt) – die Kameraeinstellungen-Sektion zeigt dann entsprechend weniger oder gar keine Regler.
- Farbtafel-Referenzwerte sind gängige sRGB-Näherungen einer 24-Felder-Farbtafel, keine Herstellermessung.
- Das CIE-Diagramm ist ein vereinfachtes xy-Diagramm (sRGB-Dreieck + Planck-Kurve im Bereich x: 0.1–0.7 / y: 0–0.7), kein vollständiges CIE-1931-Hufeisen mit Spektrallinie.

## Browser-Kompatibilität

Empfohlen: aktuelles **Chrome** oder **Edge** (beste Unterstützung für `getUserMedia`, `MediaTrackCapabilities`, Canvas-`filter`). Firefox funktioniert für die Kernfunktionen, meldet aber weniger Kamera-Fähigkeiten. Safari/iOS: Grundfunktionen laufen, die erweiterten Kameraeinstellungen bleiben meist leer.

## Dateien

- `vectorscope.html` – die komplette Anwendung, eine einzelne Datei
- `README.md` – dieses Dokument
