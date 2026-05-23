# NutriLog – Projektkontext für Claude Code

## Was ist das?
Eine **Single-File HTML-App** zum persönlichen Kalorie- und Makro-Tracking.
Gebaut für den Eigenbedarf (Zürich), deployed via GitHub Pages.
Keine Backend, kein Framework, kein Build-Schritt – alles in einer `index.html`.

**Live-URL:** `https://balmaeh.github.io/nutrilog/` _(oder jeweiliger Repo-Name)_
**Datei:** `index.html` im Root des Repos

---

## Tech-Stack
- Vanilla HTML/CSS/JS (kein Framework)
- **Chart.js 4.4.0** via cdnjs (Verlauf-Diagramm)
- **Google Fonts:** Outfit + DM Mono
- **localStorage** für alle Daten (`nutrilog_v1` als Key)
- **Open Food Facts API** für Barcode-Lookup (CORS-freie REST-API)
  - Endpoint: `https://world.openfoodfacts.org/api/v0/product/{EAN}.json`

---

## Datenstruktur (localStorage: `nutrilog_v1`)
```json
{
  "foods": [
    {
      "id": "f01",
      "kat": "Protein",
      "name": "Produktname",
      "quelle": "Coop",
      "einheit": "Stück",
      "gew": 200,
      "k": 74,
      "f": 1.5,
      "c": 4.9,
      "p": 10
    }
  ],
  "days": {
    "2025-05-19": {
      "items": [
        {
          "name": "Produktname",
          "menge": 1,
          "einheit": "Stück",
          "kcal": 148,
          "fett": 3,
          "kh": 9.8,
          "protein": 20
        }
      ]
    }
  },
  "history": {
    "2025-02-19": { "kcal": 1850, "fett": 82, "kh": 145, "protein": 128 }
  },
  "settings": {
    "ziel": 2000,
    "bedarf": 2135
  },
  "customKats": ["MeineKategorie"],
  "historySeeded": false
}
```

**Felder in `foods`:**
- `gew` = Gramm pro Einheit (z.B. 200 für einen 200g-Becher); bei `gew:1` ist die Einheit direkt "Gramm" → Menge = Grammanzahl
- `k/f/c/p` = kcal/Fett/Kohlenhydrate/Protein pro **100g**
- Nährwerte pro Einheit werden immer dynamisch berechnet: `(gew * menge / 100) * nährwert100`

**`history`** enthält pre-seeded Tagessummen aus einem früheren Excel-Export (Feb–Sep 2025).
Detaillierte Tagesprotokolle laufen über `days`.

---

## App-Struktur (JavaScript)

### Globaler State
```js
let S = {
  foods: [],      // Lebensmittel-Datenbank
  days: {},       // Tagesprotokoll { "YYYY-MM-DD": { items: [...] } }
  history: {},    // Aggregierte Historien-Daten
  settings: { ziel: 2000, bedarf: 2135 },
  customKats: [], // Benutzerdefinierte Kategorien (ohne Produkte)
  today: "YYYY-MM-DD"
};
```

### Wichtige Funktionen
| Funktion | Zweck |
|---|---|
| `load()` | State aus localStorage laden + History-Seed einmalig befüllen |
| `save()` | State in localStorage schreiben |
| `render()` | Aktiven Tab rendern (ruft renderHeute/Verlauf/DB/Cfg auf) |
| `switchTab(t)` | Tab wechseln: `'heute'`, `'verlauf'`, `'db'`, `'cfg'` |
| `openAdd()` | Hinzufügen-Sheet öffnen (aus Datenbank oder Barcode/Manuell) |
| `openEdit(food)` | Lebensmittel bearbeiten oder `null` für neues |
| `calcNutr(food, menge)` | Nährwerte für food × menge berechnen |
| `getTotals(date)` | Tagessumme: aus `days[date].items` oder `history[date]` |
| `getAllDays()` | Merged history + live days für Diagramm |
| `lookupBarcode()` | Open Food Facts API abfragen |
| `getCategories()` | Alle vorhandenen Kategorien aus foods (sortiert) |
| `renameKat(old)` | Kategorie umbenennen (alle betroffenen foods aktualisieren) |
| `deleteKat(name)` | Kategorie löschen → foods auf "Sonstiges" setzen |

### Layout (iOS-optimiert)
```
#app (position:fixed; display:flex; flex-direction:column)
  ├── #hdr       (flex-shrink:0; Höhe: --hdh: 64px)
  ├── #main      (flex:1; overflow-y:scroll)  ← kein position:fixed → iOS-Scroll-Bug vermieden
  └── #tabs      (flex-shrink:0; Höhe: --tbh: 72px)

#fab             (position:fixed; außerhalb von #app)
.ov + .sheet     (position:fixed; z-index:200/201; Bottom-Sheets für Modals)
```

**Wichtig:** `#main` ist KEIN `position:fixed`. Das war der ursprüngliche iOS-Safari-Bug
(Klick-Koordinaten stimmen nach dem Scrollen in `position:fixed`-Scroll-Containern nicht).
Die aktuelle Flexbox-Lösung löst das Problem.

---

## Tabs
| Tab-Key | Beschreibung |
|---|---|
| `heute` | Tagesübersicht, kcal-Hero, Makros, Log-Liste |
| `verlauf` | Balkendiagramm (Chart.js), Monats-Stats, 14-Tage-Tabelle |
| `db` | Lebensmittel-Datenbank, Kategorie-Filter, Suche |
| `cfg` | Ziele, Kategorie-Manager, Daten-Export |

---

## Seed-Daten
Die App enthält ~59 Schweizer Produkte (Coop/Migros) aus einem früheren Excel-Export
als `DEFAULT_FOODS` Array, sowie ~185 Tage historische Daten als `HISTORY_SEED`
(Excel-Seriennummern → ISO-Datum via `excelToISO(serial)`).

Diese werden beim ersten Start in den State geladen, danach aus localStorage bedient.

---

## Kategorien
Kategorien sind **implizit** – sie existieren nur durch die `kat`-Felder der Produkte.
Ausnahme: `S.customKats[]` speichert Kategorien ohne Produkte (für Dropdown-Vorschläge).

Bestehende Kategorien in den Seed-Daten:
`Backwaren`, `Eier`, `Fertiggericht`, `Fleisch/Ersatz`, `Gemüse`,
`Milchprodukte`, `Obst`, `Protein`, `Saucen`, `Süsses`

---

## Bekannte Eigenheiten / Gotchas
- **Menge bei Gramm-Produkten:** Produkte mit `gew:1` (z.B. Emmentaler) → Menge = Gramm direkt.
  Der Step im Menge-Input ist `10` für Gramm-Produkte, `1` für Einheiten-Produkte.
- **Chart.js** wird von cdnjs geladen. Offline → Chart fehlt, Rest der App funktioniert.
- **Open Food Facts** braucht Internet. Bei Fehler → manuelles Formular als Fallback.
- **iOS Safari:** `<select>` braucht `-webkit-appearance: auto` um den nativen Picker anzuzeigen.
- **Input-Zoom iOS:** Alle `<input>` haben `font-size: 16px` – verhindert automatisches Zoom.
- **localStorage** Limit: ~5MB. Bei sehr langem Tracking ggf. History-Daten ausdünnen.

---

## Typische Weiterentwicklungen (Ideen)
- [ ] Mahlzeiten-Vorlagen (häufige Kombinationen speichern)
- [ ] Wöchentliche Zusammenfassung / Durchschnitte im Verlauf-Tab
- [ ] Foto-Upload für Produkte
- [ ] PWA / Service Worker für echten Offline-Betrieb + "Zum Home-Bildschirm"
- [ ] Import von JSON-Export (aktuell nur Export implementiert)
- [ ] Makro-Ziele individuell einstellbar (nicht nur kcal)
- [ ] Dunkelmodus

---

## Deployment (GitHub Pages)
```
Repository: balmaeh/nutrilog (oder jeweiliger Name)
Branch: main
Datei: index.html (Root)
URL: https://balmaeh.github.io/nutrilog/
```

Keine Build-Pipeline nötig – direkt die `index.html` committen, GitHub Pages baut automatisch.
