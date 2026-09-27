# Predicting Gray Wolf Habitat Suitability and Recolonization Potential in the Continental US

## Aim

Build a habitat suitability model that identifies where gray wolves (*Canis lupus*) can sustain populations in the continental US based on environmental and human landscape features. The model predicts suitability across the entire landscape — not just where wolves currently exist — to identify high-suitability areas where wolves haven't yet established, framing these as potential recolonization zones.

This is not a sighting prediction model. Occurrence data is used as a signal for habitat suitability, with explicit handling of the gap between "where wolves were observed" and "where wolves can live" (observation bias).

---

## Methodology

### 1. Problem Framing

- **Target variable:** Binary — wolf observed (1) vs. pseudo-absence (0)
- **Spatial resolution:** ~10km grid cells across the continental US
- **Temporal scope:** Last 10–15 years of occurrence data (reflects current range, not historical)

### 2. Data Sources

| Data | Source | Purpose |
|------|--------|---------|
| Wolf occurrence records | GBIF (*Canis lupus*) | Positive class (presence points) |
| Terrain (elevation, slope, ruggedness) | USGS | Environmental covariates |
| Land cover (forest, grassland, developed, etc.) | NLCD | Landscape composition features |
| Road network | TIGER / US Census | Human pressure features |
| Human population density | US Census | Human pressure features |
| Climate (temperature, precipitation, snowfall) | PRISM or WorldClim | Climate covariates |
| Protected areas | PAD-US | Proximity / coverage features |
| Elk/deer habitat data | USDA / state wildlife agencies | Prey availability proxy |

### 3. Pseudo-Absence Generation

Wolves don't have confirmed-absent records. Background points (pseudo-absences) must be generated, and how they're generated fundamentally shapes what the model learns.

- **Naive approach (avoid):** Random points across the entire US → model learns trivially that wolves live in mountains, not Florida.
- **Better approach:** Constrained background sampling within plausible range or environmental envelope, forcing the model to distinguish suitable from unsuitable habitat in ambiguous zones.
- **Strategy must be explicitly chosen, justified, and discussed in the writeup.** This is a modeling decision, not a preprocessing step. Published methods to consider: target-group background, environmental profiling, distance-constrained sampling.

### 4. Feature Engineering

This is the core of the project. Raw spatial layers are ingredients, not features. Turning them into a modeling-ready dataset requires decisions that encode ecological hypotheses.

#### Landscape Composition at Multiple Scales
- Land cover classification gives a categorical label per pixel ("forest," "grassland," etc.)
- Engineer features as proportions within spatial buffers: % forest within 5km, 10km, 50km
- Different buffer sizes encode different hypotheses — a wolf needs territory-scale habitat, not a single pixel
- Choice of scale must be justified

#### Human Pressure (Computed from Raw Data)
- **Road density:** Calculated from road shapefiles — total road length within a chosen buffer around each point. Buffer size is a decision.
- **Distance to nearest town:** Computed spatially, not downloaded as a column
- **Population density:** Aggregated to grid cell level

#### Distance Features
- Distance to nearest protected area edge
- Distance to nearest known pack territory
- Distance to nearest major highway or interstate

#### Landscape Connectivity (Derived — Does Not Exist in Any Dataset)
- How connected is a point to existing wolf range via continuous habitat?
- Requires building a resistance surface (each land cover type assigned a "cost" to wolf movement) and running connectivity analysis
- A patch of perfect habitat surrounded by highways and cities is functionally different from one connected to established packs by a forest corridor
- Non-trivial feature engineering with real ecological meaning

#### Interaction / Composite Features
- Prey suitability × forest cover × (1 / human density) — encodes the hypothesis that wolves need prey, in cover, away from people
- Building and testing these composite features as encoded ecological hypotheses is the data science

#### Edge Features
- Proportion of habitat edge (forest-to-open transitions) within a buffer
- Wolves hunt along edges — this is derived from land cover classification boundaries, not available in any dataset directly

#### Seasonal Features (If Temporal Data Included)
- Snow depth in winter (changes movement cost)
- Vegetation greenness in summer (proxies prey forage quality)
- Same location, different feature values by season

### 5. Validation Design

**Spatial cross-validation, not random CV.**

Random CV with spatial data leaks information — nearby points are correlated, so a model that memorizes "wolves are in Yellowstone" scores well on random splits but learns nothing generalizable. Spatial CV uses block holdouts by geographic region to prevent this.

Implementation: `sklearn` `GroupKFold` with spatial blocks, or the `spacv` package.

This is a key differentiator in the writeup — most portfolio projects use naive random splits on spatial data without acknowledging the leakage.

### 6. Models

Run in progression, not just "try everything":

1. **Logistic regression** — interpretable baseline, sets the floor
2. **Random forest** — captures nonlinear interactions, feature importance for free
3. **XGBoost or LightGBM** — primary model, properly tuned via hyperparameter search
4. **Elastic net** — regularized linear model as a comparison point

### 7. Evaluation Metrics

- **AUC-ROC** — overall discrimination ability
- **Precision-recall curves** — critical because of class imbalance (more absences than presences)
- **Calibration curves** — are predicted probabilities actually meaningful?
- Model comparison table across all metrics

### 8. Observation Bias Handling

The data captures where wolves were *detected*, which is influenced by both where wolves live and where humans are looking. A wolf in remote wilderness may never be observed; a wolf crossing a highway gets reported immediately.

Mitigation:
- Thoughtful pseudo-absence generation (not placing background points only where people are)
- Including observation effort proxies as covariates (road density, proximity to towns partially control for detection probability)
- Explicitly acknowledging this limitation in the writeup

**Occupancy modeling** (jointly estimating detection probability and true occurrence) is a more formal solution but requires repeated surveys at the same sites, which GBIF data doesn't provide. Mentioned as a limitation and future work direction.

---

## Key Outputs

1. **Predicted habitat suitability map** across the continental US — the hero visual
2. **Feature importance analysis** with ecological interpretation (not just "elevation was important" but *why*, tied to published ecological research where possible)
3. **Model comparison table** showing where simpler models match or beat gradient boosting
4. **Recolonization frontier analysis** — high-suitability areas where wolves aren't yet established, with discussion of what barriers (human, landscape) may explain the gap

---

## Optional Stretch Goals

- **Streamlit deployment:** User picks a location → suitability score + top contributing factors
- **Linked prey-species modeling:** Wolf occurrence modeled alongside elk/deer as a connected system
- **Temporal range shift analysis:** How has the suitability frontier moved over the last 15 years?

---

## Writeup Standards

Research-style, not just a README. Includes:
- Methodology justification (why this spatial scale, why these features, why this pseudo-absence strategy)
- Spatial CV reasoning and comparison to naive random splits
- Results connected to published ecological literature ("model agrees with Smith et al. that road density is the dominant barrier")
- Honest discussion of limitations (observation bias, pseudo-absence assumptions, data gaps)
- Future work section (occupancy modeling, finer resolution, additional covariates)

---

## What Makes This Competitive

- Sourced own problem and data (not Kaggle, not a tutorial)
- Feature engineering from raw spatial layers, not prebuilt feature tables
- Spatial CV demonstrating awareness of data leakage in spatial problems
- Ecological reasoning driving feature selection and interpretation
- Pseudo-absence strategy as a deliberate, justified modeling decision
- Clean software engineering: proper repo structure, tests, reproducible environment
- The project is a conversation starter for interviews — depth of understanding is the real evaluation

---

## Estimated Timeline

4–5 weeks at ~1.5–2 hours on weekdays, 3–4 hours one weekend day.
