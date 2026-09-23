# Wilkins Spillway Eco‑Operational Ramp‑Down Dashboard

Decision‑support Shiny application for planning ecologically sensitive ramp‑down operations at Wilkins Spillway, comparing sudden closure against controlled, multi‑step reductions in discharge.

---

## Purpose and overview

This application supports operators, engineers, and biologists in planning spillway ramp‑down operations that balance:

- Ecological protection of unionid mussel assemblages in the Tombigbee/Bigbee reach.
- Navigation pool constraints and project storage limits.
- Real‑time river conditions from USGS gages.

The app provides a side‑by‑side comparison of a **sudden closure** (high‑risk scenario) versus a **gradual, stepwise ramp‑down** (safe/best‑available scenario), under user‑specified constraints on maximum stage decline and minimum navigation pool elevation.

---

## Key features

- **Live river conditions (USGS):**  
  Pulls recent headwater and discharge data for Bigbee, Wilkins, Fulton River, and Fulton Lock/Pool via the `dataRetrieval` R package, with 6‑hour trends and simple “rising/falling/stable” indicators.[web:7]  

- **Gate schedule recognition:**  
  Infers the current operational step from individual gate openings (GATE_1–GATE_11) using a predefined `gate_schedule_lookup` table. Gate 7 is excluded from step recognition due to known telemetry issues; bypass gate series are also excluded.[web:7]  

- **Ecological risk assessment (72‑hour lookback):**  
  Uses the last 72 hours of Bigbee flow to classify ecological risk. When flows have been elevated above a conservative threshold for sufficient duration, the app flags **HIGH RISK** and emphasizes the need for gradual ramp‑down; otherwise it reports **LOW RISK**.[file:1]  

- **Ramp‑down planning and constraints:**  
  Designs a step sequence from the current step to Step 0, constrained by:
  - User‑specified **maximum allowable stage decline (ft/hr)**.  
  - User‑specified **minimum navigation pool elevation (ft)**.  
  - A target number of intermediate steps.  

  When navigation limits storage, the app accelerates the ramp‑down and reports the resulting higher daily stage decline.

- **Operational narrative and schedule:**  
  Generates:
  - A plain‑language narrative summarizing current flows, spillway contribution to Bigbee, estimated Nettleton depth, recognition mode (manual vs telemetry), and whether navigation constraints are forcing faster‑than‑ideal ramp‑down.  
  - A comparative hydrograph for sudden vs gradual closure.  
  - A recommended action schedule with times, target steps, estimated discharges, predicted Nettleton depth, and gate settings.

---

## Data sources

The app uses real‑time USGS data via `dataRetrieval::readNWISuv()`.[web:7]  

Current implementation references:

- **Bigbee River discharge**  
  - Station: `02433500`  
  - Parameter: `00060` (discharge, cfs)  
  - Role: upstream flow context and 72‑hour ecological risk lookback.

- **Wilkins Spillway headwater**  
  - Station: `02433151`  
  - Parameters: `00065` / related headwater series  
  - Role: project pool elevation.

- **Wilkins gate telemetry**  
  - Station: `02433151`  
  - Parameter: `45592` (direct gate series `X_Gate.1`–`X_Gate.11`)  
  - Role: gate openings for step recognition (Gate 7 excluded).

- **Fulton River discharge**  
  - Station: `02431000`  
  - Parameter: `00060`  
  - Role: upstream river flow context.

- **Fulton Lock / Fulton Pool headwater and inflow**  
  - Station: `02431011`  
  - Parameters: `00065` (headwater), `00060` (discharge/inflow)  
  - Role: navigation pool inflow and stage context.

If USGS calls fail or data are missing, the app:

- Uses internal conservative defaults (e.g., a default Bigbee flow) and clearly labels values as “Data Unavailable” or “Manual Override” in the UI.  
- Displays notifications about failed data fetches so reviewers can see when live telemetry was not available.[web:7]  

---

## Core model logic

### Gate schedules and discharge

- `gate_schedule_lookup` defines gate openings for each step (0–14) and corresponding discharges under multiple tailwater conditions.  
- `get_step_discharge(step_num, tailwater)` returns modeled spillway discharge for a given step and tailwater elevation:
  - Uses table values directly for tailwaters between 215–230 ft.  
  - For higher tailwaters, interpolates between discharges at 230, 235, 240, and 244 ft.[web:7]  

- `calc_q_from_gates()`:
  - Takes observed gate openings (manual inputs or USGS gate telemetry).  
  - Excludes Gate 7 from the matching process.  
  - Finds the step in `gate_schedule_lookup` whose gate pattern is closest in Euclidean distance to observed gates.  
  - Returns the recognized step, modeled discharge, and pattern match distance.

### Nettleton depth approximation

The app uses a simple linear approximation for predicted depth at Nettleton:

- Depth is modeled as a linear function of spillway discharge and Bigbee flow, with coefficients derived from internal rating‑curve analysis, and clipped at zero depth.  
- This is intended as a **planning‑level estimate**, not a fully calibrated hydraulic model, and should be interpreted relative to observed field depths and species‑specific tolerances.[file:1]  

### Ramp‑down planning (`op_plan()`)

Given the recognized step and discharge, plus user‑specified constraints:

- Computes a total stage drop associated with the current spillway discharge.  
- Computes an “ideal” ramp duration to keep **daily stage decline** at or below the target (e.g., 0.25 ft/day, expressed as ft/hr in the UI).  
- Chooses a sequence of steps from current step down to 0, using a user‑selected number of intermediate steps.  
- Calculates total storage used from the Wilkins pool, given:
  - A capacity curve relating pool elevation and storage (ac‑ft).  
  - A user‑specified **minimum navigation pool elevation**.

If the ideal volume required for the ecological ramp‑down would push Wilkins Pool below the navigation minimum:

- The plan is re‑computed using the **maximum allowable storage** that preserves the navigation minimum.  
- Ramp duration is shortened accordingly.  
- The resulting daily stage decline rate is reported, and flagged when it exceeds the ecological target.

### Hydrograph and schedule (`ramp_data()`)

- Builds two time series scenarios:
  - **Sudden Closure (High Risk)**: discharge drops from current Q to 0 abruptly.  
  - **Gradual Ramp‑Down (Safe/Best Available)**: discharge follows the planned step sequence over the computed ramp duration.

- For each scenario:
  - Computes predicted Nettleton depth over time.  
  - Provides data for the hydrograph plot.  
  - Generates a recommended schedule table for the gradual ramp‑down with:
    - Action time stamps.  
    - Target step.  
    - Estimated discharge (cfs).  
    - Predicted Nettleton depth (ft).  
    - Gate openings (GATE_1–GATE_11) from `gate_schedule_lookup`.

---

## Ecological risk assessment

The “Ecological Risk Assessment (72‑Hour Lookback)” panel:

- Uses recent Bigbee discharge to compute a threshold:  
  - `threshold = max(1000, 0.5 * current Bigbee flow)` (cfs).  
- Counts hours during the last 72 hours where Bigbee flow exceeded this threshold (using 15‑minute data).  
- Classifies risk:
  - **HIGH RISK**: elevated duration > 48 hours.  
  - **LOW RISK**: elevated duration ≤ 48 hours.

This heuristic is intentionally conservative, informed by:

- Field studies in Pool 6 of the Upper Mississippi River, where planned drawdowns produced species‑ and slope‑specific mussel mortality, with higher mortality at low‑slope shallow sites and evidence that mussels respond to drawdowns via vertical burrowing or horizontal movement.[file:1]  
- Observations in the Apalachicola system indicating that heavy‑shelled fat threeridge mussels can move tens of centimeters per day to track receding water levels, but still experience exposure and mortality at low‑gradient sites under sustained low flows.[web:29]  

The risk classification is meant to highlight when mussels have had time to migrate onto shallow bars and gravel patches that are likely to be dewatered, increasing the stakes of ramp‑down decisions. It is **not** a substitute for site‑specific population surveys or expert ecological judgment.

---

## Scientific basis and supporting studies

### Upper Mississippi River Pool 6 drawdown (Newton et al. 2015)

Newton et al. (2015) evaluated mortality, movement, and behaviour of native mussels during a 0.3 m summer drawdown in Navigation Pool 6 of the Upper Mississippi River.[file:1] Relevant findings include:

- Mortality was low (~5%) at reference sites but much higher at shallow treatment sites (means ~27% for Lampsilis cardium and ~52% for Amblema plicata), with strong slope dependence.[file:1]  
- Mussels exhibited species‑specific responses: some primarily burrowed vertically, others moved horizontally to deeper water, and weekly movement distances were significantly correlated with changes in water elevation.[file:1]  
- Managers used these results to refine drawdown planning and assess incidental impacts on native assemblages.

These patterns motivate:

- Emphasizing **rate of stage decline** (ft/day) as a key ecological metric.  
- Treating prolonged elevated flows and low‑slope shallow habitats as higher‑risk.  
- Making navigation‑driven accelerations explicit in the narrative so trade‑offs are transparent.

Reference:  
Newton, T. J., Zigler, S. J., & Gray, B. R. (2015). Mortality, movement and behaviour of native mussels during a planned water‑level drawdown in the Upper Mississippi River. *Freshwater Biology*, 60, 1–15.[file:1]  

### Fat threeridge drawdown study (Apalachicola / Chipola; Kaeser & Herrington 2011)

U.S. Fish and Wildlife Service work on the endangered fat threeridge (Amblema neislerii) in the Apalachicola and Chipola Rivers documented:

- Daily movement capacities on the order of 50–100 cm to track receding water, with occasional longer moves on the order of 2–3 m.  
- Survival of exposed individuals typically spanning 1–6 days, with a small subset that completely buried into the substrate surviving for 1–3+ weeks depending on microclimate.  
- Elevated mortality at low‑gradient sites during drawdowns, with a clear inverse relationship between site slope and percent mortality.  
- Evidence that some mussels died in the water (never exposed to air), likely from prolonged stress, temperature, or DO, indicating that stranding is not the only pathway to mortality.

These findings support:

- Using conservative down‑ramping rates as a baseline.  
- Paying attention to slope and microhabitat, not just absolute stage change.  
- Acknowledging that even “successful” movement can carry physiological costs.

---

## Species and assemblage context for the Tombigbee

ACF maximum fall‑rate guidance was developed with particular attention to fat threeridge, a heavy‑shelled unionid used as a conservative reference species.[web:29] The Tombigbee / Bigbee reach supports a different but related assemblage of unionids, including:

- **Generalists and large‑river taxa**  
  - Southern/Ornate Pocketbook (*Lampsilis ornata*).  
  - Alabama Orb (*Rotundaria asperata*).  
  - Fragile Papershell (*Potamilus fragilis*).  
  - Three Ridge (*Amblema plicata*).  
  - Yellow Sandshell (*Lampsilis teres*).  

- **Coarse‑substrate and large‑river specialists**  
  - Butterfly (*Ellipsaria lineolata*).  
  - Bleufer (*Potamilus purpuratus*).  
  - Gulf Mapleleaf (*Quadrula nobilis*).  
  - Inflated Heelsplitter (*Potamilus inflatus*).  
  - Deer Toe (*Truncilla truncata*).  

- **Nonnative bivalves**  
  - Asian Basket Clam (*Corbicula* spp.) common at all sites.

Recent surveys at Lost Corner, Nettleton, and Standifer Creek indicate:

- Unionids are most frequently associated with shallow to moderate‑depth coarse sand‑gravel and mixed alluvial beds.  
- Occupancy is controlled by local hydraulic conditions, sediment stability, and fine‑material cover; coarse gravel alone is not sufficient.  
- Low‑energy, fine, soft substrates tend to support few or no unionids even where gravel is present at depth.

Adopting a fat‑threeridge‑based fall‑rate (0.25 ft/day) as a starting point is therefore intentionally conservative for this assemblage. The app lets operators adjust the target fall rate as more site‑specific information on movement, stranding, and survival becomes available for Tombigbee taxa.

---

## Maximum fall rate rationale

The default maximum daily stage decline used in this app (0.25 ft/day, about 3 inches per day) is adapted from Apalachicola–Chattahoochee–Flint (ACF) guidance:

- ACF Master Water Control Manual and related operations limit down‑ramping rates at the Chattahoochee gage under lower releases (within powerhouse capacity and ≤10,000 cfs) to **0.25 ft/day or less**, with higher allowable rates only at higher flows.[web:30][web:31]  
- This limit was developed to avoid rapid drops that outpace movement capacity of slow‑moving organisms such as sturgeon and unionid mussels, including fat threeridge.[web:29]  

The Tombigbee system is less regulated and hydrologically distinct from the ACF Basin. In this app:

- **0.25 ft/day is treated as a conservative starting point**, not a rigid rule.  
- Operators can adjust the “Max Allowable Stage Decline (ft/hr)” input based on:
  - Real‑time observations of mussel stranding and movement.  
  - Site‑specific slope and substrate information (e.g., low‑gradient bars vs coarse patches).  
  - Navigation and project‑storage considerations.

When a higher fall rate is selected:

- The narrative explicitly states that the resulting daily decline **exceeds the baseline ecological target**.  
- When navigation constraints are forcing faster‑than‑ideal ramp‑down, this is highlighted so trade‑offs are visible.

---

## Installation

### Dependencies

You need R and the following packages:

- `shiny`  
- `ggplot2`  
- `dplyr`  
- `lubridate`  
- `tibble`  
- `dataRetrieval`[web:16][web:7]  

Install packages in R:

```r
install.packages(c(
  "shiny", "ggplot2", "dplyr",
  "lubridate", "tibble", "dataRetrieval"
))
```

For reproducibility, it is recommended to capture the environment with `renv` and record `sessionInfo()` output or a `renv.lock` file.[web:7][web:20]  

---

## Running the app

If the app is in a single `app.R` file:

```r
library(shiny)
runApp("path/to/app.R")
```

If using `ui.R` + `server.R` (and optional `global.R`), place files together in a project directory and run:

```r
library(shiny)
runApp("path/to/project")
```

Ensure the machine has network access to USGS; otherwise, the app will rely on internal defaults and display notifications for failed data fetches.[web:7]  

---

## Usage and workflow

Typical workflow:

1. **Refresh live data**  
   - Click “Refresh Live USGS Data” to pull recent telemetry for Bigbee, Wilkins, and Fulton.

2. **Define current operations**  
   - Choose one of:
     - “Select Operational Step” (direct entry of current step).  
     - “Enter Individual Gates” (manual gate openings).  
     - “Use USGS Gate Telemetry” (automatic step recognition from gate series).

3. **Set constraints**  
   - Minimum navigation pool elevation (ft).  
   - Maximum allowable stage decline (ft/hr).  
   - Desired number of intermediate steps.

4. **Review outputs**  
   - Ecological risk panel (72‑hour Bigbee flow).  
   - Live river conditions and trends.  
   - Recognized step & flow, ramp duration, expected final pool elevation.  
   - Hydrograph comparing sudden vs gradual closure.  
   - Recommended operational schedule.

5. **Communicate plan**  
   - Use the narrative and schedule to communicate ramp‑down decisions and expected ecological implications.

---

## Assumptions and limitations

- **Simplified rating curves:**  
  Discharge and depth relationships are implemented as linear approximations over a limited range of flows and pool elevations. Operations outside these ranges may require updated curves or additional validation.

- **USGS data quality:**  
  The app assumes USGS data are timely and reasonably accurate, but does not perform full QA/QC. Known telemetry issues (e.g., Gate 7) are handled by exclusion; other anomalies may require operator judgment.[web:7]  

- **Species and slope sensitivity:**  
  Published studies show strong species‑specific and slope‑dependent responses of unionid mussels to drawdowns; low‑gradient shallow sites can experience disproportionately high mortality.[file:1][web:29] This app does not explicitly model slope or individual species; its heuristics are conservative and should be paired with site‑specific habitat and assemblage information.

- **Advisory role:**  
  The app is designed as a decision‑support and communication tool, not an automated control system. Final operational decisions remain with qualified operators, engineers, and ecologists.

---

## Hosting and deployment

This repository is intended to:

- Host the Shiny source code and documentation (README, supporting notes).  
- Serve a small documentation site via GitHub Pages (e.g., under `docs/`), linking to:
  - The live Shiny app hosted on shinyapps.io, Posit Connect, or a Shiny Server instance.  
  - Supporting studies and management guidance (e.g., Newton et al. 2015, Kaeser & Herrington 2011, ACF Master Manual).[web:43][web:46][web:59]  

For interactive use, the Shiny app itself must be hosted on a platform that runs an R session (e.g., shinyapps.io or Shiny Server); GitHub Pages provides static documentation only.[web:43][web:46]  

---

## Repository structure (suggested)

- `app.R` or (`ui.R`, `server.R`, `global.R`) – Shiny app code.  
- `README.md` – this documentation.  
- `data/` – optional small configuration or rating‑curve data.  
- `docs/` – GitHub Pages entry (`index.md`) and additional documentation.  
- `renv.lock` / environment notes – optional reproducibility and package versions.

---

*This README is intended for technical reviewers (reddot process) and operational staff. It documents the scientific basis, assumptions, and intended use of the Wilkins Spillway ramp‑down decision‑support tool.*
