# Kopfrechnen-Trainer – Projektdokumentation

## Überblick

Offline-fähige PWA (Progressive Web App) zum Kopfrechnen-Training, gehostet auf GitHub Pages. Zielgruppe: Erwachsener (Seb), der Business-Kopfrechnen wieder flüssig machen will – keine Kinder-App.

**Live-URL**: https://seb007755.github.io/kopfrechnen/
**Repo**: https://github.com/seb007755/kopfrechnen

## Dateien

```
kopfrechnen/
├── index.html        ← Die gesamte App (HTML + CSS + JS in einer Datei)
├── sw.js             ← Service Worker für Offline-Caching
├── manifest.json     ← PWA-Manifest (Name, Icons, Display-Modus)
├── icon-192.png      ← App-Icon 192×192
├── icon-512.png      ← App-Icon 512×512
├── .nojekyll         ← Verhindert Jekyll-Processing auf GitHub Pages
└── PROJECT.md        ← Diese Datei
```

## Architektur

**Single-File-App**: Alles (HTML, CSS, JS) lebt in `index.html`. Keine Frameworks, keine Build-Tools, kein npm. Vanilla JS im IIFE-Pattern.

**Offline**: Service Worker (`sw.js`) cached alle Assets beim ersten Laden. Danach funktioniert die App komplett offline (Flugmodus-tauglich). Cache-Version wird über `CACHE_NAME` in `sw.js` gesteuert – bei Updates den Versionsnamen ändern (z.B. `kopfrechnen-v2`), damit der SW den Cache invalidiert.

**Fonts**: System-Fonts (SF Pro auf iOS, Segoe UI auf Windows). Keine externen Abhängigkeiten. Die Artifact-Version (nur für Claude.ai-Chat) nutzt Google Fonts (Fraunces + Nunito), aber die GitHub-Pages-Version (`index.html`) ist komplett self-contained.

**Hosting**: GitHub Pages, Deploy from Branch `main`, Root `/`.

## Aufgabentypen

### Grundrechenarten (Chips oben, mindestens 1 muss aktiv sein)

| Typ-ID    | Label       | Beschreibung                          | Zahlenbereich        |
|-----------|-------------|---------------------------------------|----------------------|
| `mul`     | 1 × 1      | Multiplikation                        | Faktoren 2–9         |
| `div`     | ÷ glatt     | Division ohne Rest                    | Quotient 2–9, Divisor 2–9 |
| `divrest` | ÷ mit Rest  | Division mit Rest                     | Dividend 10–90, Divisor 2–9 |
| `add`     | + bis 100   | Addition                              | Summanden so dass Summe ≤ 100 |
| `sub`     | − bis 100   | Subtraktion                           | Minuend 10–100, Ergebnis ≥ 0 |

### Fokus-Bereiche (optional, 75/25-Split wenn aktiv)

| Typ-ID      | Label             | Beschreibung                                      |
|-------------|-------------------|----------------------------------------------------|
| `mul678`    | 1×1 mit 6/7/8     | Mindestens ein Faktor ist 6, 7 oder 8 (2–9)       |
| `subzehner` | − Zehnerübergang  | b > Einer von a, erzwingt Borgen (z.B. 23−7)      |
| `div3`      | ÷ dreistellig     | Dividend 100–999; 1-stelliger Divisor (2–9, Q≤150) oder 2-stelliger (10–25, Q≤19); gemischt glatt/Rest |
| `reihe`     | Zahlenreihen      | 5er-Sequenz einer Malreihe (×2 bis ×15), Position 1–5 oder 6–10, 3–4 Felder versteckt |

### Business-Typen (optional, 75/25-Split wenn aktiv, ±10% Toleranz)

| Typ-ID      | Label            | Beschreibung                                           |
|-------------|------------------|--------------------------------------------------------|
| `estDiv`    | ÷ Überschlag     | Großer Dividend ÷ 1–2-stelliger Divisor, Schätzung     |
| `estMul`    | × Überschlag     | 2–3-stellig × 2-stellig, Schätzung                     |
| `pctOf`     | % von Betrag     | z.B. "15 % von 2.400.000"                              |
| `pctShare`  | %-Anteil         | z.B. "2.560 von 8.000 = ? %"                           |
| `pctChange` | % Veränderung    | z.B. "300.000 → 351.000 = +? %", mit Vorzeichen        |

## Aufgaben-Mix-Logik

- **Kein Fokus/Business aktiv**: Aufgaben gleichmäßig aus den gewählten Grundrechenarten
- **Fokus und/oder Business aktiv**: 75 % aus den Fokus/Business-Typen (gleichmäßig verteilt), 25 % aus den Grundrechenarten
- **Dedup**: Jede Aufgabe hat einen `key`. Kommutative Operationen normalisieren (6×8 = 8×6 → gleicher Key `mul:6,8`). Mul und Mul678 teilen sich den Key-Space. Max 50 Retries pro Slot, dann wird übersprungen.

## UI-Konzepte

### Screens

1. **Startscreen**: Chip-Auswahl (Grundrechenarten, Fokus, Business) + Aufgabenanzahl (10/25/50)
2. **Quiz-Screen**: Fortschrittsbalken, Aufgabenkarte, Eingabefeld(er), Prüfen-Button
3. **End-Screen**: Score, Zeit (gesamt + Ø pro Aufgabe), Aufschlüsselung nach Typ

### Aufgabenkarte – Varianten

- **Standard** (mul, div, add, sub, mul678, subzehner): Ein Eingabefeld, `=`
- **Mit Rest** (divrest, div3 mit Rest): Zwei Eingabefelder (Ergebnis + Rest), `=`
- **Überschlag** (estDiv, estMul, pctOf): Ein Eingabefeld, `≈`, Toleranz ±10%
- **Prozent** (pctShare, pctChange): Ein Eingabefeld mit %-Suffix, `≈`, Toleranz ±10%
- **Zahlenreihe** (reihe): 5 Zellen in einer Reihe, gegebene als graue Kästchen, fehlende als gelbe Inputs, kein `=`-Zeichen

### Feedback

- **Richtig**: Grün, Pop-Animation, Auto-Advance (650ms normal, 1200ms bei Überschlag)
- **Falsch**: Rot, Shake-Animation, zeigt korrekte Antwort, "Weiter →"-Button
- **Knapp daneben** (nur bei Überschlag, innerhalb ±20%): Gelb/Amber, "≈ Knapp!"
- Bei Überschlag-Aufgaben wird nach "Richtig" auch der exakte Wert angezeigt

### Eingabe-Parsing

Die Funktion `parseInput()` akzeptiert deutsche und englische Zahlenformate:
- `1.234` und `1234` → 1234 (Tausenderpunkt erkannt)
- `1,5` → 1.5 (Dezimalkomma)
- `1.234,5` → 1234.5
- `−` (Unicode) und `-` (ASCII) für negative Zahlen

## Styling

- Warme, ruhige Farbpalette: Background `#fef9f3`, Akzent `#e86a3c`, Ink `#1f2330`
- Business-Typen visuell abgesetzt: Blau (`#2d5e8a`) statt Orange
- Große Tap-Targets, `inputmode="numeric"` für automatische Zahlentastatur
- Responsive: funktioniert ab 380px Viewport-Breite
- Keine externen Abhängigkeiten (Fonts, CSS-Frameworks, etc.)

## Bekannte Design-Entscheidungen

- **×1 und ×10 sind bewusst ausgeschlossen** – zu trivial für das Trainingsziel
- **Toleranz ist fest auf ±10%** – aktuell nicht konfigurierbar in der UI
- **Keine Persistenz** – kein Score-Tracking über Sessions hinweg (kein localStorage, da im Artifact-Kontext nicht verfügbar; auf GitHub Pages wäre es möglich)
- **Keine Timer-Pressure** – die Zeit läuft mit, wird am Ende gezeigt, aber es gibt keinen Countdown. Bewusste Entscheidung: Blockaden kommen durch Druck, also erstmal Fluency ohne Stress aufbauen.

## Mögliche Erweiterungen

- Score-Historie (localStorage auf GitHub Pages)
- Schwierigkeitsgrad-Progression (automatisch schwerer wenn Score hoch)
- Spaced Repetition: falsch beantwortete Aufgaben häufiger wiederholen
- Timer-Modus (optional, für Fortgeschrittene)
- Weitere Typen: Brüche, Dezimalzahlen, Einheiten umrechnen
- Cheat-Sheet / Lernkarten-Modus (z.B. 1/6 = 16,7% Herleitung)
- Dark Mode
