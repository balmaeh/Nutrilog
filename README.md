# NutriLog

Persönliche Kalorie- und Makro-Tracking App, optimiert für iPhone und Desktop-Browser.

**Live:** https://balmaeh.github.io/nutrilog/

---

## Für Anwender

### Was kann die App?

- **Tägliches Erfassen** von Mahlzeiten mit Kalorien, Fett, Kohlenhydraten und Protein
- **Barcode-Scanner** per Kamera – Nährwerte werden automatisch via Open Food Facts geladen
- **Lebensmittel-Datenbank** mit ~60 vordefinierten Schweizer Produkten (Coop/Migros), beliebig erweiterbar
- **Gewicht** täglich eintragen, Verlauf im Diagramm sichtbar
- **Verlauf-Diagramm** mit wählbarem Zeitraum und Ansicht (Tageswerte, Wochendurchschnitt, Monatsdurchschnitt)
- **Ziele** für Kalorien, TDEE und Protein individuell einstellbar
- **Export** als JSON (Vollbackup), CSV (Tagessummen) und PDF (Diagramm + Tabelle)
- **Import** eines früheren JSON-Exports zum Wiederherstellen oder Zusammenführen

### Erste Schritte

1. App im Browser öffnen (Link oben)
2. Im Tab **Erfassen** erscheint automatisch das heutige Datum
3. Auf **+** tippen → Mahlzeit aus Datenbank wählen oder Barcode scannen
4. Menge bestätigen → fertig

### Mahlzeiten erfassen

**Aus der Datenbank:**  
Tab *Aus Datenbank* → Produkt suchen → Menge eingeben → Hinzufügen

**Per Barcode:**  
Tab *Barcode / Manuell* → Kamera scannen oder EAN-Code eintippen + Enter → Nährwerte werden geladen → Menge und Einheit prüfen → Hinzufügen

**Manuell:**  
Tab *Barcode / Manuell* → Felder ausfüllen → *Einmalig hinzufügen* (nur für heute) oder *In Datenbank speichern* (dauerhaft)

### Vergangene Tage bearbeiten

Im Tab **Erfassen** oben das Datum ändern. Über den **Verlauf**-Tab → Bleistift-Symbol pro Zeile direkt zu diesem Tag springen.

### Gewicht eintragen

- Im Tab **Erfassen** direkt im Gewichtsfeld
- Im Tab **Verlauf** → Gewichtswert in der Tabelle anklicken → Wert tippen → Enter

### Ziele anpassen

Tab **Einstellungen** → Kalorienziel, TDEE und Proteinziel. Der Button *≈Xg* berechnet einen Vorschlag für das Proteinziel (25% der Kalorien als Protein).

### Datensicherung

**Einstellungen → Daten:**

| Funktion | Beschreibung |
|---|---|
| Export (JSON) | Vollständiges Backup aller Daten |
| Import (JSON) | Backup wiederherstellen oder zusammenführen |
| CSV | Tagessummen (Datum, kcal, Fett, KH, Protein) |
| PDF | Diagramm + Tabelle des gewählten Zeitraums |

> Tipp: Vor dem Gerätewechsel JSON exportieren und auf dem neuen Gerät importieren.

### Datenschutz

Alle Daten bleiben ausschliesslich lokal im Browser (`localStorage`). Es werden keine Daten an einen eigenen Server übertragen. Die Barcode-Suche fragt öffentlich zugängliche Daten bei [Open Food Facts](https://world.openfoodfacts.org/) ab.

---

## Für Entwickler

### Architektur

Single-File HTML-App – kein Framework, kein Build-Schritt, keine Abhängigkeiten ausser CDN-Bibliotheken.

```
index.html          ← die einzige Datei (HTML + CSS + JS)
```

### Externe Bibliotheken (CDN)

| Bibliothek | Version | Zweck |
|---|---|---|
| Chart.js | 4.4.0 | Verlauf-Balkendiagramm |
| ZXing-js | 0.19.1 | Barcode-Kamera-Scanner (Fallback) |
| jsPDF | 2.5.1 | PDF-Export |
| Google Fonts | – | Outfit + DM Mono |

Primär-Scanner: native `BarcodeDetector`-API (Chrome/Android). Fallback: ZXing `BrowserMultiFormatReader`.

### Datenpersistenz

Alle Daten werden in `localStorage` unter dem Key `nutrilog_v1` als JSON gespeichert.

```json
{
  "foods": [{ "id", "kat", "name", "quelle", "einheit", "gew", "k", "f", "c", "p" }],
  "days": {
    "2025-05-19": {
      "items": [{ "name", "menge", "einheit", "kcal", "fett", "kh", "protein" }],
      "ziel": 2000,
      "gewicht": 74.5
    }
  },
  "history": { "2025-02-19": { "kcal", "fett", "kh", "protein" } },
  "settings": { "ziel": 2000, "bedarf": 2135, "zielProtein": 134 },
  "customKats": []
}
```

**Felder in `foods`:**
- `gew` = Gramm pro Einheit; bei `gew: 1` entspricht die Einheit direkt Gramm
- `k/f/c/p` = kcal / Fett / Kohlenhydrate / Protein pro 100g
- Nährwerte werden immer dynamisch berechnet: `(gew × menge / 100) × nährwert_pro_100g`

**`days[datum].ziel`:** Das zum Zeitpunkt des Eintrags gültige Kalorienziel wird pro Tag gespeichert, damit historische Auswertungen korrekt bleiben.

**`history`:** Aggregierte Tagessummen aus einem früheren Excel-Export (wird nur einmalig als Seed geladen).

### Globaler State

```js
let S = {
  foods: [],
  days: {},
  history: {},
  settings: { ziel: 2000, bedarf: 2135, zielProtein: 134 },
  activeTab: 'erfassen',
  today: 'YYYY-MM-DD',
};
let activeDay = todayISO();       // aktiver Tag im Erfassen-Tab
let verlaufFrom = '';             // Von-Datum Verlauf (leer = -60 Tage)
let verlaufTo = '';               // Bis-Datum Verlauf (leer = heute)
let verlaufAgg = 'day';          // 'day' | 'week' | 'month'
```

### Wichtige Funktionen

| Funktion | Beschreibung |
|---|---|
| `load()` | State aus localStorage laden, Seed-Daten befüllen, Migrationen |
| `save()` | State in localStorage schreiben |
| `render()` | Aktiven Tab neu rendern |
| `switchTab(t)` | Tab wechseln: `'erfassen'`, `'verlauf'`, `'db'`, `'cfg'` |
| `calcNutr(food, menge)` | Nährwerte für food × menge berechnen |
| `getTotals(date)` | Tagessumme aus `days[date].items` oder `history[date]` |
| `getAllDays()` | Merge von history + days (nur Tage mit Items) |
| `getAllWeights()` | Alle Gewichtseinträge aus `days` |
| `drawChart()` | Chart.js-Diagramm rendern (Kcal-Balken + Gewichts-Linie) |
| `lookupBarcode()` | Open Food Facts API abfragen |
| `addToDay(entry)` | Eintrag zum aktiven Tag hinzufügen, Kalorienziel speichern |
| `saveWeight(val)` | Gewicht für aktiven Tag speichern |
| `editDayWeight(ev, d)` | Inline-Gewichtseingabe im Verlauf |
| `importData()` | JSON-Backup einlesen und mergen |
| `exportData()` | JSON-Export |
| `exportCSV()` | CSV-Export der Tagessummen |
| `exportPDF()` | PDF mit Diagramm und Datentabelle |

### Layout (iOS-optimiert)

```
#app  (position:fixed; flex-column)
  ├── #hdr    (64px; Header mit Datum + Brand)
  ├── #main   (flex:1; overflow-y:scroll)   ← kein position:fixed → iOS-Scroll-Bug vermieden
  └── #tabs   (72px; Tab-Navigation)
#fab          (position:fixed; ausserhalb #app)
.ov + .sheet  (position:fixed; z-index 200/201; Bottom-Sheets für Modals)
```

### Deployment

```
Repository:   balmaeh/nutrilog
Branch:       main
Datei:        index.html (Root)
URL:          https://balmaeh.github.io/nutrilog/
```

Push auf `main` → GitHub Pages baut automatisch (kein Build-Schritt).

### Bekannte Eigenheiten

- **iOS Safari `<select>`:** braucht `-webkit-appearance: auto` für nativen Picker
- **Input-Zoom iOS:** alle `<input>` haben `font-size: 16px` (verhindert Auto-Zoom)
- **Chart.js offline:** fehlt wenn kein Internet, Rest der App funktioniert
- **Open Food Facts:** braucht Internet; bei Fehler → manuelles Formular als Fallback
- **localStorage Limit:** ~5 MB; bei sehr langem Tracking ggf. History-Daten ausdünnen
- **Barcode-Scanner:** `BarcodeDetector` (nativ, Chrome/Android) → Fallback ZXing; Windows-Webcam oft unzuverlässig
