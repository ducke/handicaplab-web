# Landingpage auf den aktuellen App-Stand — Implementierungsplan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Die Landingpage handicaplab.app zeigt den Funktionsstand von `origin/main` des iOS-Repos: neue Screenshots aus dem aktuellen Build, „Meine Plätze" und Verlaufskarte im Text, TestFlight-Beta-CTA, browser-spezifische Import-Anleitung.

**Architecture:** Reines Content-Update im bestehenden statischen HTML (GitHub Pages). Screenshots entstehen aus einem Simulator-Build des iOS-Repos (`origin/main`) über den UITest-Seed-Mechanismus der App (`-uitest-seed` + `UITEST_SEED_PDF_PATH`), damit kein Demo-Banner im Bild ist. Bildbearbeitung mit `sips`/ImageMagick auf die exakten Maße der bestehenden Dateien — HTML-Attribute und CSS bleiben unangetastet.

**Tech Stack:** Statisches HTML/CSS (keine Build-Pipeline), `xcodebuild`/`simctl`, `sips` oder ImageMagick (`magick`), `asc`-CLI, `gh`.

## Global Constraints

- Web-Worktree: `/private/tmp/claude-501/-Users-manuelhenke-dev-ducke-handicaplab-ios/aca46252-2c0b-4da9-be5c-743f81ab44a6/scratchpad/wt-web` (Branch `feat/landing-aktueller-stand`). Im Folgenden `$WT_WEB`.
- iOS-Repo: `/Users/manuelhenke/dev/ducke/handicaplab-ios`. Screenshots IMMER von `origin/main`, nie vom lokalen `main` (19 Commits alt). Kein Commit im iOS-Hauptverzeichnis.
- Nur diese Dateien im Web-Repo ändern: `index.html`, `support.html`, `assets/img/*.png` (plus Spec/Plan unter `docs/superpowers/`).
- Ziel-Bildmaße exakt: `iphone-uebersicht.png` 660×1235, `ipad-uebersicht.png` 1440×846, `crop-hcpi.png` 900×410, `crop-kacheln.png` 900×526, `crop-trend.png` 900×565.
- Simulator-Appearance: **dark** (wie die bisherigen Seiten-Screenshots), Statusleiste per `simctl status_bar` auf 9:41, volles WLAN, voller Akku.
- Copy auf Deutsch, Ton wie Bestand (nüchtern, zweite Person Singular).
- Commit-Typen: feat/fix/docs/chore; keine Attribution-Fußzeile.
- Der PR wird erst gemerged, wenn der echte TestFlight-Link eingetragen ist (kein `data-todo` auf `main`).

---

### Task 1: Rohe Simulator-Screenshots erzeugen (iPhone + iPad)

**Files:**
- Create: `$SCRATCH/shots/iphone-raw.png`, `$SCRATCH/shots/ipad-raw.png` (Scratchpad, nicht committen) — `$SCRATCH` = `/private/tmp/claude-501/-Users-manuelhenke-dev-ducke-handicaplab-ios/aca46252-2c0b-4da9-be5c-743f81ab44a6/scratchpad`
- Nutzt: iOS-Worktree auf `origin/main` (read-only)

**Interfaces:**
- Produces: zwei rohe PNG-Screenshots der Übersicht (iPhone hochkant, iPad quer mit Seitenleiste), dunkles Design, ohne Demo-Banner, mit gefüllten Karten (inkl. neuer Handicap-Verlaufskarte).

- [ ] **Step 1: iOS-Worktree auf origin/main anlegen**

```bash
cd /Users/manuelhenke/dev/ducke/handicaplab-ios
git fetch origin -q
git worktree add "$SCRATCH/wt-shots" --detach origin/main
```

- [ ] **Step 2: Simulatoren wählen und booten**

```bash
xcrun simctl list devices available | grep -iE "iphone 16 pro|ipad pro"
```
Erwartet: je ein Gerät „iPhone 16 Pro" und „iPad Pro 13-inch (M4)" (bei Abweichung das nächstliegende Pro-Modell nehmen und die UDIDs in den Folgeschritten verwenden).

```bash
xcrun simctl boot "iPhone 16 Pro" || true
xcrun simctl bootstatus "iPhone 16 Pro"
xcrun simctl ui "iPhone 16 Pro" appearance dark
xcrun simctl status_bar "iPhone 16 Pro" override --time "9:41" --batteryState charged --batteryLevel 100 --wifiBars 3 --cellularMode notSupported
```

- [ ] **Step 3: App für Simulator bauen und installieren**

```bash
cd "$SCRATCH/wt-shots/HandicapLabApp"
xcodebuild -project HandicapLabApp.xcodeproj -scheme HandicapLabApp \
  -destination "platform=iOS Simulator,name=iPhone 16 Pro" \
  -derivedDataPath "$SCRATCH/dd-shots" build | tail -5
APP="$SCRATCH/dd-shots/Build/Products/Debug-iphonesimulator/HandicapLabApp.app"
xcrun simctl install "iPhone 16 Pro" "$APP"
```
Erwartet: `BUILD SUCCEEDED`, Install ohne Fehler.

- [ ] **Step 4: App mit Seed-Daten starten (kein Demo-Banner)**

Der Seed-Mechanismus ist derselbe wie in `HandicapLabAppUITests/DesignBaselineTests.swift`: Launch-Argumente `-uitest-reset -uitest-seed` plus Env `UITEST_SEED_PDF_PATH`. Als Sheet dient das Beispiel-Sheet mit Ligarunden aus dem App-Bundle:

```bash
PDF="$SCRATCH/wt-shots/HandicapLabApp/HandicapLabApp/Resources/beispiel-sheet.pdf"
SIMCTL_CHILD_UITEST_SEED_PDF_PATH="$PDF" \
  xcrun simctl launch --terminate-running-process "iPhone 16 Pro" \
  dev.ducke.handicaplab -uitest-reset -uitest-seed
sleep 5
```
Erwartet: App startet auf dem Übersichts-Tab mit gefüllten Karten. KEIN „BEISPIELDATEN"-Band (das erscheint nur im echten Demo-Modus über `demoStarten()`).

- [ ] **Step 5: iPhone-Screenshot aufnehmen und prüfen**

```bash
mkdir -p "$SCRATCH/shots"
xcrun simctl io "iPhone 16 Pro" screenshot "$SCRATCH/shots/iphone-raw.png"
```
Screenshot mit Read ansehen und prüfen: Übersicht sichtbar, oben die Handicap-Verlaufskarte (eigene Karte mit Kurve), darunter vier Kacheln, darunter Score-Trend; kein Demo-Banner; dunkles Design; Uhrzeit 9:41. Falls die Verlaufskarte nicht im sichtbaren Bereich liegt: nicht scrollen — der Hero zeigt die Startansicht, so wie sie beim Öffnen aussieht.

- [ ] **Step 6: Dasselbe für das iPad (Querformat)**

```bash
xcrun simctl boot "iPad Pro 13-inch (M4)" || true
xcrun simctl bootstatus "iPad Pro 13-inch (M4)"
xcrun simctl ui "iPad Pro 13-inch (M4)" appearance dark
xcrun simctl status_bar "iPad Pro 13-inch (M4)" override --time "9:41" --batteryState charged --batteryLevel 100 --wifiBars 3 --cellularMode notSupported
xcodebuild -project HandicapLabApp.xcodeproj -scheme HandicapLabApp \
  -destination "platform=iOS Simulator,name=iPad Pro 13-inch (M4)" \
  -derivedDataPath "$SCRATCH/dd-shots" build | tail -3
xcrun simctl install "iPad Pro 13-inch (M4)" "$APP"   # gleicher Pfad wie in Step 3, Debug-iphonesimulator gilt für beide Geräte
SIMCTL_CHILD_UITEST_SEED_PDF_PATH="$PDF" \
  xcrun simctl launch --terminate-running-process "iPad Pro 13-inch (M4)" \
  dev.ducke.handicaplab -uitest-reset -uitest-seed
sleep 5
xcrun simctl io "iPad Pro 13-inch (M4)" screenshot "$SCRATCH/shots/ipad-raw.png"
```
Wichtig: Das iPad muss im **Querformat** stehen (Seitenleiste sichtbar). Falls der Screenshot hochkant ist: im Simulator `Device → Rotate Left` (oder `osascript`-frei per Hardware-Menü) und Screenshot wiederholen. Mit Read prüfen: Seitenleiste links, Übersicht rechts, kein Demo-Banner.

- [ ] **Step 7: Kein Commit** — Rohbilder bleiben im Scratchpad. iOS-Worktree noch stehen lassen (wird in Task 8 für die Build-Nummer nicht gebraucht, kann aber erst nach Task 2 weg).

---

### Task 2: Bilder auf Zielmaße bringen und im Web-Repo ersetzen

**Files:**
- Modify: `$WT_WEB/assets/img/iphone-uebersicht.png`, `ipad-uebersicht.png`, `crop-hcpi.png`, `crop-kacheln.png`, `crop-trend.png`
- Modify: `$WT_WEB/index.html` (nur `alt`-Attribute der fünf Bilder)

**Interfaces:**
- Consumes: `$SCRATCH/shots/iphone-raw.png`, `$SCRATCH/shots/ipad-raw.png` aus Task 1.
- Produces: fünf PNGs in exakt den Maßen aus den Global Constraints.

- [ ] **Step 1: Geräte-Bilder skalieren und beschneiden**

Die Zielmaße sind schmaler-höher als das Rohbild — erst auf Zielbreite skalieren, dann von oben auf Zielhöhe beschneiden (`-gravity north`), damit Verlaufskarte und Kacheln im Bild bleiben:

```bash
cd "$SCRATCH/shots"
magick iphone-raw.png -resize 660x -gravity north -crop 660x1235+0+0 +repage iphone-uebersicht.png
magick ipad-raw.png   -resize 1440x -gravity north -crop 1440x846+0+0 +repage ipad-uebersicht.png
```
(Ist `magick` nicht installiert: `sips --resampleWidth 660 …` und dann `sips -c 1235 660 …` — `sips -c` beschneidet mittig; für Top-Crop mit `magick` arbeiten oder vorher per `sips --cropOffset 0 0` prüfen. Notfalls `brew install imagemagick`.)

- [ ] **Step 2: Drei Crops aus dem iPhone-Rohbild schneiden**

Die drei Crops zeigen: die neue Handicap-Verlaufskarte (`crop-hcpi`), die vier Statistik-Kacheln (`crop-kacheln`), den Score-Trend (`crop-trend`). Vorgehen: Rohbild mit Read ansehen, die Pixel-Bereiche der drei Karten ablesen (Kartenränder ein paar Pixel großzügig fassen), dann je Karte:

```bash
# <B>x<H>+<X>+<Y> = je Karte abgelesener Bereich im Rohbild
magick iphone-raw.png -crop <B>x<H>+<X>+<Y> +repage -resize 900x tmp-hcpi.png
magick tmp-hcpi.png    -gravity center -crop 900x410+0+0 +repage crop-hcpi.png

magick iphone-raw.png -crop <B>x<H>+<X>+<Y> +repage -resize 900x tmp-kacheln.png
magick tmp-kacheln.png -gravity center -crop 900x526+0+0 +repage crop-kacheln.png

magick iphone-raw.png -crop <B>x<H>+<X>+<Y> +repage -resize 900x tmp-trend.png
magick tmp-trend.png   -gravity center -crop 900x565+0+0 +repage crop-trend.png
```
Falls der Score-Trend im Startzustand unterhalb des sichtbaren Bereichs liegt: in der App scrollen (`xcrun simctl io … screenshot` nach einem Swipe per Simulator-Fenster) und einen zweiten Roh-Screenshot als Crop-Quelle nehmen.

- [ ] **Step 3: Maße verifizieren**

```bash
for f in iphone-uebersicht.png ipad-uebersicht.png crop-hcpi.png crop-kacheln.png crop-trend.png; do
  sips -g pixelWidth -g pixelHeight "$f" | tr '\n' ' '; echo;
done
```
Erwartet exakt: 660×1235, 1440×846, 900×410, 900×526, 900×565. Jedes Bild einmal mit Read ansehen: richtiger Ausschnitt, nichts Abgeschnittenes mitten im Text.

- [ ] **Step 4: Ins Web-Repo kopieren und alt-Texte anpassen**

```bash
cp iphone-uebersicht.png ipad-uebersicht.png crop-hcpi.png crop-kacheln.png crop-trend.png "$WT_WEB/assets/img/"
```
In `$WT_WEB/index.html` die fünf `alt`-Texte an den neuen Bildinhalt anpassen. Der HCPI-Wert des Beispiel-Sheets steht im Screenshot — den echten Wert einsetzen (im Folgenden `X,X`):

- `iphone-uebersicht.png`: `Übersicht in HandicapLab: Handicap-Verlaufskarte mit Index X,X, vier Statistik-Kacheln und das Score-Trend-Diagramm.`
- `crop-hcpi.png`: `Handicap-Verlaufskarte mit Index X,X, Low HCPI und Kurve über zwölf Monate.`
- `crop-kacheln.png`: unverändert lassen (Inhalt gleich), nur prüfen.
- `crop-trend.png`: unverändert lassen, nur prüfen.
- `ipad-uebersicht.png`: `HandicapLab auf dem iPad: Seitenleiste mit Navigation, rechts Handicap-Verlaufskarte, Statistik-Kacheln und Score-Trend nebeneinander.`

`width`/`height`-Attribute NICHT ändern.

- [ ] **Step 5: Commit**

```bash
cd "$WT_WEB"
git add assets/img/*.png index.html
git commit -m "feat(web): Screenshots vom aktuellen App-Stand (Verlaufskarte, Demo-Datensatz)"
```

---

### Task 3: Funktions- und Premium-Texte in index.html aktualisieren

**Files:**
- Modify: `$WT_WEB/index.html` (Bento-Zelle 1, Premium-Liste, Detail-Analyse-Punkt)

**Interfaces:**
- Consumes: nichts aus anderen Tasks (unabhängig von Task 1/2 ausführbar).
- Produces: aktualisierte Copy; keine neuen Klassen, IDs oder Strukturelemente.

- [ ] **Step 1: Bento-Zelle „Handicap-Index mit Verlauf" ersetzen**

In `index.html`, Zelle `cell-lead` (um Zeile 100): Überschrift und Absatz ersetzen durch:

```html
<h3>Handicap-Verlauf als eigene Karte</h3>
<p>Index und Low HCPI kommen direkt aus dem Sheet-Kopf, nicht aus einer
  Schätzung. Darunter die Verlaufskurve der letzten zwölf Monate.</p>
```

- [ ] **Step 2: Premium-Liste erweitern und präzisieren**

Im `plan-premium`-Block (um Zeile 172): den Detail-Analyse-Punkt ändern von
`Heim gegen Auswärts, privat gegen Turnier, Saisonverlauf, Plätze, Wochentage` zu
`Heim gegen Auswärts, RPR gegen Turnier, Saisonverlauf, Wochentage`.

Direkt danach als neues `<li>` (SVG-Häkchen identisch zu den Nachbarn kopieren):

```html
<li><svg viewBox="0 0 256 256" fill="currentColor" aria-hidden="true"><path d="M229.66,77.66l-128,128a8,8,0,0,1-11.32,0l-56-56a8,8,0,0,1,11.32-11.32L96,188.69,218.34,66.34a8,8,0,0,1,11.32,11.32Z"/></svg><span><b>Meine Plätze</b>Alle gespielten Plätze im Vergleich zum eigenen Schnitt, mit Detailseite je Platz</span></li>
```

Den Liga-Punkt ändern von `Mannschaftsrunden inklusive Team-Auswertung` zu
`Mannschaftsrunden mit Team-Auswertung und Formkurve je Mannschaft`.

- [ ] **Step 3: Sichtprüfung**

```bash
cd "$WT_WEB" && python3 -m http.server 8321 &
```
`http://localhost:8321` im Browser öffnen: Premium-Karte hat jetzt 5 Punkte, Zeilenumbrüche ok, nichts überläuft. Server danach beenden.

- [ ] **Step 4: Commit**

```bash
cd "$WT_WEB"
git add index.html
git commit -m "feat(web): Meine Plätze und Verlaufskarte im Funktions- und Premium-Text"
```

---

### Task 4: TestFlight-Beta-CTA einbauen

**Files:**
- Modify: `$WT_WEB/index.html` (Hero-`hero-actions`, Abschluss-Sektion `close`)

**Interfaces:**
- Consumes: öffentlichen TestFlight-Link aus App Store Connect (wird in anderer Session aktiviert).
- Produces: CTA-Button an zwei Stellen; Badge „Bald im App Store" bleibt.

- [ ] **Step 1: Link auslesen**

```bash
asc testflight groups list --app 6797008248 --paginate | python3 -c "
import json,sys
for g in json.load(sys.stdin)['data']:
    a=g['attributes']
    print(a['name'], a.get('publicLinkEnabled'), a.get('publicLink'))"
```
Erwartet: Zeile `Beta True https://testflight.apple.com/join/XXXXXXXX`. Wenn `publicLink` noch `None` ist: Platzhalter-Variante aus Step 2 nutzen und am Ende des Tasks vermerken, dass der Link vor dem Merge nachgetragen werden muss.

- [ ] **Step 2: Buttons einbauen**

In der Hero (`hero-actions`, um Zeile 44): VOR dem bestehenden Ghost-Button einfügen (`LINK` = echter Link; Fallback: `href="#" data-todo="testflight-link"`):

```html
<a class="btn btn-primary" href="LINK">Beta über TestFlight testen</a>
```

Das `status`-Badge „Bald im App Store" bleibt an erster Stelle. Dieselbe Zeile zusätzlich in der Abschluss-Sektion (`close`, um Zeile 228) vor dem Mail-Ghost-Button einfügen. Den Abschluss-Lede-Satz ändern von
`Ab iOS 18. Bis zur Freigabe im App Store erreichst du mich per Mail.` zu
`Ab iOS 18. Bis zur Freigabe im App Store kannst du die Beta über TestFlight testen — oder du schreibst mir.`

- [ ] **Step 3: Sichtprüfung im Browser** (wie Task 3 Step 3): Button-Reihenfolge Hero = Badge, Primary-CTA, Ghost; auf schmalem Viewport (390 px) umbrechen die Actions sauber untereinander.

- [ ] **Step 4: Commit**

```bash
cd "$WT_WEB"
git add index.html
git commit -m "feat(web): TestFlight-Beta als Call-to-Action in Hero und Abschluss"
```

---

### Task 5: Browser-spezifische Import-Anleitung auf support.html

**Files:**
- Modify: `$WT_WEB/support.html` (Abschnitt „Wo finde ich mein Handicap History Sheet?", Premium-Absatz)

**Interfaces:**
- Consumes: nichts.
- Produces: getrennte Schrittfolgen für Safari und Chrome/andere; Premium-Absatz nennt Plätze.

- [ ] **Step 1: Schritt 3–4 der Anleitung durch browser-spezifische Wege ersetzen**

In `support.html` die beiden `<li>` (Zeilen 45–47, „Sichere es …" und „Öffne die Datei …") ersetzen durch ein einzelnes `<li>`:

```html
<li>Wie es weitergeht, hängt vom Browser ab — beide Wege unten.</li>
```

Direkt nach dem `</ol>` einfügen:

```html
<h3>Safari</h3>
<p>Safari öffnet das PDF direkt im Viewer. Tippe auf
<strong>Teilen → HandicapLab</strong>. Taucht HandicapLab nicht sofort auf,
verbirgt es sich hinter <strong>„Mehr"</strong> in der App-Zeile — dort
kannst du es auch anpinnen, dann steht es beim nächsten Mal vorn.</p>

<h3>Chrome und andere Browser</h3>
<p>Chrome legt das PDF unter <strong>Downloads</strong> ab, ohne es zu
öffnen. Öffne HandicapLab, tippe auf <strong>„PDF importieren"</strong> und
wähle die Datei unter „Zuletzt benutzt" oder im Ordner Downloads aus.</p>

<p>Für beide Wege gilt: Du kannst das PDF auch über
<strong>„In Dateien sichern"</strong> ablegen und später aus der App heraus
öffnen.</p>
```

- [ ] **Step 2: Premium-Absatz auf Stand bringen**

Den Absatz unter „Was ist in Premium enthalten?" (Zeile 66) ersetzen durch:

```html
<p>Detail-Analyse (Heim/Auswärts, RPR/Turnier, Saisonverlauf, Wochentage),
Meine Plätze mit einer Detailseite je Platz, Liga-Statistiken mit
Team-Auswertung und Formkurve sowie das durchsuchbare Rundenbuch. Es ist ein
einmaliger Kauf, kein Abonnement. Auf einem neuen Gerät stellst du den Kauf
über „Käufe wiederherstellen“ in den Einstellungen der App wieder her.</p>
```

- [ ] **Step 3: Sichtprüfung im Browser** (`http://localhost:8321/support.html`): Zwischenüberschriften greifen die `prose`-Styles, Ablauf liest sich schlüssig von Schritt 1 bis zu den Browser-Wegen.

- [ ] **Step 4: Commit**

```bash
cd "$WT_WEB"
git add support.html
git commit -m "feat(web): Import-Anleitung nach Browser getrennt, Premium-Absatz aktualisiert"
```

---

### Task 6: Gesamt-Verifikation und PR

**Files:**
- Keine neuen Änderungen (nur Prüfung; Fixes fließen als `fix(web):`-Commits nach).

**Interfaces:**
- Consumes: alle vorigen Tasks.

- [ ] **Step 1: Diff-Scope prüfen**

```bash
cd "$WT_WEB"
git diff --stat origin/main...HEAD
```
Erwartet: NUR `index.html`, `support.html`, `assets/img/*.png`, `docs/superpowers/**`. Alles andere = Fehler, zurückbauen.

- [ ] **Step 2: Seite komplett durchklicken**

`python3 -m http.server 8321` und im Browser: Startseite (Desktop + 390 px), alle Anker (`#funktionen`, `#premium`, `#datenschutz`), Support, Datenschutz, Impressum, Beispiel-PDF-Link, TestFlight-Button (führt zum echten Link bzw. ist als Platzhalter markiert). Bildmaße final:

```bash
for f in assets/img/*.png; do sips -g pixelWidth -g pixelHeight "$f" | tr '\n' ' '; echo; done
```

- [ ] **Step 3: PR erstellen**

```bash
cd "$WT_WEB"
git push -u origin feat/landing-aktueller-stand
gh pr create --title "feat: Landingpage auf den aktuellen App-Stand" --body "$(cat <<'EOF'
## Was
- Screenshots neu vom origin/main-Build (Handicap-Verlaufskarte sichtbar), gleiche Maße wie bisher
- Meine Plätze + Formkurve je Mannschaft in der Premium-Liste, RPR statt privat
- TestFlight-Beta-CTA in Hero und Abschluss
- support.html: Import-Anleitung nach Browser (Safari/Chrome) getrennt

## Test
- [ ] Bildmaße identisch zu vorher (Layout unverändert)
- [ ] Seite auf Desktop + Mobile geprüft, hell/dunkel
- [ ] TestFlight-Link führt in die Beta-Gruppe

Spec: docs/superpowers/specs/2026-08-07-landingpage-aktueller-stand-design.md
EOF
)"
```
Falls der TestFlight-Link noch Platzhalter ist: PR als **Draft** (`--draft`) anlegen und im Body einen Hinweis ergänzen; erst nach Nachtrag des Links ready markieren.

- [ ] **Step 4: Aufräumen**

```bash
git worktree remove "$SCRATCH/wt-shots" --force
cd /Users/manuelhenke/dev/ducke/handicaplab-ios && git worktree prune
```

---

### Task 7: TestFlight-Kaufnotiz in die What-to-Test-Notes

**Files:**
- Keine Repo-Dateien — reine ASC-Operation (App-ID 6797008248).

**Interfaces:**
- Consumes: neuesten Build (aktuell Build 20).

- [ ] **Step 1: Build-ID des neuesten Builds holen**

```bash
asc builds list --app 6797008248 | python3 -c "
import json,sys
b=json.load(sys.stdin)['data'][0]
print(b['id'], b['attributes']['version'])"
```

- [ ] **Step 2: Note setzen (bestehende Notes ergänzen, nicht überschreiben)**

Zuerst prüfen, ob der Build schon eine de-DE-Note hat (`asc builds test-notes …`-Hilfe konsultieren, Anzeige der vorhandenen Localizations). Text anhängen bzw. neu anlegen:

```bash
asc builds test-notes create --build-id "BUILD_ID" --locale "de-DE" --whats-new \
"Hinweis zum Premium-Kauf in der Beta: TestFlight fragt beim Kauf nach dem Apple-Account-Passwort — das ist eine TestFlight-Eigenheit und kostet nichts. In der App-Store-Version läuft der Kauf über Face ID/Touch ID."
```
Existiert schon eine Note, stattdessen `test-notes update --localization-id …` mit bestehendem Text + angehängtem Hinweis.

- [ ] **Step 3: Verifizieren** — Notes erneut auslesen und prüfen, dass der Hinweis (und ggf. der Alt-Text) vollständig drinsteht.
