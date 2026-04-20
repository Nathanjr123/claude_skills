# CLAUDE CODE PROMPT — AI Securewatch Entity Resolution Demo

## Context

You are building a **working demo** of AI Securewatch's core fraud detection technology. AI Securewatch is a South African startup that provides **Payment-Based Vendor Discovery** — it starts from payment data and uses entity resolution + graph analysis to reveal hidden relationships between employees and vendors that indicate potential insider-linked vendor (ILV) fraud.

This demo uses **realistic South African dummy data** with intentionally embedded fraud patterns to showcase the full pipeline: data generation → entity resolution → graph building → fraud detection → visualization.

The base repo structure, config, README, data model docs, and SA ID validator are already created. You need to **implement all the modules** described below.

---

## PRIORITY ORDER — Build in this sequence

### Phase 1: Data Generation (START HERE)
### Phase 2: Entity Resolution Pipeline  
### Phase 3: Graph Building & Analysis
### Phase 4: Fraud Pattern Detection
### Phase 5: Streamlit Dashboard
### Phase 6: FastAPI Endpoints
### Phase 7: Integration Scripts & Tests

---

## Phase 1: Data Generation (`src/data_gen/`)

### File: `src/data_gen/sa_dummy_data.py`

Build a comprehensive dummy data generator that creates realistic South African data. Use the constants defined in `src/config.py` (SA names, surnames by language group, cities, departments, etc.).

**Requirements:**
1. Generate **200 employees** with:
   - Valid SA ID numbers (use `sa_id_validator.generate_sa_id()` from `src/data_gen/sa_id_validator.py`)
   - Names drawn from `config.SA_FIRST_NAMES` and `config.SA_SURNAMES` (mix across language groups)
   - Realistic SA addresses in Gauteng, Western Cape, KwaZulu-Natal
   - Department assignments matching `config.DEPARTMENTS` headcount distribution
   - Approval limits tied to seniority (junior R50K, mid R100K, senior R500K, director R1M)
   - SA mobile phone numbers (format: 07X-XXX-XXXX)
   - Company email addresses (format: first.surname@msc-sa.co.za)

2. Generate **150 vendors** with:
   - Company names (mix of realistic SA business names: "[Surname] Trading", "[Name] Logistics", "[Name] Consulting (Pty) Ltd")
   - CIPC-format registration numbers: `YYYY/NNNNNN/NN` (e.g., `2019/458392/07`)
   - `in_procurement_system`: True for ~60%, False for the rest (shadow vendors)
   - Categories: Logistics, IT Services, Consulting, Maintenance, Catering, Stationery, Fleet Management, Security
   - Bank account numbers (10-digit numeric)

3. Generate **300 companies** (CIPC-style registry data):
   - Each with 1-4 directors (name + SA ID)
   - Registration dates spanning 2010-2025
   - Registered addresses
   - Shareholding structures

4. Generate **5,000 transactions** over 12 months:
   - Realistic amount distribution: 60% under R50K, 25% R50K-R200K, 10% R200K-R500K, 5% over R500K
   - Each linked to a vendor and an approving employee
   - Descriptions matching vendor category ("IT support services Q3", "Diesel fuel delivery Oct", etc.)
   - PO numbers for ~70% (the rest are "emergency" or "recurring")

5. Generate **50 bank statement entries** for shadow vendors not in vendor master file

All data should be saved as **Parquet files** in `data/raw/` using Polars:
- `employees.parquet`
- `vendors.parquet`
- `companies.parquet`
- `directors.parquet` (flattened: one row per director-company pair)
- `transactions.parquet`
- `bank_statements.parquet`

### File: `src/data_gen/fraud_patterns.py`

This is the **most critical data generation file**. It modifies the base data to embed specific, detectable fraud patterns. Each pattern creates a "ground truth" that the entity resolution and fraud detection should find.

**Embed these patterns:**

1. **Shell Companies (5 instances):**
   - Create 5 vendor companies all directed by the same person (use slightly different name spellings: "J. Nkosi", "James Nkosi", "J Nkosi")
   - All registered at the same address (e.g., "Suite 12, 45 Commissioner St, Johannesburg")
   - All registered within 2 months of each other
   - Each receives payments from different departments (to avoid single-approver detection)

2. **Self-Supply Loops (3 instances):**
   - Pick 3 employees from Procurement department
   - For each, create a vendor company where that employee is a director — but under a variation of their name:
     - Employee "Thandiwe Dlamini" → Director "T. Dlamini" or maiden name "Thandiwe Ngcobo"
     - Employee "Johan van der Merwe" → Director "J. van der Merwe" 
   - The employee approves purchase orders to their own vendor company
   - Their SA ID number appears in the company's director records (key match for entity resolution)

3. **Split Purchasing (4 instances):**
   - For 4 vendor-approver pairs, create clusters of transactions:
     - 4-6 transactions within a 7-day window
     - Each amount between R80,000 and R99,000 (just below R100K threshold)
     - Same approver on all
     - Combined total exceeds R400K

4. **Circular Payments (2 instances):**
   - Create payment chains: Vendor A receives payment, Vendor B receives similar amount from same department next day, Vendor C receives similar amount day after
   - Amounts within 5% of each other
   - Monthly recurrence over 6 months

5. **Ghost Vendors (3 instances):**
   - 3 vendors that receive regular monthly payments
   - All with generic descriptions ("Professional services", "Consulting fees", "Advisory services")
   - All round numbers (R75,000, R120,000, R95,000)
   - No corresponding PO numbers
   - Not in procurement system

6. **Shared Identifiers (6 instances):**
   - 2 cases: Employee phone number = Vendor contact phone
   - 2 cases: Employee residential address = Vendor registered address
   - 2 cases: Employee bank account = Vendor bank account

7. **Dormancy Spikes (2 instances):**
   - 2 vendors with no transactions for months 1-8, then large transactions in months 9-12
   - Total dormancy-period spend = R0, spike-period spend > R500K

**CRITICAL:** Each fraud pattern must set `is_fraudulent=True` and `fraud_type="pattern_name"` on the relevant entities and transactions. This is the ground truth for evaluating detection accuracy.

Save a `fraud_ground_truth.json` in `data/raw/` listing every fraudulent entity and its pattern type.

### File: `src/data_gen/company_generator.py`

Helper for generating realistic CIPC-style company data:
- Company name generator (combine SA names + industry terms)
- Registration number generator (valid CIPC format)
- Director assignment with SA ID numbers
- Address generator for SA business addresses

---

## Phase 2: Entity Resolution (`src/entity_resolution/`)

### File: `src/entity_resolution/splink_pipeline.py`

This is the **core technical showcase**. Implement the Splink entity resolution pipeline that matches entities across the employee, vendor, and company director datasets.

```python
# Key imports
from splink import DuckDBAPI, Linker, SettingsCreator, block_on
import splink.comparison_library as cl
```

**Implementation:**

1. **Prepare datasets for linking:**
   - Load employees, vendors, and directors from Parquet files
   - Normalize names (lowercase, strip whitespace, handle "van der", "du" prefixes)
   - Normalize phone numbers (strip spaces, dashes, leading zeros → +27)
   - Normalize addresses (standardize "St"/"Street", "Rd"/"Road", etc.)
   - Create a unified schema with columns: `unique_id`, `source_dataset`, `first_name`, `surname`, `sa_id_number`, `phone`, `email`, `address`, `postal_code`, `company_reg`

2. **Configure Splink settings:**
   ```python
   settings = SettingsCreator(
       link_type="link_only",  # We're linking across datasets, not deduping
       blocking_rules_to_generate_predictions=[
           block_on("sa_id_number"),                    # Exact SA ID match
           block_on("company_registration_number"),      # Exact company reg
           "l.surname = r.surname AND l.postal_code = r.postal_code",  # Surname + area
           "l.phone_normalized = r.phone_normalized",    # Same phone
       ],
       comparisons=[
           cl.JaroWinklerAtThresholds("first_name", [0.92, 0.85, 0.7]),
           cl.JaroWinklerAtThresholds("surname", [0.92, 0.85, 0.7]),
           cl.ExactMatch("sa_id_number"),
           cl.LevenshteinAtThresholds("address_normalized", [2, 5]),
           cl.ExactMatch("phone_normalized"),
           cl.ExactMatch("email"),
           cl.ExactMatch("postal_code"),
       ],
       retain_intermediate_calculation_columns=True,
   )
   ```

3. **Train the model:**
   ```python
   linker = Linker([employees_df, vendors_df, directors_df], settings, DuckDBAPI())
   linker.training.estimate_u_using_random_sampling(max_pairs=1e6)
   linker.training.estimate_parameters_using_expectation_maximisation(
       block_on("sa_id_number")
   )
   linker.training.estimate_parameters_using_expectation_maximisation(
       block_on("surname")
   )
   ```

4. **Predict and cluster:**
   ```python
   predictions = linker.inference.predict(threshold_match_probability=0.8)
   clusters = linker.clustering.cluster_pairwise_predictions_at_threshold(
       predictions, threshold_match_probability=0.8
   )
   ```

5. **Export results:**
   - Save predictions (pairwise matches with probabilities) to `data/processed/entity_matches.parquet`
   - Save clusters (entity groups) to `data/processed/entity_clusters.parquet`
   - Generate a match summary showing how many cross-dataset links were found
   - Calculate precision/recall against ground truth fraud patterns

6. **Provide diagnostic outputs:**
   - Splink match weight chart data
   - Comparison viewer data for the most interesting matches
   - Blocking rule hit rates

### File: `src/entity_resolution/blocking_rules.py`

SA-specific blocking rule helpers:
- `normalize_sa_name(name)`: Handle "van der Merwe" → "vandermerwe", strip titles
- `normalize_sa_phone(phone)`: "082-555-1234" → "+27825551234"
- `normalize_sa_address(address)`: "123 Commissioner St, JHB" → standardized components
- `extract_surname_prefix(surname, n=3)`: First 3 chars for blocking
- `extract_postal_zone(postal_code)`: First 2 digits for area grouping

### File: `src/entity_resolution/comparisons.py`

Custom comparison helpers for SA data:
- Company name comparison (handle "Pty Ltd", "(Pty) Ltd", "Proprietary Limited")
- SA ID similarity (same DOB extraction, gender match)
- Address component comparison (separate street/city/province matching)

---

## Phase 3: Graph Building (`src/graph/`)

### File: `src/graph/builder.py`

Build a NetworkX graph from the entity resolution results and raw relationship data.

**Node types:** Employee, Vendor, Company, Address, Phone, BankAccount, Transaction
**Edge types:** All relationship types from `docs/DATA_MODEL.md`

**Implementation:**
1. Load entity resolution results (clusters and matches)
2. Create nodes for each unique entity, with attributes (type, name, risk flags)
3. Create edges for:
   - Employment relationships (employee → employer company)
   - Directorship links (person → company they direct)
   - Ownership links (person/company → company they own shares in)
   - Payment links (company → vendor via transactions)
   - Approval links (employee → transactions they approved)
   - **Entity resolution matches** (resolved same-entity links across datasets)
   - Shared identifier links (same address, phone, bank account)
4. Assign edge weights based on `config.LINK_RISK_WEIGHTS`
5. Save graph as GraphML to `data/graphs/entity_graph.graphml`
6. Export node and edge lists as Parquet for dashboard

### File: `src/graph/algorithms.py`

Graph analysis algorithms:
- **Community detection** (Louvain): Find clusters of potentially colluding entities
- **PageRank**: Surface hub entities with disproportionate connections
- **Betweenness centrality**: Find intermediaries connecting separate networks
- **Shortest path**: Trace indirect employee → vendor relationships through intermediaries
- **Connected components**: Identify isolated fraud networks
- **Cycle detection**: Find circular payment patterns (A→B→C→A)

Each algorithm should return results as a Polars DataFrame with entity IDs, scores, and flags.

### File: `src/graph/visualizer.py`

Interactive graph visualization using pyvis:
- Color nodes by type (Employee=blue, Vendor=red, Company=green, Address=gray)
- Size nodes by PageRank score (more connected = larger)
- Color edges by risk weight (higher risk = redder)
- Highlight fraud pattern nodes with glow/border
- Generate standalone HTML files saved to `data/graphs/`
- Support filtering by: community, risk score threshold, fraud pattern type, specific entity neighborhood

---

## Phase 4: Fraud Detection (`src/fraud_detection/`)

### File: `src/fraud_detection/patterns.py`

Implement rule-based fraud pattern detection:

```python
def detect_shell_companies(companies_df, directors_df) -> pl.DataFrame:
    """Find companies sharing directors or registration addresses."""

def detect_self_supply(employees_df, directors_df, entity_matches_df) -> pl.DataFrame:
    """Find employees who are directors of vendor companies."""

def detect_split_purchasing(transactions_df, threshold=100_000) -> pl.DataFrame:
    """Find transaction clusters below approval thresholds."""

def detect_circular_payments(transactions_df, vendors_df) -> pl.DataFrame:
    """Find A→B→C→A payment chains."""

def detect_ghost_vendors(vendors_df, transactions_df) -> pl.DataFrame:
    """Find vendors with payments but no delivery evidence."""

def detect_shared_identifiers(employees_df, vendors_df) -> pl.DataFrame:
    """Find shared phone/address/bank between employees and vendors."""

def detect_dormancy_spikes(transactions_df, dormancy_months=6) -> pl.DataFrame:
    """Find vendors with long dormancy followed by sudden activity."""

def run_all_detections(data_dir: Path) -> dict[str, pl.DataFrame]:
    """Run all detection algorithms and return results by pattern type."""
```

Each detection function returns a DataFrame with: entity_id, pattern_type, confidence_score, evidence_summary, related_entities.

### File: `src/fraud_detection/risk_scorer.py`

Multi-hop risk propagation:
```python
def calculate_risk_score(graph, entity_id, decay_rate=0.5) -> float:
    """Calculate risk score with exponential decay across hops.
    
    Risk_at_hop_n = base_risk × (decay_rate)^n
    """
```

Also implement:
- Aggregate risk scoring for entities (combine all link types)
- Risk ranking (sorted list of highest-risk entities)
- Risk explanation (human-readable description of why entity is flagged)

### File: `src/fraud_detection/vendor_reality.py`

The **Vendor Reality Snapshot** — the key output:
```python
def generate_vendor_reality_snapshot(
    transactions_df, vendors_df, bank_statements_df, entity_matches_df, fraud_results
) -> dict:
    """Generate the Vendor Reality vs Vendor Record gap analysis.
    
    Returns:
        {
            "total_entities_in_payments": int,
            "total_in_vendor_master": int,
            "total_unknown": int,
            "unknown_percentage": float,
            "total_value_to_unknown": float,
            "classification": {
                "formally_onboarded": {"count": int, "total_value": float},
                "informally_known": {"count": int, "total_value": float},
                "unknown": {"count": int, "total_value": float},
                "expired_contract": {"count": int, "total_value": float},
            },
            "top_pattern_flags": [
                {"pattern": str, "count": int, "description": str},
            ],
            "concentration_risk": {
                "top_10_unmonitored_vendors_value": float,
                "percentage_of_total_payments": float,
            },
        }
    """
```

---

## Phase 5: Streamlit Dashboard (`src/dashboard/`)

### File: `src/dashboard/app.py`

Main Streamlit app with sidebar navigation. Use `st.set_page_config(page_title="AI Securewatch", page_icon="🔍", layout="wide")`.

**Sidebar:** AI Securewatch logo/title, navigation to pages, filter controls.

### Page: `src/dashboard/pages/1_vendor_reality.py`

**The Vendor Reality Snapshot page** — the money shot of the demo.

Show:
- Large metric cards: Total Vendors Discovered | In Procurement System | Unknown | % Gap
- Donut chart: Vendor classification breakdown
- Bar chart: Value flowing to each category
- Top 5 pattern flags as alert cards
- "Vendor Record vs Vendor Reality" side-by-side comparison

### Page: `src/dashboard/pages/2_entity_resolution.py`

Show entity resolution results:
- Total matches found across datasets
- Match probability distribution histogram
- Table of top matches (employee ↔ vendor director matches)
- Splink comparison viewer for selected match pairs
- Precision/recall metrics against ground truth

### Page: `src/dashboard/pages/3_relationship_graph.py`

Interactive graph visualization:
- Embed pyvis HTML graph using `st.components.v1.html()`
- Filter controls: entity type, risk threshold, community, fraud pattern
- Node click shows entity details
- Highlight fraud pattern subgraphs
- Show graph statistics (nodes, edges, communities, diameter)

### Page: `src/dashboard/pages/4_fraud_patterns.py`

Fraud detection results:
- Pattern type selector (Shell Companies, Self-Supply, Split Purchasing, etc.)
- For each pattern: count, total value at risk, list of flagged entities
- Evidence detail for each flag
- Confusion matrix against ground truth (detected vs embedded)

### Page: `src/dashboard/pages/5_risk_scores.py`

Entity risk ranking:
- Sortable table of all entities with risk scores
- Risk score distribution chart
- Drill-down: click entity to see all relationships, flags, and evidence
- Export capability (CSV download)

### File: `src/dashboard/components/metric_cards.py`

Reusable Streamlit components:
- `metric_card(title, value, delta, color)`: Large KPI display
- `alert_card(title, description, severity)`: Pattern flag alert
- `entity_detail_card(entity)`: Expandable entity information

### File: `src/dashboard/components/graph_widget.py`

Wrapper for embedding pyvis graphs in Streamlit:
- Generate HTML from NetworkX graph
- Apply consistent styling
- Handle click events

### File: `src/dashboard/components/data_tables.py`

Styled data tables with sorting, filtering, and download.

---

## Phase 6: FastAPI (`src/api/`)

### File: `src/api/main.py`

FastAPI app with CORS, exception handling, and route registration.

### File: `src/api/models.py`

Pydantic models for all request/response schemas.

### Routes:
- `GET /api/v1/entities` — List entities with pagination and filters
- `GET /api/v1/entities/{id}` — Entity detail with relationships
- `GET /api/v1/entities/{id}/relationships` — Entity's graph neighborhood
- `POST /api/v1/entities/resolve` — Run entity resolution on provided data
- `GET /api/v1/fraud-patterns` — List detected fraud patterns
- `GET /api/v1/fraud-patterns/{pattern_type}` — Detail for specific pattern
- `GET /api/v1/risk-scores` — Ranked risk scores
- `POST /api/v1/reports/vendor-reality` — Generate Vendor Reality Snapshot
- `POST /api/v1/graph/visualization` — Generate graph visualization HTML

---

## Phase 7: Integration Scripts & Tests

### File: `scripts/generate_data.py`

CLI script (use Typer) that runs the full data generation pipeline:
```bash
python scripts/generate_data.py --employees 200 --vendors 150 --transactions 5000
```

### File: `scripts/run_pipeline.py`

CLI script that runs the full analysis pipeline:
```bash
python scripts/run_pipeline.py
# 1. Load data from data/raw/
# 2. Run entity resolution
# 3. Build graph
# 4. Run fraud detection
# 5. Generate Vendor Reality Snapshot
# 6. Save all outputs to data/processed/
# 7. Print summary to console
```

### File: `scripts/run_demo.py`

CLI script that starts everything for demo:
```bash
python scripts/run_demo.py
# 1. Generate data (if not exists)
# 2. Run pipeline (if not processed)
# 3. Launch Streamlit dashboard
# 4. Print access URL
```

### Tests:
- `tests/test_sa_id.py`: Validate SA ID generation and validation
- `tests/test_entity_resolution.py`: Test Splink pipeline finds embedded matches
- `tests/test_fraud_patterns.py`: Test each detection algorithm against ground truth
- `tests/test_graph.py`: Test graph construction and algorithm outputs

---

## CRITICAL IMPLEMENTATION NOTES

### 1. Use Polars, not Pandas
All DataFrames should use **Polars** (`import polars as pl`). It's 5-22x faster and uses less memory. Splink returns DuckDB results that can be converted to Polars with `.pl()` or by reading from parquet.

### 2. Splink 4 API
Use Splink 4 API (not Splink 3). Key differences:
- `from splink import DuckDBAPI, Linker, SettingsCreator, block_on`
- `import splink.comparison_library as cl`
- Settings use `SettingsCreator` not `Settings`
- Training uses `linker.training.estimate_*` methods

### 3. Data Files are Parquet
All intermediate data uses **Parquet** format (not CSV). Read/write with Polars:
```python
df.write_parquet("data/raw/employees.parquet")
df = pl.read_parquet("data/raw/employees.parquet")
```

### 4. SA ID Numbers Must Be Valid
Every SA ID number in the dummy data must pass Luhn checksum validation. Use the `generate_sa_id()` function from `src/data_gen/sa_id_validator.py`.

### 5. Fraud Patterns Must Be Detectable
The embedded fraud patterns are the demo's proof of concept. If entity resolution or fraud detection misses them, the demo fails. Design the data to have clear signals:
- Self-supply loops: The SA ID number must match between employee and vendor director
- Shared identifiers: Exact matches on phone/address/bank account
- Split purchasing: Amounts clearly clustered below threshold

### 6. Graph Visualization Must Be Interactive
The pyvis graph should be navigable — zoom, pan, click nodes for details. Use physics simulation for layout. Cap at ~500 nodes for performance; filter for fraud-relevant subgraphs if needed.

### 7. Dashboard Must Work End-to-End
The Streamlit dashboard is the demo deliverable. It should:
- Load in <5 seconds
- Show real data from the pipeline outputs
- Allow interactive exploration
- Tell the story: "We started with payment data → found X hidden relationships → detected Y fraud patterns → here's the Vendor Reality gap"

### 8. Error Handling
Every module should handle missing data files gracefully. If `data/raw/` is empty, prompt to run `generate_data.py`. If `data/processed/` is empty, prompt to run `run_pipeline.py`.

---

## DEMO NARRATIVE

The dashboard should tell this story:

1. **"We ingested 12 months of payment data"** → Show transaction volume, vendor count
2. **"We discovered X entities receiving money"** → Show Vendor Reality gap (40% unknown)
3. **"Entity resolution found Y hidden connections"** → Show employee-vendor matches
4. **"Graph analysis revealed Z suspicious networks"** → Show interactive graph
5. **"We detected 7 categories of fraud patterns"** → Show pattern details
6. **"Here are the highest-risk entities"** → Show risk rankings
7. **"This is your Vendor Reality Snapshot"** → The 1-page summary output

This is the story that gets shown to MSC, investors, and Willem at FSG.

---

## GETTING STARTED

```bash
cd ai-securewatch-demo
pip install -r requirements.txt
python scripts/generate_data.py
python scripts/run_pipeline.py
streamlit run src/dashboard/app.py
```

Build Phase 1 first, test it, then proceed to Phase 2, etc. Each phase should work independently before integration.
