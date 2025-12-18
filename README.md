This `README.md` is designed as a technical guide for your bar’s (beer) inventory system. It documents the "Whole Algorithm" I built, specifically tuned for the volatile Metro Manila market.

# 🍺 Bar Inventory Logic: The "Zero-Ubos" Algorithm

This doc breaks down how to figure out how much beer to buy each day—no more guessing! Instead of relying on gut feel, I use a simple weighted forecast and a safety buffer to keep you stocked (but not overstocked).

# ❌ The Cost of having Emergency buys

Core idea: emergency buys from a sari‑sari shop inflate purchase cost, cut margin, and/or cause lost sales and unhappy customers.
Quick numeric example (assumptions: case cost from supplier ₱300, retail price per case ₱600, emergency sari‑sari cost per case ₱450):

- Normal night (no OOS): buy 10 cases × ₱300 = ₱3,000 cost → revenue 10×₱600 = ₱6,000 → gross profit = ₱3,000.
- Emergency buy from sari‑sari: buy 10 cases × ₱450 = ₱4,500 cost → revenue still ₱6,000 → gross profit = ₱1,500 → lost profit = ₱1,500 (50% reduction).
- If you can’t source and lose sales (no restock): lost revenue = ₱6,000 and lost gross profit = ₱3,000, plus reputational/customer churn risk.

Other direct & indirect costs:

- Per‑unit premium (markup) and extra delivery/time cost.
- Opportunity cost: lost repeat customers, bad reviews, lower future nights.
- Operational hassle: staff scrambling, manual purchases, possible price arbitration.

Bottom line: even a small emergency premium quickly erodes profit; safety stock that costs a few extra cases is usually cheaper than repeated emergency buys or lost nights.

## 🧮 The Core Formula

To determine the number of cases to buy today (Q), we use:

Where:

* **D_{next}**: Predicted base demand.
* **M**: Manila Event Modifier (Multi-factor).
* **SS**: Safety Stock (Buffer).
* **I**: Current Inventory (On-hand count).

---

## 📋 The Algorithm

### 1. Base Forecast (D_)

We use a **Weighted Moving Average**—basically, we care more about what happened last week on the same day than what happened yesterday. Manila bars have strong "Friday is Friday" patterns, so last Friday’s sales matter more than just yesterday’s.

* **Formula:** `(Yesterday_Sales * 0.3) + (Same_Day_Last_Week_Sales * 0.7)`
* **Rationale:** Think of the weights like shares of trust: 0.3 means “30% trust yesterday,” 0.7 means “70% trust last week.” Adding to 1 means you’re just splitting 100% of your trust between the two signals.

**Practical rule-of-thumb for choosing weights:**

* Default: start with 0.3 (yesterday) / 0.7 (same day last week).
* If strong day-of-week pattern (Fridays always high): push last-week weight up (0.6–0.9).
* If you see a clear recent trend (3-day rolling mean changes > ~10%): raise yesterday’s weight (0.4–0.6).
* Always normalize so weights sum to 1 (keeps units sane).

### 2. The Manila Modifier (M)

This adjusts the forecast based on external local factors.
Think of M as a simple knob you twist up when you expect more customers and turn down for slow or stormy days.

- Default = 1.0 (no change).
- Bump it when you expect crowds: Payday/holiday → 1.3–1.5; big event → ~1.2.
- Lower it for bad weather or typhoons: 0.6–0.8.
- Quick rule: estimate the percent change in customers and set M = 1 + percent_change (e.g., +20% → 1.2).
- Calibrate over time: compare actual vs predicted after events and nudge M up/down for future similar days.
- When unsure, prefer a small uplift for busy nights rather than undershooting.

### 3. Safety Stock (SS)

This is the extra buffer so a surprise rush or a delayed delivery doesn’t leave you out-of-stock (OOS).

* **Formula:** `Z * σ_d * sqrt(L)`
* **Quick default:** `1.65 * STDEV(Last 7 days) * 1` (so SS ≈ 1.65 × STDEV)
* **What the bits mean:**

  * Z — service-level factor (z‑score for your desired probability of avoiding a stockout).
  * σ_d — standard deviation of daily sales (use STDEV of last 7 days as a quick estimate).
  * L — lead time in days (usually 1 for daily ordering; use sqrt(2) if lead time is 2 days).

  Quick numeric example (σ_d = 4, L = 1):

  - 95%: SS = 1.65 × 4 × 1 = 6.6 → round up to 7 cases.
  - 90%: SS = 1.28 × 4 × 1 = 5.12 → round up to 6 cases.
  - 85%: SS = 1.04 × 4 × 1 = 4.16 → round up to 5 cases.
  - 80%: SS = 0.84 × 4 × 1 = 3.36 → round up to 4 cases.
* **Tiny example:** STDEV = 4 → SS ≈ 1.65×4 = 6.6 → round up to 7 cases.

**No historical sales? — quick safe fallback**

1. Use proxies: Ask waiters or supplier reps for a typical nightly sell-through.
2. Make a conservative estimate of peak nightly cases (P). If totally unsure, assume P = 10–20 cases for a small bar.
3. Set Safety Stock = P * LeadTime (L) as a minimum buffer, or use a conservative variance:
   - Estimate mean μ ≈ P. Set σ ≈ max(0.25·μ, sqrt(μ)).
   - Then SS = 1.65 * σ * sqrt(L).
4. Operational rule: start conservative (higher SS), run a 7–14 day observational period, then replace estimates with real STDEV/mean and lower SS as confidence grows.
5. Also use supplier minimums and storage limits: if supplier min order > recommended, treat extra as temporary buffer and track consumption.

---

## 📊 Sample Calculation

**Scenario:** It is **Friday morning**. We are ordering for the Friday night shift.

| Variable                      | Value    | Description                     |
| ----------------------------- | -------- | ------------------------------- |
| Last Friday Sales             | 30 cases | D_{t-7}                         |
| Yesterday (Thursday) Sales    | 18 cases | D_{t-1}                         |
| Standard Deviation (\sigma_d) | 4 cases  | Daily sales fluctuation         |
| Current Inventory (I)         | 8 cases  | Counted in chiller this morning |
| Modifier (M)                  | 1.2      | Payday weekend adjustment       |

### Calculations:

1. **Base Forecast:** (18 \cdot 0.3) + (30 \cdot 0.7) = \mathbf{26.4}
2. **Adjusted Demand:** 26.4 \cdot 1.2 = \mathbf{31.68}
3. **Safety Stock:** 1.65 \cdot 4 \cdot 1 = \mathbf{6.6}
4. **Total Required:** 31.68 + 6.6 = \mathbf{38.28}
5. **Final Order (Q):** 38.28 - 8 = \mathbf{30.28}

**✅ ACTION:** Order **30 Cases**.

---

## 📈 Implementation in Excel/Google Sheets

To automate this, use the following cell logic:

```excel
=ROUNDUP(((Yesterday*0.3)+(LastWeek*0.7)*Modifier) + (STDEV(Last7Days)*1.65) - CurrentStock, 0)

```

## 🌐 Using the Web App

The web app follows the steps in "The Algorithm". Provide the same inputs the algorithm expects, with one simple exception for Safety Stock: instead of asking you to compute the standard deviation of the last 7 days, the app asks for the average sales over the last 7 days and estimates the standard deviation automatically.

- Inputs the app asks for:
  - Yesterday sales
  - Same day last week sales
  - Manila modifier (`M`)
  - Average sales (last 7 days) — enter the 7-day average (no STDEV required)
  - Current inventory (`I`)
  - Lead time in days (`L`)
  - Desired service level (defaults to 95%)
    - UI helpers: click the `?` icon next to any input to see a short explanation, examples, and recommended defaults for that field.
    - Sample `?` tooltip text:
      - **Yesterday sales:** "Enter the number of cases sold yesterday. Example: 18. Use 0 only for forced/full-day closures (e.g., typhoon or mandated closure). For private-event closures that stopped public sales, exclude that day from averages."
      - **Same day last week sales:** "Enter cases sold on the same weekday last week. Example: last Friday = 30. Helps capture weekly patterns."
      - **Manila modifier (M):** "Multiplier for local events/weather. Default 1.0. Use 1.2 for +20% expected crowd, 0.8 for bad weather."
      - **Average sales (last 7 days):** "Enter the average daily cases sold over the past 7 days. Example: 22. The app estimates variability from this."
      - **Current inventory (I):** "Current on-hand cases in the chiller right now. Count physically for accuracy."
      - **Lead time (L):** "Days between ordering and receiving stock. Default 1 for daily ordering; use 2 if delivery may be delayed."
      - **Desired service level:** "Target probability of avoiding stockouts. Default 95% (Z ≈ 1.65). The app shows a small reference table so you can preview Safety Stock for common levels (e.g., 80%–97.5%). Quick guidance: choose based on how painful a stockout would be vs. how costly extra cases are; start at 95% and adjust after you see real results."

**Desired service level**

Target probability of avoiding stockouts. Defaults to 95% (Z ≈ 1.65). Lower values reduce Safety Stock (SS) but increase the chance you’ll run out during a rush.

The app surface includes a small reference table in the ordering UI that shows several common service levels, the corresponding chance of stockout, and the resulting Safety Stock for a sample σ_d — so you can preview the impact of different choices before committing. Use the interactive table to help choose a service level that balances cost vs. risk.

In plain language: the service level is the chance you’ll have enough beer on hand when customers show up. Higher service levels mean carrying more buffer cases (higher cost, lower risk of lost sales). Lower service levels save inventory cost but raise the risk of emergency buys or lost revenue. Use 95% as a sensible default; pick lower if you can tolerate occasional stockouts or have very cheap/fast resupply, and higher for critical nights (big events, paydays).

Practical note on the numbers: SS = Z × σ_d × sqrt(L). With σ_d = 4 and L = 1, SS ≈ Z × 4. We round up to whole cases.

  Sample "Service level" reference table (shown in the app for quick preview, σ_d = 4)

  | Desired Service level | Chance of Stockout | Safety Stock |
  | --------------------- | ------------------ | ------------ |
  | 80%                   | 20%                | 4            |
  | 85%                   | 15%                | 5            |
  | 90%                   | 10%                | 6            |
  | 95%                   | 5%                 | 7            |
  | 97.5%                 | 2.5%               | 8            |

  

- How the app computes Safety Stock from the average:

  - The app estimates the daily-sales standard deviation using a simple heuristic: `σ_est = max(0.25 * avg, sqrt(avg))`, where `avg` is the 7-day average you entered.
  - It then computes Safety Stock as `SS = Z * σ_est * sqrt(L)` (default `Z = 1.65` for ~95% service level).
- Full computation flow the app performs:

  1. Base forecast = `(Yesterday * 0.3) + (LastWeek * 0.7)`
  2. Adjusted demand = `Base forecast * M`
  3. Safety stock = `Z * σ_est * sqrt(L)` using the average-based `σ_est`
  4. Total required = `Adjusted demand + Safety stock`
  5. Final order `Q` = `ROUNDUP(Total required - Current inventory, 0)` (floors at 0)

Notes:

- The average-based σ heuristic simplifies data entry; if you later have the actual 7-day STDEV you can override `σ_est` in the app for a more accurate Safety Stock.
- All other operational rules (rounding, lead-time adjustments, event modifiers, and not counting exceptional private-event zeros) remain the same as in "The Algorithm" section.

## ⚠️ Notes on Local Constraints

* **Coding/Truck Ban:** If your supplier is affected by UVVRP (Coding) on a delivery day, manually change Lead Time (L) to **2** for the day prior.
* **Data Cleaning:** If you had a private event that closed the bar to public sales, exclude that day's "0" from your averages. Use `0` only for forced/full-day closures (e.g., typhoons or mandated shutdowns) that genuinely reflect zero demand.

---

## Technologies Used

- **Vanilla JS:** Lightweight front-end logic written without frameworks for fast, dependency-free UI behavior.
- **Simple Statistics:** The `simple-statistics` library is used for statistical helpers (mean, standard deviation, z-scores) in the forecasting logic.
- **HTML5:** Semantic markup and form controls power the ordering UI and make the app accessible across devices.
- **Materialize CSS:** UI styling, tables, and components are provided by Materialize for a clean, responsive look and faster prototyping. 
- **SessionStorage:** Browser `sessionStorage` is used to persist user inputs (recent averages, modifiers, and inventory) during a session so users don't lose entries when navigating the app.

