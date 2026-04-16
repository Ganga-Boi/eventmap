# EventMap — Komplet projekt

## Mappestruktur

```
eventmap_complete/
│
├── pipeline/                   ← Kører lokalt / GitHub Actions
│   ├── schema.py               RØRER IKKE — kontrakten
│   ├── base_scraper.py         Interface alle scrapers arver
│   ├── sheets_adapter.py       Al Google Sheets-logik
│   ├── pipeline.py             Kører alt, merger, skriver output
│   ├── ida_scraper.py          IDA via Cludo REST API
│   ├── dtu_library_scraper.py  DTU Bibliotek via iCal + HTML
│   └── new_source_template.py  Kopiér for ny kilde
│
├── ui/                         ← Deployes på Vercel
│   ├── public/
│   │   ├── index.html          Frontend — én fil, ingen build
│   │   └── events_api.json     Mockdata til lokal udvikling
│   ├── api/
│   │   └── events.js           Serverless function med cache (fase 2)
│   └── vercel.json             Routing config
│
└── .github/
    └── workflows/
        └── pipeline.yml        Automatisk scraping hver 6. time
```

---

## Hvad du gør nu — i rækkefølge

### Dag 1: Find IDA's Cludo IDs (15 min)

```bash
cd pipeline
pip install requests icalendar
python ida_scraper.py
```

Det gemmer `ida_source.html`. Åbn filen i din browser (eller teksteditor),
søg efter `customerId`. Du finder noget som:

```js
cludoSettings = { customerId: 4545589, engineId: 7578030, ... }
```

Indsæt tallene øverst i `ida_scraper.py`:

```python
CUSTOMER_ID = 4545589   # ← dit tal
ENGINE_ID   = 7578030   # ← dit tal
```

Kør igen:

```bash
python ida_scraper.py
```

Det gemmer `ida_cludo_response.json`. Åbn den og se hvilke feltnavne
Cludo bruger (fx `StartDate`, `Location`, `Organizer`).
Tilpas `_parse()` i bunden af `ida_scraper.py` til de faktiske feltnavne.

---

### Dag 1: Sæt Google Sheets op (20 min)

1. Gå til [console.cloud.google.com](https://console.cloud.google.com/)
2. Opret nyt projekt → aktivér **Google Sheets API** + **Google Drive API**
3. IAM → Service Accounts → Opret → Download JSON → gem som `pipeline/credentials.json`
4. Opret et tomt Google Sheet
5. Del sheetet med service account emailen (fra JSON: `client_email`)
6. Kopiér Sheet ID fra URL'en (den lange streng mellem `/d/` og `/edit`)
7. Indsæt i `sheets_adapter.py`:

```python
SHEET_ID = "DIN_SHEET_ID_HER"
```

Test:

```bash
python pipeline.py
```

Første kørsel opretter fanerne **Events**, **Manuel input**, **Kilder** automatisk.

---

### Dag 1: Tilføj manuelle events (5 min)

Åbn Google Sheet → fanen **Manuel input**.
Udfyld events fra Nakkeosten og Fredagsbar manuelt.
Kør `python pipeline.py` igen — de dukker op i Events-fanen.

---

### Dag 2: Deploy UI (10 min)

Test lokalt:

```bash
cd ui
npx serve public
# → åbn http://localhost:3000
```

Deploy til Vercel:

```bash
git init && git add . && git commit -m "EventMap init"
gh repo create eventmap-ui --public --push
# → vercel.com/new → importér repo → Deploy
```

UI'en læser fra `events_api.json` (mockdata) indtil du kopierer den rigtige fil:

```bash
# Efter pipeline har kørt:
cp pipeline/events_api.json ui/public/events_api.json
git add . && git commit -m "events: update" && git push
```

---

### Dag 3+: Automatisér med GitHub Actions

Sæt disse secrets i GitHub → Settings → Secrets → Actions:

| Secret | Værdi |
|--------|-------|
| `EVENTMAP_SHEET_ID` | Dit Google Sheet ID |
| `EVENTMAP_CREDENTIALS_JSON` | Hele credentials.json som tekst |
| `VERCEL_DEPLOY_HOOK` | Fra Vercel → Settings → Git → Deploy Hooks |

Herefter kører pipeline automatisk kl. 08, 14, 20, 02 dansk tid.
GitHub sender mail hvis den fejler.

---

## Tilføj ny kilde (30–60 min)

```bash
cp pipeline/new_source_template.py pipeline/nakkeosten_scraper.py
```

1. Implementér `_fetch()` og `_parse()`
2. Test: `python nakkeosten_scraper.py`
3. Tilføj i `pipeline.py`: `from nakkeosten_scraper import NakkeostenScraper`
4. Tilføj i `SCRAPERS`-listen
5. Tilføj i Google Sheet → Kilder-fanen: `Nakkeosten | TRUE`

---

## Fase 2: Skift til live API (når klar)

I stedet for at committe `events_api.json` til git:

1. I `ui/api/events.js`: uncomment Fase 2-blokken, slet Fase 1
2. I `ui/public/index.html` linje 1: skift til `const API_URL = '/api/events'`
3. I Vercel dashboard → Environment Variables:
   - `GOOGLE_SHEET_ID`
   - `GOOGLE_SERVICE_ACCOUNT` (credentials JSON som string)

---

## Åbne problemer (gør ikke nu)

- **Vercel in-memory cache er ikke garanteret** — opgradér til Vercel KV hvis du
  oplever mange cache misses under load
- **IDA's Cludo filternavne er ukendte** — afklares i `ida_cludo_response.json`
  første gang du kører scraperen
- **Dato-validering i Manuel input** — ingen validering endnu; sørg for at
  skrive datoer som `2025-04-25`, ikke `25/4` eller `25. april`
- **"Sidst opdateret"-signal i UI** — mangler; brugeren ved ikke om data er gammelt
