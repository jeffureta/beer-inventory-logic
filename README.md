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

### 1. Base Forecast (D_{next})

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

| Variable | Value | Description |
| --- | --- | --- |
| Last Friday Sales | 30 cases | D_{t-7} |
| Yesterday (Thursday) Sales | 18 cases | D_{t-1} |
| Standard Deviation (\sigma_d) | 4 cases | Daily sales fluctuation |
| Current Inventory (I) | 8 cases | Counted in chiller this morning |
| Modifier (M) | 1.2 | Payday weekend adjustment |

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

## ⚠️ Notes on Local Constraints

* **Coding/Truck Ban:** If your supplier is affected by UVVRP (Coding) on a delivery day, manually change Lead Time (L) to **2** for the day prior.
* **Data Cleaning:** If you had a private event that closed the bar, do not include that "0" in your average, as it will skew the forecast too low.

---
