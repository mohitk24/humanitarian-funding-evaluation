<div align="center">

# 🌍 United Nations Humanitarian Funding Evaluation

### A transparent, data-driven framework to benchmark aid efficiency, detect cost anomalies, and map global donor contributions across UN humanitarian funding mechanisms

![Python](https://img.shields.io/badge/Python-3.11+-3776AB?logo=python&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-Data%20Wrangling-150458?logo=pandas&logoColor=white)
![NumPy](https://img.shields.io/badge/NumPy-Statistical%20Analysis-013243?logo=numpy&logoColor=white)
![Plotly](https://img.shields.io/badge/Plotly-Interactive%20Viz-3F4F75?logo=plotly&logoColor=white)
![Seaborn](https://img.shields.io/badge/Seaborn-Statistical%20Plots-4C72B0)
![Databricks](https://img.shields.io/badge/Azure%20Databricks-Cloud%20Platform-FF3621?logo=databricks&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)

**Co-authors: Aravindhan Anbazhagan**

▶️ [Watch the Presentation](https://www.youtube.com/watch?v=WkQN94EvdBY)

[Overview](#-project-overview) · [How It Works](#-analytical-framework--how-it-works) · [Key Features](#-key-features) · [Structure](#-project-structure) · [Visuals](#-visual-insights--sample-artifacts) · [Use Cases](#-industry--humanitarian-use-cases)

</div>

---

## 📖 Overview

Global humanitarian funding mechanisms - including the UN Central Emergency Response Fund (CERF) and Country-Based Pooled Funds (CBPF) - disburse billions of dollars annually across disaster relief, health response, food security, and protection clusters. Yet comparing operational efficiency across diverse regions, funding windows, and allocation categories remains a persistent challenge for auditors, donors, and policy teams alike.

This project proposes a **context-aware cost efficiency framework** built on five core UN datasets. Rather than treating cost figures in isolation, the pipeline benchmarks each project **within its own humanitarian cluster**, computes log-scaled cost-per-beneficiary (CPB) KPIs, applies IQR-based statistical outlier detection to surface anomalies, and integrates geospatial donor mapping for a full-picture view of aid delivery.

The entire pipeline runs on **Azure Databricks** using Python, with LLM-assisted code generation (Cursor) used for accelerated development - a workflow explicitly recommended by Databricks during the event.

> 💡 **Context:** This project was built for the **CMU Datathon**, a time-constrained analytical competition focused on real-world UN humanitarian data.

---

## 🧠 How It Works

### Pipeline Overview

### Step 1 · Data Ingestion & Wrangling

Five raw CSV datasets are loaded and cleaned through an **ETL pipeline** built in Pandas:

- Currency strings (e.g., `" 1,087,230.72 "`) are stripped and coerced to float via a shared `parse_currency()` helper.
- Demographic beneficiary columns (`Men`, `Women`, `Boys`, `Girls`, `IndirectBeneficiaries`) are coerced to numeric and aggregated into `direct_beneficiaries` and `total_beneficiaries`.
- A configurable denominator (`DENOM`) lets analysts switch between direct, total, or catchment population for CPB calculation.
- Rows with missing or zero cost/beneficiary values are dropped, keeping only analyzable project records - resulting in a clean working set of **9,141 CBPF projects**.

### Step 2 · Feature Engineering & Benchmarking

Two derived columns power all downstream analysis:

| Derived Feature | Formula | Purpose |
|---|---|---|
| `cost_spent` | `TotalDirectCost + TotalSupportCost` | Actual implementation cost per project |
| `cost_per_beneficiary` | `cost_spent / beneficiaries_used` | Core efficiency KPI |
| `log_cpb` | `log₁₀(cost_per_beneficiary)` | Logarithmic transformation for right-skewed normalization |

Log-transformation (`log₁₀`) is applied to compress the extreme right skew in CPB distributions, enabling meaningful group-level statistical comparisons and fair **benchmarking** across 8 allocation categories.

### Step 3 · Statistical Outlier Detection

Group-wise **IQR-based outlier flagging** (`k = 1.5`) is applied on `log_cpb` within each `AllocationTypeCategory` cluster:

```python
q1 = grp.transform(lambda s: s.quantile(0.25))
q3 = grp.transform(lambda s: s.quantile(0.75))
iqr = q3 - q1
is_outlier = (log_cpb < q1 - 1.5 * iqr) | (log_cpb > q3 + 1.5 * iqr)
```

This approach benchmarks each project **against peers in the same allocation type**, not against the global distribution - avoiding false flags caused by structural cost differences between, say, Frontline Services and HFU Management.

**Result:** **77 high-cost projects** flagged across 9,141 analyzed records (~3.6% outlier rate).

### Step 4 · Regional Risk Profiling

Regional outlier rates and median CPB are computed per `PooledFundName` (e.g., Ukraine, oPt, Syria, DRC). A minimum threshold of 20 projects per region is enforced to avoid unstable rates from small samples.

**Notable finding:** Ukraine showed the highest outlier rate at **23.8%** and highest median CPB at **$147**, consistent with conflict-zone operational cost premiums.

### Step 5 · Geospatial Donor Mapping

CERF donor country names are resolved to **ISO-3 codes** using `pycountry`, with a manual fix dictionary for common naming mismatches (e.g., "United States" → "USA", "Viet Nam" → "VNM"). A `log₁₀`-scaled choropleth map is generated via Plotly Express, surfacing global contribution patterns across all donor countries.

---

## ⚡ Key Features

| | Feature | Detail |
|---|---|---|
| 📊 | **CPB Distribution Analysis** | Histogram of `log₁₀(CPB)` with mean/median markers and RdYlGn gradient shading across 2,160 projects |
| 📦 | **Allocation Category Boxplots** | Side-by-side box plots comparing CPB spread across 8 `AllocationTypeCategory` groups |
| 🔍 | **Outlier Scatter Plot** | Log-log implementation cost vs. beneficiaries with 77 flagged outliers and a ±30% efficiency band |
| 🏆 | **Top-10 Category CPB Ranking** | Horizontal bar chart of median CPB by category with IQR error bars on a log scale |
| 🗺️ | **Regional Risk Profile** | Dual bar chart of outlier rate and median CPB by pooled fund region |
| 🌍 | **Interactive Donor Map** | Plotly choropleth of CERF donor contributions by country (HTML, fully interactive) |
| 🍩 | **CERF Allocation Donut** | Sector breakdown of $9.92B in CERF allocations across top 10 project cluster categories |
| 🧮 | **UN Efficiency Review Framework** | Styled summary table of outlier rate, median CPB, total budget, and beneficiaries per cluster |

---

## 📁 Project Structure

```
UN Humanitarian Funding Evaluation/
├── app.py                                    # Full analysis pipeline (ETL → EDA → Outlier Detection → Visualization)
├── requirements.txt                          # Pinned Python dependencies
├── data/
│   ├── CBPF Projects.csv                     # 9,141 project-level records (cost, beneficiaries, allocation type)
│   ├── CBPF Contributions.csv                # Donor contributions to country-based pooled funds
│   ├── CERF Donor contributions.csv          # CERF donor received amounts by country and year
│   ├── CERF Donor allocations.csv            # Project-level CERF allocations with cluster tags
│   └── CERF Climate Related Allocations.csv  # Climate-related allocation summaries (2006–2021)
└── output/
    ├── 01_Distribution_Cost_per_Beneficiary.png
    ├── 02_Boxplot_Cost_per_Beneficiary.png
    ├── 03_Scatterplot_Cost_vs_Beneficiaries.png
    ├── 04_Top_10_Categories_by_Median_CPB.png
    ├── 05_Outlier_Rate_by_Region.png
    ├── 06_Donor_Map.html
    ├── 07_Top_Donors.png
    └── 08_CERF_Allocations.png
```

| File | Role |
|---|---|
| `app.py` | End-to-end pipeline: data loading, currency parsing, beneficiary aggregation, CPB computation, IQR outlier detection, cluster summary tables, and all 8 chart exports |
| `requirements.txt` | Pinned dependencies for reproducible environment setup |
| `CBPF Projects.csv` | Primary dataset: project titles, organizations, pooled fund names, allocation types, budgets, direct/support costs, and beneficiary demographics |
| `CERF Donor contributions.csv` | Donor-country level records used for geospatial choropleth mapping |
| `CERF Donor allocations.csv` | CERF project allocations used for cluster-level donut chart and sector analysis |
| `06_Donor_Map.html` | Fully interactive Plotly choropleth - open in any browser, hover for country-level totals |

---

## 📊 Visual Insights & Sample Artifacts

### Distribution of Cost per Beneficiary
`01_Distribution_Cost_per_Beneficiary.png`

Log-scaled histogram across 2,160 projects (mean: $1.30 log₁₀, median: $1.23 log₁₀, std dev: 0.67). The right tail confirms significant skew - a small share of projects account for extreme per-beneficiary costs.

### Outlier Detection — Cost vs. Beneficiaries
`03_Scatterplot_Cost_vs_Beneficiaries.png`

Log-log scatter of implementation cost against total beneficiaries. 77 flagged outliers (shown in red) fall well outside the ±30% efficiency band around the trend line. Normal projects (n=2,083) cluster tightly along the expected scaling relationship.

### Regional Outlier Rates
`05_Outlier_Rate_by_Region.png`

Ukraine leads at **23.8% outlier rate** and **$147 median CPB**, followed by oPt (12.0%, $125). Sudan and Ethiopia sit at $12 median CPB - highlighting the structural cost gap between conflict-zone and stable-country operations.

### Interactive Donor Map
`06_Donor_Map.html`

Choropleth of CERF `donorreceived` totals mapped to ISO-3 country codes via `pycountry`. Top donors: United Kingdom ($1.97B), Sweden ($1.31B), Netherlands ($1.30B), Germany ($1.24B), Norway ($1.07B).

### CERF Allocation Breakdown
`08_CERF_Allocations.png`

$9.92B in total CERF allocations across top 10 project clusters. Food Assistance leads at 17.6%, followed by Health (13.4%) and Agriculture (8.2%).

> ⚠️ **Note on outlier interpretation:** A flagged project does not indicate misuse of funds. High CPB can reflect legitimate factors such as remote access, conflict surcharges, or small-scale emergency response. This framework surfaces candidates for further review, not conclusions.

---

## 💡 Industry & Humanitarian Use Cases

| Use Case | How This Framework Helps |
|---|---|
| 🔍 **Aid Transparency & Auditing** | Automated flagging of anomalous CPB spikes gives UN auditors a prioritized shortlist of projects warranting deeper review — without manually scanning thousands of rows |
| 💰 **Donor Impact Tracking** | Geospatial choropleth and top-donor bar charts give contributing governments and foundations a clear view of where funds originate and how efficiently they reach beneficiaries |
| ⚖️ **Resource Allocation Optimization** | Cluster-level benchmarking enables fund managers to compare operational costs across structurally similar categories, supporting evidence-based reallocation decisions |
| 📋 **Operational Risk Profiling** | Regional outlier rate tables identify funds operating in high-risk, high-cost environments (e.g., Ukraine, oPt), useful for cost-escalation planning and country strategy reviews |
| 📡 **Scalable Pipeline Template** | The ETL pipeline and IQR outlier engine are dataset-agnostic — the same approach applies to any disbursement dataset with cost and beneficiary columns |

---

## 📜 License & Acknowledgements

Released under the [MIT License](LICENSE).

**Data Sources**
- [UN OCHA CBPF Data](https://cbpfgms.github.io/) - Country-Based Pooled Fund project and contribution records
- [CERF Open Data](https://cerf.un.org/what-we-do/allocation-data) - Central Emergency Response Fund donor and allocation datasets

**Acknowledgements**
- Built during the **CMU Datathon**, with Azure Databricks as the development and execution platform.
- LLM-assisted development workflow (Cursor) used for code generation, as recommended by Databricks.

Built with [Pandas](https://pandas.pydata.org/) · [NumPy](https://numpy.org/) · [Plotly](https://plotly.com/python/) · [Seaborn](https://seaborn.pydata.org/) · [Matplotlib](https://matplotlib.org/) · [pycountry](https://pypi.org/project/pycountry/) · [Azure Databricks](https://azure.microsoft.com/en-us/products/databricks)
