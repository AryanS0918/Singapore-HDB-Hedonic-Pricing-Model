# Singapore HDB Resale Prices: A Hedonic Pricing Model
 
A hedonic pricing analysis of Singapore's HDB resale market, using 240,000+ transactions to estimate the implicit price of a flat's physical attributes (floor area, remaining lease, storey) and its location (town, distance to the nearest MRT station).
 
## Framing
 
This is an **inference** problem, not a prediction problem. Following Rosen's (1974) hedonic pricing framework, the goal is to recover interpretable marginal prices for each attribute — not to minimize forecast error. Specification correctness and coefficient interpretability are prioritized throughout; a chronological out-of-sample check is included as a robustness test, not as the primary objective.
 
## What this project does
 
1. **Geocodes** ~9,700 unique HDB addresses via the OneMap API.
2. **Computes** distance from each address to the nearest MRT station using the haversine formula.
3. **Engineers features**: parsed remaining lease (years), log-transformed price, transaction year, and a numeric storey midpoint (parsed from `storey_range`, e.g. "07 TO 09" → 8.0).
4. **Builds four models in sequence**, each adding a deliberate refinement:
   - Model 1 — baseline linear regression (levels)
   - Model 2 — semi-log specification with town fixed effects
   - Model 3 — adds a **quadratic** remaining-lease term (leases don't decay linearly) and storey level
   - Model 4 — adds a floor-area × MRT-distance interaction, tested against the full Model 3 specification
5. **Validates out-of-sample** using a strict chronological train/test split (train ≤2024, test 2025+) — not random k-fold, since resale prices trend upward over time and a random split would leak future price levels into training.
## Key findings
 
| Model | R² |
|---|---|
| 1. Linear (levels) | 0.3858 |
| 2. Semi-log + town FE | 0.6346 |
| 3. + quadratic lease + storey + year FE | **0.9167** |
| 4. Model 3 + interaction | 0.9168 |
 
- **MRT proximity carries a real, sizeable penalty**: roughly **14.2%** lower price per additional km from the nearest MRT station (Model 3), holding location, size, and lease constant.
- **Remaining lease decays non-linearly, not linearly.** The linear lease coefficient is positive while the squared term is negative and highly significant — a concave-down relationship. Losing a year off a fresh 90-year lease barely moves price; losing a year off a 50-year lease triggers a much steeper drop, consistent with tightening bank financing and CPF withdrawal limits as leases run down.
- **Storey level carries a modest, significant premium**: roughly 0.8% per floor, holding location and time constant.
- **The floor-area × MRT-distance interaction is statistically significant but practically negligible** (ΔR² < 0.0001 over Model 3) — larger flats do face a measurably steeper MRT-distance penalty, but the effect is too small to justify preferring the more complex model. Model 3 is the preferred specification on parsimony grounds.
- **The model generalizes well out-of-sample**: holdout R² (0.8958, on 2025+ transactions never seen during training) is comparable to in-sample R² (0.8893) — reassuring evidence the model isn't overfit to historical patterns.
## Files
 
- `HDB_Hedonic_Pricing_Final.ipynb` — main analysis notebook
- `data/ResaleflatpricesbasedonregistrationdatefromJan2017onwards.xlsx` — HDB resale transactions (source: [data.gov.sg](https://data.gov.sg))
- `data/LTAMRTStationExitGEOJSON.geojson` — MRT station exit locations (source: [data.gov.sg](https://data.gov.sg))
- `data/hdb_coordinates_backup.csv` — cached geocoding results, so the notebook doesn't need to re-query OneMap on every run
- `requirements.txt` — Python dependencies
Note: `hdb_resale_final_engineered.csv` (the fully merged, feature-engineered dataset) is **not** committed to this repo — at ~62 MB it exceeds a convenient web-upload size, and it's fully reproducible by running the notebook top to bottom.
 
## How to run
 
```bash
pip install -r requirements.txt
```
 
The notebook needs a OneMap API token to geocode any addresses not already in the cache (all 9,744 addresses are cached, so a fresh run shouldn't need one — but get your own token at [onemap.gov.sg/apidocs](https://www.onemap.gov.sg/apidocs/) if you want to verify the geocoding step yourself):
 
```bash
export ONEMAP_TOKEN="your_token_here"
```
 
Then run all cells in order.
 
## Methodology notes & limitations
 
- **Distance metric**: haversine (straight-line) distance likely understates true walking/commute distance, since it ignores road layout and pedestrian barriers.
- **Omitted variable bias**: MRT proximity correlates with other amenities (malls, bus interchanges, popular schools) not captured in the model, so the MRT-distance coefficient likely absorbs some of their effect, slightly overstating the pure "transit premium."
- **Geocoding coverage**: addresses that failed to geocode were dropped; if failures were geographically clustered rather than random, this could introduce a small selection bias.
- **MRT vintage**: distance is computed to all *currently existing* stations, including some that opened after certain transactions took place — a look-ahead bias not corrected for in this version. A vintage-aware correction (using station opening dates, or ideally announcement dates to capture anticipation effects on price) is a natural extension, left for future work.
- **Sample period**: transactions span January 2017 onward; earlier market regimes aren't captured.
## Possible extensions
 
- MRT vintage correction — restrict each transaction's distance calculation to only stations open (or announced) at the time of sale
- A spline (rather than quadratic) specification for lease decay, for a more flexible functional form
- Extending the model with school proximity or CBD distance as additional locational controls
