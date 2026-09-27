# BhuMilan — Land Record Harmonization for India

**SIH 2026 · Problem Statement SIH26013**

BhuMilan links the land records that different departments keep about the same parcel:
- the cadastral map
- the Revenue record of rights
- the Municipal property-tax record
- drone orthoimagery (ORI)
- the DSM/DTM height model
- GNSS surveys
- utility lines

It checks these records against each other and scores each parcel. Clean parcels are approved automatically. Only parcels with a real problem go to an officer, and each one comes with evidence and a recommended fix. Every upload, decision and certification is written to a tamper-evident ledger.

The website works for any State or UT. A GIS officer adds a city, ULB or tehsil and uploads that location's files. BhuMilan then:
- detects the coordinate system (every UTM zone from 42N to 47N)
- maps English, Hindi or Telugu column headers to a standard schema
- converts local area units, including the state-specific bigha

---

## Run it

**Windows:** double-click `start_server.bat`. It checks Python, installs the packages and starts the server.

**Any OS:**
```bash
pip install -r requirements.txt
python run_bhumilan.py
```
Then open **http://localhost:8000**.

Python 3.10 or later is needed. No database, GPU or internet connection is needed. The background map tiles use the internet when it's available; without it, the parcels still draw.

### Sign-in

The website opens on an **officer sign-in page**, and nothing else loads until someone signs in. Every data route on the server (parcels, images, exports, OGC API, ledger, models) refuses requests without a valid session. Each officer only sees the locations in their own jurisdiction.

### Demo accounts (password `demo123`)
| Username | Role | Can do | Jurisdiction |
|---|---|---|---|
| `surveyor` | Field Surveyor | view | all |
| `gis.officer` | GIS Data Officer | view, upload data, add locations | all |
| `ri.vsk` | Revenue Inspector | view, decide | Visakhapatnam only |
| `ri.jaipur` | Revenue Inspector | view, decide | Jaipur only |
| `tahsildar.vsk` | Tahsildar | view, decide, certify | Visakhapatnam only |
| `admin` | National Admin | everything | all |

Passwords are stored as PBKDF2 hashes. Sessions last 8 hours. After 5 failed logins in 2 minutes, the account is locked out briefly.

---

## 5-minute demo script

1. **Sign in** as `admin`. **India overview:** 5 pilot cities in 5 states, 72 parcels, and the problems found.
2. Sign out and sign in as `ri.vsk`: only Visakhapatnam is visible. **Map console → AP-VSK-014.** The cadastral map is 8.1 m off from the wall seen in the drone image and from the GNSS corners. Click **Apply the recommended actions**. The boundary is corrected from GNSS, the score goes up, and the change is written to the ledger.
3. **AP-VSK-023:** the Municipal record is still in the late father's name, while Revenue shows the heir in Telugu script. The owner-name AI reads the Telugu, matches it and recommends mutation.
4. **AP-VSK-019 → Evidence → Change 2019→2025:** new construction that isn't in the municipal record, detected by comparing roofs across survey years. The **Heights** tab shows extra floors taken from the DSM − DTM difference.
5. **Review queue:** only the parcels that need an officer are listed.
6. **Upload data:** log in as `gis.officer` and upload any CSV. The column mapping and area unit are suggested automatically.
7. **Audit ledger:** log in as `admin` and click **Verify chain** (it passes). Then click **Simulate tampering** and verify again. The edited entry is caught. Click **Undo tampering** to restore it.
8. **AI models:** see how accurate each model is, and type two names in any script to test the matcher.

---

## What happens to each parcel

```
Upload (GeoJSON / KML / CSV / zipped Shapefile, any UTM zone, any header language, any unit)
   │  ingest.py – CRS detection, column mapping, unit conversion
   ▼
Link records     cadastral ⇄ revenue (survey no.) ⇄ municipal (point-in-polygon, ≤6 m)
   │             ⇄ GNSS (nearest corner, ≤10 m) ⇄ utilities (buffer)
   ▼
Checks           geometry: shift vs wall / GNSS, overlap, self-intersection, utility clearance
   │             owners: name-matching model across 9 Indian scripts, inheritance detection
   │             buildings: roof extraction, 2019→2025 change, floors from nDSM
   ▼
Score 0–100      auto-approve ≥ 85 with no safety issue; otherwise → review queue
   ▼
Officer          apply recommendation · accept record · order GNSS survey · field enquiry · undo
   ▼
Tahsildar        batch-certify verified parcels
   ▼
Ledger           SHA-256 hash chain · GeoJSON / CSV export · OGC API – Features
```

**Which source wins when records disagree:**
- Boundary: GNSS › drone › cadastral
- Ownership: Revenue › Municipal
- Buildings: DSM › Municipal

---

## Measured results (synthetic pilot data, 72 parcels)

| Model | Result |
|---|---|
| Boundary walls from ORI | 72/72 found; 68/72 IoU ≥ 0.9; mean corner error 0.41 m |
| Buildings | 69/77 found, 0 false detections; median built-up area error 3% |
| New construction (2019→2025) | 5/5 found, 0 false alarms |
| Floors from DSM − DTM | 69/72 correct |
| Owner-name matcher (logistic regression, 11 features) | 98.1% accuracy vs 76.6% for a plain string-similarity rule |
| Owner names across scripts (Telugu/Hindi/Bengali/… vs English) | 94.3% vs 40% for the plain rule |
| Planted problems caught end to end | 25/25 |

To reproduce: `python tools/evaluate_vision.py` and `python tools/train_name_matcher.py`.

Officers can mark two names as "same person" or "different people" in the owner panel. Those labels are saved and weighted 5× when an admin retrains the model from the AI models page.

---

## Honest limits

- **Synthetic pilot data.** The imagery, height models and owners are generated by `tools/generate_pilot_data.py`, and all names are fictional. The numbers above are measured against the generator's ground truth, not against real ORI from SVAMITVA. Real imagery will need re-tuning, or a trained segmentation model.
- **Vision uses classical OpenCV** (morphology and colour thresholds), not a deep network. It is fast, explainable and needs no GPU, but it is sensitive to roof materials and trees it hasn't seen.
- **Storage is JSON files** on disk, which is fine for a single server and pilot scale. For state scale, move to PostgreSQL/PostGIS; the data layout below maps one-to-one onto tables.
- **Demo accounts** are built in. A real deployment would use the state's SSO or Parichay.
- No real state portals are scraped. The data comes only from officer uploads, in line with DPDP Act principles.

---

## Project layout

```
server.py                FastAPI app: auth, roles, all API routes, OGC API
run_bhumilan.py          starts uvicorn on :8000
engine/
  geo.py                 WGS84 ⇄ UTM (all Indian zones), CRS detection, polygon maths
  ingest.py              readers (GeoJSON/KML/CSV/Shapefile), column mapping, units
  names.py               9-script transliteration + owner-name matching model
  vision.py              walls, roofs, change detection, nDSM floors, evidence images
  pipeline.py            record linking, checks, scoring, recommendations
  store.py               locations, datasets, decisions, officer labels
  ledger.py              SHA-256 hash-chained audit ledger
static/                  web app (HTML/CSS/JS + Leaflet, no build step)
models/                  name_matcher.json (weights), vision_metrics.json
tools/                   pilot-data generator, model training, vision evaluation
tests/test_api.py        end-to-end tests (roles, decisions, uploads, OGC, ledger)
data/locations/<ID>/     manifest.json + each department's files + ORI + surface model
data/state/              decisions + ledger (created at run time; git-ignored)
```

## API (selected)

| Method | Path | |
|---|---|---|
| GET | `/api/locations` | all locations with summary |
| POST | `/api/locations/{loc}/run` | re-run harmonization |
| GET | `/api/locations/{loc}` | location with every harmonized parcel |
| GET | `/api/locations/{loc}/parcels/{pid}` | full parcel report |
| POST | `/api/locations/{loc}/parcels/{pid}/decide` | officer decision |
| POST | `/api/locations/{loc}/certify` | Tahsildar batch certification |
| POST | `/api/upload/preview` → `/api/upload/commit` | two-step upload |
| GET | `/api/locations/{loc}/export.geojson` / `.csv` | harmonized export |
| GET | `/ogc/collections/{loc}/items?bbox=` | OGC API – Features |
| GET | `/api/audit-log/verify` | verify the hash chain |

## Tests

```bash
python tests/test_api.py
```

The tests:
- check that all 25 planted problems are caught
- enforce role and jurisdiction rules
- apply decisions and check how they cascade to neighbouring parcels
- create a new Telangana location from a UTM-zone-44 Shapefile, a Hindi-header CSV in hectares and a KML
- check the exports, the OGC API, and ledger tamper detection

Your data and decisions are backed up before the tests and restored afterwards. The same tests run in GitHub Actions (`.github/workflows/ci.yml`).
