# UK Wildfire Risk Viewer & CFFDRS Tactical Forecaster

A client-side geospatial application engineered for mobile operational awareness, monitoring wildland fire risk across the United Kingdom. The system models the **Canadian Forest Fire Danger Rating System (CFFDRS)** over a spatial grid, cross-references values with **UK Met Office Fire Severity Index (FSI)** tiers, tracks satellite thermal anomalies, and provides forward-spread screening.

---

## 1. UK Meteorological & CFFDRS Mathematical Validation

Standard implementations often collapse daily weather data into naive minimums or averages, leading to significant calculation drift. This application enforces meteorological data integrity standards aligned with **Van Wagner (1987)** and the **Canadian Forest Service (CFS)**.

### Strict 12:00 UTC Sampling vs. Daily Aggregates
* **The Standard:** CFFDRS moisture codes ($FFMC$, $DMC$, $DC$) require single-point atmospheric observations sampled at solar noon / **12:00 Local Standard Time (LST)**.
* **The Problem with Daily Aggregates:** Pairing daily maximum temperature with daily mean relative humidity ($RH$) suppresses extreme drying events, artificially underestimating the Fine Fuel Moisture Code ($FFMC$) by 5 to 12 points.
* **Implementation:** The application queries Open-Meteo hourly models (ECMWF IFS / UKMO UKV downscaled) and isolates the **12:00 UTC** time step for:
  * 2-meter dry-bulb temperature ($T$)
  * 2-meter relative humidity ($RH$)
  * 10-meter open-wind speed ($W$)
  * 24-hour accumulated precipitation ($r_0$, 12:00–12:00 UTC)

### Stateful Daily Moisture Equilibrium
Moisture codes are calculated as a continuous temporal Markov chain rather than independent daily calculations:
Day (D+0) [FFMC₀, DMC₀, DC₀] + 12:00 UTC Weather (D+1) ──> Day (D+1) [FFMC₁, DMC₁, DC₁]
* **Fine Fuel Moisture Code ($FFMC$):** Models drying and wetting cycles of litter/cured fine fuels ($1\text{–}2\text{ cm}$) using equilibrium moisture content equations ($E_d, E_w$).
* **Duff Moisture Code ($DMC$):** Incorporates latitude- and month-indexed daylength factors ($L_e$) reflecting northern European solar irradiance patterns (50°N–59°N).
* **Drought Code ($DC$):** Tracks seasonal moisture depletion in deep, compacted organic layers and peat soils ($10\text{–}20\text{ cm}$), calibrated with potential evapotranspiration ($L_f$).

---

## 2. Statutory Alignment: EFFIS, UKMO FSI, & Land Management

The viewer reconciles European Forest Fire Information System (EFFIS) thresholds with domestic UK fire service and land management statutory mechanisms:

| Danger Class | EFFIS FWI Range | UK Met Office FSI | Statutory / Tactical Operational State |
| :--- | :--- | :--- | :--- |
| **Very Low** | $< 5.2$ | **Level 1** | Baseline monitoring; green vegetation. |
| **Low** | $5.2 - 11.2$ | **Level 1–2** | Prescribed moorland burning season window. |
| **Moderate** | $11.2 - 21.3$ | **Level 2–3** | Surface flash fires possible in cured grasses/calluna. |
| **High** | $21.3 - 38.0$ | **Level 3–4** | Elevated readiness; rapid surface fire spread. |
| **Very High** | $38.0 - 50.0$ | **Level 4** | Extreme head fire behavior; severe resistance to control. |
| **Extreme** | $\ge 50.0$ | **Level 5** | **Statutory Land Closure Trigger (CROW Act 2000).** |

### Countryside & Rights of Way (CROW) Act 2000 Compliance
Under **Section 24 of the CROW Act 2000**, relevant authorities (Natural England, Natural Resources Wales, Forestry Commission) possess statutory powers to suspend Open Access land rights on registered common land, heathland, and moorland during exceptional fire risk. 
* The application monitors the **$\text{FWI} \ge 50.0$** threshold and raises an explicit **CROW Act Closure Trigger** badge within the Point Inspector.

### Peatland & Peat Deficit Monitoring ($DC > 300$)
Superficial indices often fail to detect deep smouldering peat fires common in the Peak District, Pennines, and Scottish Highlands.
* **Peatland Vulnerability Threshold:** When the Drought Code ($DC$) exceeds **300**, deep organic layers decouple from surface moisture.
* **$DC > 400$:** Indicates critical peat desiccation, transitioning suppression operations from direct attack to extensive ground-soaking and deep trenching.

---

## 3. Data Ingestion & Stream Integrity

### NASA FIRMS Thermal Telemetry (VIIRS 375m)
* **Sensor Selection:** The pipeline prioritizes the **VIIRS S-NPP / NOAA-20** Active Fire Product (375m) over MODIS (1 km), ensuring higher spatial resolution for identifying localized UK wildland fires.
* **Dynamic Header Parsing:** To prevent token-index corruption inherent in fixed-column parsing, the client indexes the CSV schema dynamically:
  ```javascript
  const headers = lines[0].split(',').map(h => h.trim().toLowerCase());
  const frpIdx  = headers.indexOf('frp');
  const latIdx  = headers.indexOf('latitude');
  const lonIdx  = headers.indexOf('longitude');
Prescribed Burning Context Notice: Displays an operational notice alerting incident commanders to the Heather and Grass etc. Burning Regulations statutory window (uplands: 1 October – 15 April), mitigating false alarms from legal rotational muirburn.
Storage & Network Decoupling
Key Persistence: API credentials (NASA FIRMS MAP Key) persist locally via localStorage and bypass introductory modals on boot.
Zero External Telemetry Trackers: No tracking SDKs or ad scripts are included. All API calls target standard public endpoints directly (api.open-meteo.com, firms.modaps.eosdis.nasa.gov, nominatim.openstreetmap.org).
4. Wildfire Propagation Engine: Technical Limits
The onboard incident simulator integrates a directional Length-to-Width (L/W) Elliptical Perimeter Generator based on the Canadian Fire Behavior Prediction (FBP) system:
W
L
​	
 =1.0+0.1259×U 
0.785
 
Forward Rate of Spread (ROS)=(0.012×ISI+0.005×FWI)×K 
fuel
​	
 ×K 
slope
​	
 
Where U is open wind speed (km/h).
Points are projected trigonometrically across a rotated 16-node elliptical polygon anchored to the ignition origin.
Operational Disclaimer:

The incident simulation module is a simplified, 2D parametric screening model. It is intended strictly for table-top visualization and training. It does not incorporate real-time DEM elevation rasters, dynamic mid-flame wind reduction factors (WRF), or live canopy spotting physics, and must not be used as a primary tactical dispatch tool during active wildfire operations.

5. Architectural Specifications
Frontend: Single-file HTML5 / Vanilla ES6+ architecture (no build steps, zero node_modules dependencies).
Styling: Tailwind CSS with hardware-accelerated frosted glass panels (backdrop-filter) and an opaque #020617 solid navigation surface.
Mapping Engine: Leaflet 1.9.4 with dynamic layer toggling (OpenStreetMap cartographic / Google Satellite imagery).
Export Standards:
JSON: Serialized grid snapshot with full CFFDRS moisture codes.
CSV: Raw tabular telemetry for third-party spreadsheet ingestion.
GeoJSON: Vector feature collection containing cell polygons with attached hazard payloads.
6. Deployment
Deploy the file to any static host (GitHub Pages, Cloudflare Pages, S3/CloudFront):
