# 🛒 Superstore Sales Analytics

From raw transactions to a profit-prediction model and an executive dashboard — a four-phase retail analytics project covering dimensional modelling, statistical validation, gradient-boosted regression, and business intelligence.

**Core question:** what actually drives profit in a retail operation — and can it be predicted per transaction?

**Result:** an XGBoost regressor explains **73% of profit variance** on unseen data (R² = 0.73, MAE = 37.13), with a train–test gap of just 0.05 — tuned to generalise rather than to score well on training data.

---

## 📐 Phase 1 — Dimensional modelling & data preparation

The raw transaction extract was cleaned in **Power Query** and restructured into a **star schema** rather than left as a flat table:

| Table | Role |
| --- | --- |
| `Dim_Fact` | Fact table — sales, quantity, discount, profit, shipping cost, return flag |
| `Dim_Customer` | Customer name, segment |
| `Dim_Product` | Brand, name, feature, category, sub-category |
| `Dim_Geography` | City, state, country, region, latitude, longitude |
| `Dim_Date` | Order and shipping date parts |
| `Dim_Shipping` | Shipping mode |
| `Dim_Market` | Marketplace |
| `Dim_OrderPriority` | Priority level |

Splitting the descriptive attributes out of the fact table is what makes the Power BI model in Phase 4 filter cleanly across dimensions, and what keeps the feature engineering in Phase 3 from re-deriving the same joins repeatedly.

**Data quality:** 0 duplicate rows. Nulls confined to `Product_Name` (232) and `Product_Feature` (7,590) — both descriptive fields excluded from modelling.

---

## 📊 Phase 2 — Statistical analysis

Business assumptions were tested before anything was modelled. Each variable was screened with **Shapiro-Wilk** first; normality failed throughout, so non-parametric tests were used rather than defaulting to t-tests and ANOVA.

| Question | Test | Result |
| --- | --- | --- |
| Do discounts meaningfully affect the business? | Mann-Whitney U | **Significant** (p < 0.05) for quantity, sales, profit *and* shipping cost |
| Does season affect units sold? | Kruskal-Wallis | **Significant** — H = 14.62, p = 0.0022 |
| Which numeric drivers move with sales? | Spearman rank correlation | Monotonic relationships identified without assuming linearity |
| Are the categorical attributes independent? | Chi-square | Association tested across contingency tables |

The discount finding is the commercially interesting one: discounting shifts quantity *and* profit distributions, so it is not a free lever — it changes the profit profile of the transactions it touches.

---

## 🤖 Phase 3 — Profit prediction

A regression problem: predict `Profit` per transaction.

### Feature engineering

Thirteen numeric features, several constructed rather than raw:

- `Lag_Time_Days` — shipping date minus order date
- `Discount_to_Sales_Ratio`, `Discount_per_Unit`, `Discount_x_Quantity`
- `Net_Sales_After_Discount`, `Sales_per_Unit`, `Sales_x_Quantity`
- `Shipping_Cost_per_Unit`
- `Avg_Sales_per_Product` — per-product group mean
- `Order_Season` → `PeakSeason_x_Discount` interaction term

Plus four label-encoded categoricals: category, sub-category, marketplace, priority level.

The interaction terms exist because Phase 2 established that discount and season both matter independently — the features encode that finding rather than hoping the model rediscovers it.

### Models

All four tuned with `GridSearchCV` (5-fold, scoring on negative MSE), evaluated on a held-out 20% test split:

| Model | Test R² | Train R² | Overfit gap | MAE | MSE |
| --- | --- | --- | --- | --- | --- |
| **XGBoost** | **0.73** | 0.78 | 0.05 | 37.13 | 8,889.80 |
| Gradient Boosting | 0.73 | 0.79 | 0.06 | 39.05 | 9,115.62 |
| Random Forest | 0.72 | 0.76 | **0.04** | **36.60** | 9,420.84 |
| Decision Tree | 0.68 | 0.74 | 0.05 | 38.74 | 10,645.24 |

**Overfit gap is reported explicitly for every model.** All four stay at or below 0.06, meaning the tuning targeted generalisation, not leaderboard R². XGBoost and Gradient Boosting tie on test R²; Random Forest has the lowest error in absolute terms and the tightest gap.

---

## 📈 Phase 4 — Power BI dashboard

An executive dashboard built on the Phase 1 star schema, tracking:

- Yearly growth trends
- Profitability by region and market
- Customer segmentation
- Category and sub-category performance

📄 **[View the dashboard as PDF](Phase1&4_PBI/Superstore_dashboard.pdf)** — no Power BI licence needed.

The `.pbix` source file is included for anyone who wants to open the live model.

---

## 🛠️ Tech stack

| Layer | Tools |
| --- | --- |
| Data preparation | Power Query, `pandas` |
| Statistics | `SciPy` — Shapiro-Wilk, Mann-Whitney U, Kruskal-Wallis, Spearman, chi-square |
| Machine learning | `scikit-learn`, `XGBoost` — GridSearchCV, 5-fold CV |
| Visualisation | Power BI, `matplotlib`, `seaborn` |

---

## 📂 Repository structure

```text
├── Phase1&4_PBI/
│   ├── Superstore_dashboard.pbix   # Power BI model + report
│   ├── Superstore_dashboard.pdf    # Static export — view without Power BI
│   ├── Structured_data.png         # Star schema diagram
│   └── Unstructured_data.png       # Pre-modelling state
├── Phase2_Statistic/
│   ├── Statistics.ipynb            # Hypothesis testing
│   └── Online_Store.xlsx           # Modelled dataset
├── Phase3_ML/
│   └── Profit_prediction.ipynb     # Feature engineering + 4 tuned regressors
└── README.md
```

---

## 🚀 Running it

```bash
git clone https://github.com/SareMahdavifar/Superstore_sales_analytics.git
cd Superstore_sales_analytics
pip install -r requirements.txt
jupyter notebook
```

The notebooks read `Online_Store.xlsx` from their own directory. If you relocate the file, update the `pd.read_excel` path at the top of each notebook.

---

## ⚠️ Limitations

- **R² = 0.73 is transaction-level, not forecast-level.** The model predicts profit for a transaction whose sales, quantity and discount are already known — it is a profit *decomposition*, not a demand forecast, and cannot be used to project future periods.
- **Feature leakage risk is real here.** `Net_Sales_After_Discount` and `Sales_x_Quantity` are arithmetically close to the profit definition. The strong R² partly reflects that closeness rather than learned business structure.
- **Label encoding imposes false ordinality** on nominal categories like marketplace and sub-category. Tree models tolerate this better than linear ones, but one-hot or target encoding would be cleaner.
- **No SQL scripts are included.** The dimensional model was built in Power Query and Excel; a MySQL implementation is a natural next step, not something this repo currently demonstrates.

---

## 📄 License

Released under the [MIT License](LICENSE).
