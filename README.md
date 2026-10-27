# Drug Interaction Knowledge Graph

A FastAPI backend + interactive vis.js frontend for checking drug-drug and drug-food interactions across a knowledge graph built from the [DDInter](https://ddinter.scbdd.com/) database.

**1,947 nodes · 130,445 interactions · fully searchable in the browser**

---

## What it does

Doctors, pharmacists, and researchers deal with polypharmacy every day — patients on 5, 8, even 12 medications at once. Manually checking every pair of drugs for interactions is error-prone and slow.

This project builds a **knowledge graph** of known drug interactions, exposes it via a REST API, and lets you explore it visually in the browser. You send in a list of drugs (and optionally foods), and it tells you:

- Which pairs interact and how severely
- The mechanism behind each interaction
- Safer alternatives where available
- Whether a drug name was recognized, even if typed messily (e.g. `"Ecosprin 75"` resolves to `aspirin`)

---

## Getting started

**Requirements:** Python 3.9+

```bash
# 1. Clone the repo
git clone https://github.com/Shiva9801/KNOWLEDGE_GRAPH.git
cd KNOWLEDGE_GRAPH

# 2. Create and activate a virtual environment
python -m venv venv
venv\Scripts\activate        # Windows
# source venv/bin/activate   # Mac/Linux

# 3. Install dependencies
pip install -r requirements.txt

# 4. Drop in your seed data
# Place seed_graph_ddinter.json in the data/ folder
# (too large for GitHub — see note below)

# 5. Start the server
python app.py
```

Then open **http://localhost:8000** in your browser.

> **Note on the data file:** `data/seed_graph_ddinter.json` is ~125 MB so it's excluded from this repo. You can regenerate it by running `import_ddinter.py` followed by `enrich_ddinter.py`, or download the raw CSVs from the `ddinter/` folder.

---

## API endpoints

| Method | Endpoint | What it does |
|--------|----------|-------------|
| `GET`  | `/`      | Serves the interactive graph UI |
| `GET`  | `/health` | Quick check — returns node/edge counts |
| `GET`  | `/drugs` | Lists all known drugs and foods in the graph |
| `GET`  | `/graph` | Raw nodes + edges (for custom visualizations) |
| `POST` | `/check` | The main endpoint — check interactions for a patient's drug list |

### Example: checking interactions

```bash
curl -X POST http://localhost:8000/check \
  -H "Content-Type: application/json" \
  -d '{
    "patient_id": "patient-001",
    "drugs": [
      {"name": "warfarin"},
      {"name": "ibuprofen"},
      {"name": "aspirin"}
    ],
    "foods": ["leafy greens"]
  }'
```

**Response includes:**
- `interactions` — sorted by severity (contraindicated → major → moderate → minor)
- `alternatives` — safer drug swaps, cross-checked against the rest of the patient's meds
- `unresolved_drugs` — anything the graph didn't recognize
- `low_confidence_drugs` — drugs matched but with a confidence score below 0.7
- `disclaimer` — always there, because this is decision support, not a prescription

---

## Drug name resolution

The `/check` endpoint doesn't require exact generic names. It tries to match what you give it in three ways:

1. **Exact match** — `warfarin` → warfarin
2. **First word** — `warfarin 5mg` → warfarin
3. **Substring fallback** — `Ecosprin 75` → aspirin (via alias lookup)

This makes it practical for downstream systems that feed in OCR'd prescription text.

---

## The graph UI

Served at `/`, the frontend pulls data from `/graph` and renders it with [vis.js Network](https://visjs.github.io/vis-network/docs/network/).

- **Search** any drug or food to center the graph on it
- **Filter** by severity level (contraindicated / major / moderate / minor)
- **Click** any node to see its interactions in the side panel
- Edge colors: black = contraindicated, red = major, amber = moderate, yellow = minor
- Dashed edges = drug-food interactions

---

## Project structure

```
KNOWLEDGE_GRAPH/
├── app.py                              # FastAPI app — all endpoints live here
├── graph.html                          # Browser UI (vis.js)
├── requirements.txt
│
├── data/
│   ├── seed_graph_ddinter.json         # Main graph data (not in repo, ~125 MB)
│   ├── seed_graph.json                 # Smaller curated seed
│   ├── seed_graph_backup.json
│   └── mock_prescriptions.json        # Test prescriptions
│
├── ddinter/                            # Raw DDInter CSVs + knowledge base
│   ├── ddi_knowledge_base.json
│   └── ddinter_downloads_code_*.csv
│
├── import_ddinter.py                   # Parse DDInter CSVs → seed JSON
├── enrich_ddinter.py                   # Add mechanisms, alternatives, sources
├── import_ddinter_certolizumab_updated.py
├── inspect_ddinter.py                  # Quick graph inspection script
├── remove_three.py                     # Utility: remove specific nodes
└── test_check.py                       # Backend tests (13 test cases)
```

---

## Running the tests

```bash
venv\Scripts\activate
python test_check.py
```

Covers: health check, drug/food listing, major interactions, drug-food interactions, polypharmacy, brand-name resolution, OCR-style fuzzy matching, unknown drugs, low-confidence flags, alternatives, severity ordering, and the graph endpoint.

---

## Data source

Interaction data comes from **DDInter** — a dedicated drug-drug interaction database curated from literature and clinical sources. The raw downloads (by ATC code) are in the `ddinter/` folder.

Curated interactions (warfarin + others, aspirin, amiodarone, etc.) have `details_status: "curated"` and `mechanism_verified: true` in the graph.

---

## Disclaimer

This is a **decision support tool** — not a replacement for a doctor or pharmacist. Always verify interactions clinically before acting on them.

---

## License

MIT
