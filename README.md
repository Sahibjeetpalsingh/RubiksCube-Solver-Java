<div align="center">



# Renewable Energy Classifier

*Given the climate of a place, which renewable energy source does it actually support?*

[![Live App](https://img.shields.io/badge/Live_App-weather--energy.netlify.app-1db954?style=flat-square&logo=netlify&logoColor=white)](https://weather-energy.netlify.app/)
[![R](https://img.shields.io/badge/R-3000%2B_lines-276DC3?style=flat-square&logo=r&logoColor=white)](https://github.com/Sahibjeetpalsingh/Weather-Classification)
[![NOAA](https://img.shields.io/badge/Data-NOAA_GHCN-0072C6?style=flat-square)](https://www.ncdc.noaa.gov/data-access/land-based-station-data/land-based-datasets/global-historical-climatology-network-ghcn)
[![License: MIT](https://img.shields.io/badge/License-MIT-f2c94c?style=flat-square)](LICENSE)

<img src="docs/images/Gemini_Generated_Image_nytdnznytdnznytd.png" width="100%" alt="Renewable Energy Classifier" />
</div>

<br>

The renewable energy conversation usually stops too early. People say things like *"solar is good in sunny places"* or *"wind works on coasts"* and technically they are right. But that level of precision does not help a homeowner decide whether rooftop solar is worth the investment, or a student building a regional sustainability case, or a planner comparing options across a watershed.

What they actually need is a sharper question: **given the specific climate of this exact place, which energy source does the physics actually favour and by how much?**

That is the question this project was built to answer. Not with a model trained on labels someone else assigned. With a transparent scoring engine grounded in climate physics, validated against real NOAA station data, and wrapped in an interface anyone can open in a browser without installing anything.

<br>

## See It in Action

<p align="center">
  <a href="https://weather-energy.netlify.app/">
    <img src="docs/images/Recording+2026-03-09+170553.gif" width="760" alt="App demo" />
  </a>
</p>

Move a slider. The scores update instantly. The recommendation appears with its full reasoning, not just a label but every condition that mattered and every point it contributed. That transparency is not a design flourish. It is the whole point.

<br>

## What It Looks Like

The interface has three distinct views, each serving a different part of the experience.

---

### 01 &nbsp; Climate Controls

> Set any climate profile using five sliders or jump straight in with one of eight built-in presets-- Desert, Monsoon, Mountain, Coastal and more. No configuration, no install. Open the page and the tool is ready.

<p align="center">
  <a href="https://weather-energy.netlify.app/">
    <img src="docs/images/1.png" width="92%" alt="Climate sliders and preset controls" />
  </a>
</p>

---

### 02 &nbsp; Score Breakdown

> Every score is shown, not just the winner. You can see exactly why Solar lost to Hydro or why Wind barely edged out the competition. The margin between first and second is always visible, so you always know how much to trust the recommendation.

<p align="center">
  <a href="https://weather-energy.netlify.app/">
    <img src="docs/images/2.png" width="92%" alt="Full score breakdown panel" />
  </a>
</p>

---

### 03 &nbsp; R Analysis Dashboard

> The R pipeline produces real statistical charts from NOAA station data. Boxplots, heatmaps, radar charts and correlation plots all live here-- proving that the scoring rules hold up in practice, not just on paper.

<p align="center">
  <img src="docs/images/3.png" width="92%" alt="R analysis charts and dashboards" />
</p>

---

<br>

## How It Works

The engine takes five inputs, the five climate signals that most directly determine whether solar panels, wind turbines or hydro infrastructure will produce usable energy in a given place. It scores Solar, Wind and Hydro independently against explicit thresholds. The highest scorer wins. The gap between first and second determines how confident the recommendation really is.

```mermaid
flowchart LR
    Inputs["Temperature · Wind · Rain · Elevation · Snow"]
    Inputs --> Engine["Scoring Engine"]
    Engine --> Solar["Solar  /9"]
    Engine --> Wind["Wind   /7"]
    Engine --> Hydro["Hydro  /8"]
    Solar & Wind & Hydro --> Result["Recommendation and Confidence"]
    Result --> App["Web App"]
    Inputs --> Analysis["R Analysis Pipeline"]
    Analysis --> Stats["ANOVA · Regression · Feature Importance"]
```

Nothing is hidden in this diagram. The same logic that runs in the browser also runs in the R batch pipeline, applied to thousands of real NOAA weather station records, validated statistically and exported for inspection.

### The Scoring Rules

Every threshold here comes from domain research in climate science and renewable energy engineering. Temperature above 15°C meaningfully extends solar production hours. Wind speed below 4 m/s means a turbine rarely reaches its operating range. Precipitation above 100 mm a month means a catchment has real water to work with. These are not arbitrary cutoffs. They are the lines where viability actually changes.

| | ☀️ Solar | 🌬️ Wind | 💧 Hydro |
|:---|:---|:---|:---|
| **Max score** | 9 | 7 | 8 |
| **Key triggers** | Temp > 15°C, Precip < 50mm, Snow < 10cm | Speed ≥ 4 m/s, Elev 500 to 2000m | Precip > 100mm, Snow > 20cm, Elev 300 to 2000m |
| **Penalties** | Precip > 150mm loses 2 points | Weak wind, flat terrain | Dry climate, no elevation |
| **Best climate** | Warm, dry, clear sky | Exposed ridgelines and coasts | High rainfall, mountain catchments |

### Confidence Tiers

A recommendation without a confidence measure is incomplete. Two climates can both recommend Solar, one because it scores 8 out of 9 and another because Solar scored 5 and Wind scored 4. Those are very different situations. The margin tells you which one you are in.

| Margin between 1st and 2nd | Confidence | What It Means |
|:---:|:---:|:---|
| 6 or more points | 🟢 High | The climate strongly favours one source |
| 3 to 5 points | 🟡 Moderate | Clear winner but the runner-up is worth noting |
| 0 to 2 points | 🔴 Low | The climate is genuinely split and a hybrid approach makes sense |

<br>

## Real Climates, Real Results

These five presets show the range of what the engine produces. Notice that the Mountain case returns *low confidence* on Wind, not because the recommendation is wrong, but because Hydro is close behind. That honesty is the point.

| Climate | Temp | Wind | Precip | Elev | Snow | Result |
|:---|:---:|:---:|:---:|:---:|:---:|:---:|
| 🏜️ Desert | 35°C | 4 m/s | 10 mm | 300 m | 0 cm | ☀️ Solar-- High |
| 🏔️ Mountain | 5°C | 7 m/s | 120 mm | 2000 m | 30 cm | 🌬️ Wind-- Low |
| 🌧️ Monsoon | 26°C | 5 m/s | 200 mm | 500 m | 0 cm | 💧 Hydro-- High |
| 🌊 Coastal | 18°C | 8 m/s | 70 mm | 50 m | 0 cm | 🌬️ Wind-- High |
| 🌿 Temperate | 12°C | 3 m/s | 90 mm | 400 m | 5 cm | 💧 Hydro-- Mod |

<br>

## The Design Choice That Shaped Everything

Early in the project, the obvious path was to train a classifier. Collect labelled climate energy data, pick a model, tune it, deploy it. That approach has real strengths-- it can capture nonlinear patterns, handle edge cases gracefully and it looks impressive in a portfolio.

But it also has a problem. When it tells a homeowner that Solar is their best option, they cannot see why. They cannot ask what would change the answer. They cannot point a domain expert at the reasoning and ask whether it is sound.

For this problem, where the physics are well understood, where labelled ground truth data is scarce and where the user *trusting* the output matters as much as the output being correct, a rule-based system is the better tool.

| | Rule-Based | ML Classifier |
|:---|:---:|:---:|
| Logic is visible | ✅ | ❌ |
| Needs labeled training data | ❌ | ✅ |
| Users can challenge the output | ✅ | Rarely |
| Confidence is easy to communicate | ✅ | Extra work |
| Works where domain knowledge is strong | ✅ Best fit | Not ideal |

> Interpretability is not a feature here. It is the product.

<br>

## The R Layer: Where the Rules Get Tested

The web app makes the tool accessible. The R analysis suite makes it credible.

After building the scoring engine, the natural question is whether these rules actually separate climates the way the physics says they should. To answer that, the project pulls real station data from NOAA's Global Historical Climatology Network, runs the classifier in batch mode across thousands of records and validates the results statistically.

```mermaid
flowchart LR
    NOAA["NOAA Station Data"] --> Fetch["Fetch and Clean\ndata_fetch.R"]
    Fetch --> Classify["Batch Scoring\nclassify_energy.R"]
    Classify --> Validate["Statistical Tests\nanalysis.R"]
    Classify --> Charts["Charts and Plots\nvisualizations.R"]
```

Each module has a specific job. `data_fetch.R` handles the messy reality of working with a public API, including pagination, unit conversion, retry logic and regional bounding boxes. `classify_energy.R` runs the same scoring logic from the browser but at scale. `analysis.R` is where the rules are put under pressure.

| Module | Purpose |
|:---|:---|
| `classify_energy.R` | Batch scoring, confidence logic and export |
| `analysis.R` | ANOVA, correlation, regression and feature importance |
| `visualizations.R` | Boxplots, violins, heatmaps, scatter maps and radar charts |
| `data_fetch.R` | NOAA API pagination, caching and unit conversion |
| `utils.R` | Validation, climate zones, daylight helpers and logging |

### What the Data Said

The results validated the design. ANOVA confirmed strong separation between the climate profiles of Solar, Wind and Hydro classes. The three groups look genuinely different in the data, not just in theory. Temperature and precipitation were the strongest separators, which matches the scoring weights. Wind speed behaved more independently, which explains why wind recommendations appear across a wider spread of climate types than Solar or Hydro ever do.

Permutation feature importance gave a clear ordering: `Temperature > Precipitation > Wind Speed > Elevation > Snow Depth`. The engine's scoring priorities matched what the data independently ranked as most discriminating. That agreement between the rule design and the statistical validation is the result that matters most.

<br>

## Running the Project

The web app needs nothing. Just open it in any browser.

```
open https://weather-energy.netlify.app/
```

To run the full R analysis pipeline locally:

```r
install.packages(c("jsonlite", "ggplot2", "tidyr", "dplyr", "readr", "purrr"))

source("classify_energy.R")
source("analysis.R")
source("visualizations.R")
```

To fetch real NOAA station data for any region, add your free CDO API token to `data_fetch.R` and then call:

```r
fetch_region_data(bbox = c(lat_min, lon_min, lat_max, lon_max))
```

<br>

## Project Structure

```
Weather-Classification/
├── index.html             the web app, open and run
├── classify_energy.R      core scoring engine
├── analysis.R             statistical validation
├── visualizations.R       charts and dashboards
├── data_fetch.R           NOAA API integration
├── utils.R                shared helpers
├── data.csv               sample dataset
└── docs/images/           screenshots, gif, hero image
```

<br>

## What This Project Is, at Root

It is a tool for a specific gap: the space between vague climate intuition and a decision someone can actually act on. It is also a methodological argument, that in domains where the science is understood and labels are hard to come by, a transparent rule-based system earns more trust than a model that performs better on paper but cannot explain itself.

The web app makes that argument accessible. The R pipeline makes it defensible. Together they demonstrate something that matters beyond this particular problem: **a model you can read is more useful than a model you can only believe.**

<br>

<div align="center">

**Sahibjeet Pal Singh**

[GitHub](https://github.com/Sahibjeetpalsingh) · [Live App](https://weather-energy.netlify.app/) · [LinkedIn](https://linkedin.com/in/sahibjeet-pal-singh-418824333)

</div>
