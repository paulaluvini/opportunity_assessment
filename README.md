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

- **`create_indexes.ipynb` – Compute Economic Complexity indexes**
  - Central notebook that takes **either trade or labor data** and produces all the EC metrics.
  - Parameters:
    - `data_choice = "trade" / "labor"`
    - `labor_aggregation = "state" / "metropolitan_area"` (for the labor case)
  - For **trade**:
    - Reads `datasets_trade/df_trade_complete.parquet` (unified trade database).
    - Aggregates by `[location, HS6]` to obtain a value matrix (`trade_value`).
  - For **labor**:
    - Reads `df_labor_usa_states.parquet` or `df_labor_usa_metropolitan_area.parquet`
      from the corresponding `datasets_labor_*` folder.
    - Aggregates employment by `[location, NAICS]`.
  - Common steps (for both trade and labor):
    - Builds a **location–product matrix** `value_level` (value or employment).
    - Computes **Revealed Comparative Advantage (RCA)** and the binary matrix `M` (1 if RCA ≥ 1).
    - Runs the standard **Economic Complexity algorithm** to compute:
      - **ECI** (Economic Complexity Index) per location.
      - **PCI** (Product Complexity Index) per product.
    - Computes **Mpa** (location–product presence matrix) from `M`.
    - Builds the **product–product proximity matrix** using co–export/presence patterns.
    - Calculates:
      - **Density** – how close each product is to a location’s existing capabilities.
      - **Relative density** – density normalized against available options.
      - **Strategic value / “COG”** – how useful a product is as a stepping stone to more complex products (taking into account proximities and PCI).
    - Exports to `OUTPUTS_DIR` (depends on mode, see below):
      - `RCA.parquet`
      - `Mpa.parquet`
      - `codes.parquet` (list of product codes: HS6 or NAICS)
      - `locations.parquet`
      - `value_level.parquet`
      - `eci.parquet` (location–level ECI)
      - `pci.parquet` (product–level PCI)
      - `proximity.parquet` (product–product network)
      - `relative_density.parquet`
      - `relative_cog.parquet`

  - Output folders (by mode):
    - Trade: `./outputs_trade/`
    - Labor, states: `./outputs_labor_states/`
    - Labor, metros: `./outputs_labor_metropolitan_area/`

---

- **`select_activities_trade.ipynb` – Select trade diversification opportunities**
  - Assumes **trade mode**:
    - `data_choice = "trade"`
    - Uses `./datasets_trade/` and `./outputs_trade/`.
  - Reads all precomputed EC matrices:
    - `relative_density.parquet`, `relative_cog.parquet`, `pci.parquet`
    - `codes.parquet`, `locations.parquet`, `RCA.parquet`, `Mpa.parquet`
    - `value_level.parquet`, `eci.parquet`, `proximity.parquet`
  - Sets a **target location**: `loc = "Michigan"` (or any other state).
  - For that location, it:
    - Distinguishes between **already specialized** products (M=1) and **non–specialized**.
    - Builds a full table (`df_full`) with, for each product:
      - PCI
      - Density and relative density
      - Relative “COG” / strategic value
      - Current specialization status (`mcp`)
      - Sector information (e.g. NAICS / strategic sector tags, including EV–related sectors).
    - Constructs **opportunity indexes** combining:
      - Density (feasibility)
      - Relative COG (strategic value)
      - PCI (complexity)
    - Ranks and marks **Top 25 products** per criterion.
  - Produces a **multi–sheet Excel file** per location:
    - `outputs_trade/outputs_<Location>.xlsx`, e.g. `outputs_trade/outputs_Michigan.xlsx`
    - Sheets include:
      - `ECI` – location–level ECI summary.
      - `PCI` – product–level PCI.
      - `Full database` – full product table with indexes.
      - `Selected HS` – top products selected by the diversification criteria.
      - `Specialized Sectors` – sectors where the location is already specialized.
      - `Not Specialized Sectors` – promising sectors not yet fully developed.
      - `NAICS Averages` – average PCI/density/COG by NAICS sector.
      - `Sector Averages` – averages by broader strategic sector categories.
      - `Proximity Matches` – product–product proximity matches for further exploration.

---

- **`select_activities_labor.ipynb` – Select labor/industry diversification opportunities**
  - Mirror of the previous notebook but in **labor mode**:
    - `data_choice = "labor"`
    - `labor_aggregation = "state"` or `"metropolitan_area"`
  - Reads the same EC outputs, but from:
    - `outputs_labor_states/` or `outputs_labor_metropolitan_area/`
  - Interprets the matrix as **employment by NAICS** instead of trade by HS.
  - For a chosen location (`loc = "Michigan"` or any other state/metro), it:
    - Identifies specialized and non–specialized NAICS activities.
    - Computes the same composite **opportunity index** (density + COG + PCI).
    - Focuses on identifying **industries** where:
      - The region has partial or no presence.
      - Density and COG suggest feasible and strategic upgrading of its labor structure.
  - Outputs a similar **Excel report**:
    - `outputs_labor_states/outputs_<Location>.xlsx` or
      `outputs_labor_metropolitan_area/outputs_<Location>.xlsx`
    - With sheets for ECI, PCI, full database, selected activities, specialized/non–specialized sectors, NAICS averages, sector averages, and proximity matches. Sheets include:
      - `ECI` – location–level ECI summary.
      - `PCI` – product–level PCI.
      - `Full database` – full product table with indexes.
      - `Selected HS` – top products selected by the diversification criteria.
      - `Specialized Sectors` – sectors where the location is already specialized.
      - `Not Specialized Sectors` – promising sectors not yet fully developed.
      - `NAICS Averages` – average PCI/density/COG by NAICS sector.
      - `Sector Averages` – averages by broader strategic sector categories.
      - `Proximity Matches` – product–product proximity matches for further exploration.


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

4. (Optional) **`matrix_occupations_education.ipynb`**
   → builds an occupation × education matrix used to flag occupations by required education group.


This structure lets you re–use exactly the same Economic Complexity machinery on both **export** and **labor** data and then compare or combine the diversification opportunities seen from each side.
