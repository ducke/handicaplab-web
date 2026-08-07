# Landingpage auf den aktuellen App-Stand bringen

**Datum:** 2026-08-07
**Repo:** handicaplab-web
**Status:** Entwurf zur Review

## Ausgangslage

Die Landingpage (Redesign vom 2026-08-05, PR #4) zeigt den App-Stand vor den
iOS-PRs #14–#18. Es fehlen: „Meine Plätze" (Premium), die neue
Handicap-Verlaufskarte samt Liga-Formkurve, und alle fünf Screenshots zeigen
das alte UI. Außerdem hat die Seite keinen echten Call-to-Action — nur das
Badge „Bald im App Store".

Aus dem Beta-Test kamen zwei Onboarding-Befunde:
- Der Sandbox-Kauf in TestFlight erzwingt einen Apple-Passwort-Dialog ohne
  Passwortmanager-Zugriff (TestFlight-Artefakt, im App Store läuft der Kauf
  über Face ID/Touch ID).
- Der PDF-Import von golf.de ist je nach Browser unterschiedlich
  (Safari öffnet das PDF im Viewer → Teilen-Menü; Chrome speichert nach
  Downloads → Import über den Datei-Picker der App).

## Ziel

Die Seite spiegelt den tatsächlichen Funktionsstand von `origin/main` des
iOS-Repos wider, bekommt einen TestFlight-Beta-CTA und eine
browser-spezifische Import-Anleitung. Layout und Design-System bleiben
unverändert.

## Nicht im Scope

- Aktivierung des öffentlichen TestFlight-Links (läuft in einer anderen
  Session; hier wird der Link nur ausgelesen und eingebaut).
- WebView-Import des History Sheets (eigene Spec im iOS-Repo, PR #3 dort).
- Neue Sektionen oder Layout-Änderungen (Ansatz A: gezieltes Update).
- Eigener Screenshot der Plätze-Ansicht.

## 1. Screenshots — fünf Dateien, gleiche Namen und Maße

Quelle: App-Build von `origin/main` (iOS-Repo) im Simulator, **Demo-Modus**
(enthält seit PR #7 echte Ligarunden, alle Karten sind gefüllt).

| Datei | Inhalt | Zielmaß (px) |
|---|---|---|
| `assets/img/iphone-uebersicht.png` | Übersicht, iPhone | 660 × 1235 |
| `assets/img/ipad-uebersicht.png` | Übersicht mit Seitenleiste, iPad quer | 1440 × 846 |
| `assets/img/crop-hcpi.png` | Neue Handicap-Verlaufskarte | 900 × 410 |
| `assets/img/crop-kacheln.png` | Vier Statistik-Kacheln | 900 × 526 |
| `assets/img/crop-trend.png` | Score-Trend | 900 × 565 |

Vorgehen: Simulator-Screenshots (3×) aufnehmen, Crops aus dem
iPhone-Screenshot schneiden, mit `sips`/ImageMagick exakt auf die Zielmaße
skalieren. Die HTML-`width`/`height`-Attribute bleiben unverändert, dadurch
keine Layout-Verschiebung. `alt`-Texte an die neuen Bildinhalte anpassen
(u. a. der HCPI-Wert des Demo-Datensatzes).

## 2. Textänderungen in `index.html`

- **Bento-Zelle „Handicap-Index mit Verlauf":** beschreibt die eigenständige
  Verlaufskarte; Index und Low HCPI kommen direkt aus dem Sheet-Kopf statt
  aus einer Ableitung.
- **Premium-Liste:**
  - Neuer Punkt **„Meine Plätze"** — Ranking aller gespielten Plätze mit
    Abweichung vom eigenen Schnitt, Detailseite je Platz.
  - Liga-Punkt ergänzt um „Formkurve je Mannschaft".
  - „Plätze" aus dem Aufzählungstext des Detail-Analyse-Punkts entfernen
    (steckt jetzt im eigenen Punkt).
- Hero, Import-Schritte (3-Schritte-Sektion), Datenschutz-Sektion und
  Preisangaben bleiben unverändert.

## 3. TestFlight-CTA

- Link-Quelle: `asc testflight groups list --app 6797008248` → `publicLink`
  der Gruppe „Beta" (wird in anderer Session aktiviert).
- Einbau: In Hero und Abschluss-Sektion ein `btn`-Button
  **„Beta über TestFlight testen"** neben dem bestehenden Badge
  „Bald im App Store". Der Ghost-Button „So kommst du an dein Sheet" bleibt,
  rückt in der Hero hinter den neuen CTA.
- Fallback: Ist der Link beim Implementieren noch nicht aktiv, wird der
  Button fertig gebaut und das `href` als markierter Platzhalter
  (`href="#" data-todo="testflight-link"`) belassen; der PR merged erst,
  wenn der echte Link eingetragen ist.

## 4. Import-Anleitung auf `support.html`

Browser-spezifische Schritte ergänzen (die 3-Schritte-Sektion der Startseite
bleibt bewusst einfach und verlinkt weiter hierher):

- **Safari:** PDF öffnet im Viewer → Teilen → „HandicapLab". Hinweis, dass
  die App im Share-Sheet unter „Mehr" liegen kann und sich nach oben pinnen
  lässt.
- **Chrome (und andere Browser):** PDF landet in „Downloads" → in
  HandicapLab „PDF importieren" antippen und die Datei dort auswählen.
- Alternative für beide: „In Dateien sichern" und aus der App heraus öffnen.

## 5. TestFlight-Kaufnotiz (kleiner Extra-Punkt, iOS-seitig)

In die „What to Test"-Notes des nächsten Builds (Build 20 oder Folgebuild)
einen Hinweis aufnehmen: Der Kauf fragt in der Beta nach dem
Apple-Passwort — das ist TestFlight-spezifisch, im App Store genügt
Face ID/Touch ID. Umsetzung per
`asc builds test-notes create --build-id … --locale de-DE`.

## Verifikation

- Seite lokal im Browser prüfen: Desktop- und Mobile-Viewport, hell/dunkel.
- Bildmaße per `sips` gegen die Tabelle oben prüfen.
- Alle Links klicken (Support, Datenschutz, Impressum, TestFlight-CTA).
- `git diff` gegen `origin/main`: keine Änderungen außerhalb von
  `index.html`, `support.html`, `assets/img/*.png` und dieser Spec.

## Git

- Branch `feat/landing-aktueller-stand` (Worktree), PR gegen `main` im
  Web-Repo. Kein Direkt-Commit auf `main`.
