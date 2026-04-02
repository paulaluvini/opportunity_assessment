# Trade & Labor Data Pipeline (Complex Systems Approach)

This repository implements a data pipeline that integrates multiple layers of economic information—trade, labor, and education—using tools from complex systems and network analysis to generate insights on productive structure, specialization patterns, and diversification potential across U.S. states and metropolitan areas. By combining these data layers, the pipeline supports the construction of relational metrics and enables the identification of priority sectors, products, and occupations under different analytical lenses.

The workflow is:

1. **Build raw trade database** (`create_trade_database.ipynb`)
2. **Build raw labor database** (`create_labor_database.ipynb`)
3. **Compute complexity indexes** for either trade or labor (`create_indexes.ipynb`)
4. **Select priority sectors/products** using those indexes  
   - Trade: `select_activities_trade.ipynb`  
   - Labor: `select_activities_labor.ipynb`  
5. **Build occupation × education matrix** (`matrix_occupations_education.ipynb`) used to classify and filter occupations by education requirements in downstream labor analyses.

---

## Repository structure

### Notebooks

- **`create_trade_database.ipynb` – Build unified trade database**
  - Calls the **U.S. Census international trade API** to download exports by state and HS6 code.
  - Loads and cleans **BACI (CEPII)** international trade data for the same HS6 classification.
  - Harmonizes HS codes, cleans names (e.g. states like “Georgia (st)”), and merges:
    - U.S. state–level exports
    - Country–level exports from BACI
  - Produces:
    - `datasets/df_trade_usa_states.parquet` – U.S. exports by state–HS6.
    - `datasets/df_trade_baci.parquet` – world trade from BACI, cleaned & grouped.
    - `datasets/df_trade_complete.parquet` – unified trade database with a `location` column (states + countries), used later to compute RCA, PCI, etc.

---

- **`create_labor_database.ipynb` – Build labor database (CBP)**
  - Calls the **Census County Business Patterns (CBP) API** to download employment by:
    - Geography: `state` or `metropolitan statistical area`
    - Industry: NAICS codes
  - Maps CBP state/metropolitan codes to human–readable names.
  - Produces (depending on `labor_aggregation`):
    - `datasets_labor_states/df_labor_usa_states.parquet`
    - `datasets_labor_metropolitan_area/df_labor_usa_metropolitan_area.parquet`
  - These files are the labor equivalent of the trade database and will feed into the same index–generation pipeline as trade.

---

- **`create_indexes.ipynb` – Compute network-based specialization and relatedness metrics**
  - Core notebook that takes either trade or labor data and produces the full set of metrics used throughout the project.
  - Depending on the mode (`data_choice = "trade"` or `"labor"`), it reads unified trade data (HS6) or labor data (employment by NAICS at the state or metropolitan level) and constructs a location–activity matrix.
  - It then computes comparative advantage, location- and activity-level complexity scores, product/industry proximities, and feasibility and strategic value indicators (density, relative density, and COG).
  - Outputs are written to:
    - Trade: `./outputs_trade/`
    - Labor (states): `./outputs_labor_states/`
    - Labor (metros): `./outputs_labor_metropolitan_area/`

---
- **`select_activities_trade.ipynb` – Select trade diversification opportunities**
  - Uses precomputed trade-based metrics (RCA, ECI, PCI, density, proximity, strategic value).
  - For a chosen location (e.g. a U.S. state), identifies:
    - already specialized products, and
    - promising non-specialized products based on feasibility and strategic value.
  - Constructs composite opportunity scores under alternative criteria
    (e.g. feasibility-focused vs. long-jump strategies).
  - Outputs a multi-sheet Excel report:
    - `outputs_trade/outputs_<Location>.xlsx`
    - including ECI/PCI summaries, full product tables, and ranked opportunity lists.

---
- **`select_activities_labor.ipynb` – Select labor / industry diversification opportunities**
  - Labor analogue of the trade selection notebook, operating on employment by NAICS.
  - For a chosen state or metropolitan area:
    - identifies specialized and non-specialized industries,
    - ranks diversification opportunities using density, strategic value, and complexity.
  - Outputs a multi-sheet Excel report:
    - `outputs_labor_states/outputs_<Location>.xlsx` or
      `outputs_labor_metropolitan_area/outputs_<Location>.xlsx`
    - with summaries, full databases, and ranked industry opportunities.

---
- **`matrix_occupations_education.ipynb` – Build occupation × education matrix**
  - Constructs a crosswalk between occupations (SOC) and required education levels using national education data.
  - Classifies occupations by highest typical education requirement (e.g. Associate’s, Bachelor’s, Master’s, Doctoral/professional).
  - **Inputs:**  
    - Occupational employment and wage data (OEWS / national aggregates)  
    - Education-by-occupation table (BLS education requirements)
  - **Output:**  
    - `datasets_labor/matrix_occupations_education.csv` (occupation × education-level matrix)
  - **Used for:**  
    - Identifying “critical occupations” based on education requirements  
    - Filtering and grouping occupations in labor demand and workforce gap analyses

---

## How everything connects

The project has **two parallel pipelines** – one for trade, one for labor – that share the same complexity logic:

### Trade pipeline
1. **`create_trade_database.ipynb`**  
   → builds `df_trade_complete.parquet` (unified state + world trade).

2. **`create_indexes.ipynb`** with `data_choice = "trade"`  
   → reads `df_trade_complete.parquet`  
   → computes RCA, ECI, PCI, density, COG, proximity  
   → writes results to `outputs_trade/`.

3. **`select_activities_trade.ipynb`**  
   → reads `outputs_trade/`  
   → for a chosen state, ranks products/sectors and exports Excel reports.

### Labor pipeline
1. **`create_labor_database.ipynb`**  
   → builds `df_labor_usa_states.parquet` or `df_labor_usa_metropolitan_area.parquet`.

2. **`create_indexes.ipynb`** with `data_choice = "labor"`  
   → reads the labor parquet  
   → computes RCA/ECI/PCI/density/COG for NAICS activities  
   → writes results to `outputs_labor_states/` or `outputs_labor_metropolitan_area/`.

3. **`select_activities_labor.ipynb`**  
   → reads those outputs  
   → for a chosen state or metro, ranks industries and exports Excel reports.

Both pipelines can have a fourth step: 

4. **`matrix_occupations_education.ipynb`**
   → builds an occupation × education matrix used to flag occupations by required education group.

This structure lets you re–use exactly the same Economic Complexity machinery on both **export** and **labor** data and then compare or combine the diversification opportunities seen from each side.


## Related repositories & references

Parts of this repository build on, adapt, or are inspired by code and methodological work from the following open-source projects:

- **Paula Luvini. (2024).** datos-Fundar/complejidad_economica_empleo: Complejidad Económica de una provincia a través de datos de empleo (1.0). Zenodo. https://doi.org/10.5281/zenodo.13274055
  GitHub repository: https://github.com/datos-Fundar/complejidad_economica_empleo  
  - Source of core implementations for Economic Complexity metrics (ECI, PCI), RCA computation, and proximity/density logic.

- **goljavieglc, Juan Ignacio Cuiule, Marcos Feole, & Juan Pablo Ruiz Nicolini. (2024).** datos-Fundar/complejidad-economica: Complejidad Económica (v1.0). Zenodo. https://doi.org/10.5281/zenodo.13931731
  GitHub repository: https://github.com/datos-Fundar/complejidad_economica_verde  
  - Reference implementation for extending complexity and relatedness frameworks to environmental and green-production dimensions.