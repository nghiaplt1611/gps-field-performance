# 📐 Metrics Definitions — GPS Field Performance

> *This document defines all KPIs, metrics, and business rules used in the GPS Field Performance project. Each metric includes its formula, data source, grain, and edge case handling.*

---

## Table of Contents

- [Business Rules](#business-rules)
  - [The 5-Timeframe Rule](#the-5-timeframe-rule)
  - [Visit Legitimacy Criteria](#visit-legitimacy-criteria)
- [Core Metrics](#core-metrics)
  - [Visit Metrics](#visit-metrics)
  - [GPS Quality Metrics](#gps-quality-metrics)
  - [Performance Metrics](#performance-metrics)
  - [Compliance Metrics](#compliance-metrics)
- [Composite Scores](#composite-scores)
  - [Legitimacy Score](#legitimacy-score)
- [Metric Catalog Summary](#metric-catalog-summary)

---

## Business Rules

### The 5-Timeframe Rule

> **Purpose**: Ensure field agents conduct visits throughout the working day, rather than batch-processing visits in a single window.

#### Definition

The working day is divided into **5 timeframes**:

| Timeframe | Window | Code |
|---|---|---|
| TF1 | 08:00 – 10:00 | `morning_early` |
| TF2 | 10:00 – 12:00 | `morning_late` |
| TF3 | 12:00 – 14:00 | `midday` |
| TF4 | 14:00 – 16:00 | `afternoon_early` |
| TF5 | 16:00 – 18:00 | `afternoon_late` |

<!-- 🔧 UPDATE: Adjust timeframes to match your actual business rules -->

#### Compliance Criteria

| Level | Timeframes Covered | Status |
|---|---|---|
| ✅ **Fully Compliant** | 5 / 5 | Agent active in all timeframes |
| 🟡 **Partially Compliant** | 3–4 / 5 | Acceptable with justification |
| 🔴 **Non-Compliant** | ≤ 2 / 5 | Flagged for review |

#### Calculation Logic

```sql
-- Pseudocode for timeframe compliance
WITH visit_timeframes AS (
    SELECT
        agent_id,
        visit_date,
        CASE
            WHEN EXTRACT(HOUR FROM visit_timestamp) BETWEEN 8 AND 9 THEN 'TF1'
            WHEN EXTRACT(HOUR FROM visit_timestamp) BETWEEN 10 AND 11 THEN 'TF2'
            WHEN EXTRACT(HOUR FROM visit_timestamp) BETWEEN 12 AND 13 THEN 'TF3'
            WHEN EXTRACT(HOUR FROM visit_timestamp) BETWEEN 14 AND 15 THEN 'TF4'
            WHEN EXTRACT(HOUR FROM visit_timestamp) BETWEEN 16 AND 17 THEN 'TF5'
            ELSE 'outside_hours'
        END AS timeframe
    FROM collection_visit
    WHERE visit_gps_lat IS NOT NULL  -- Only count GPS-verified visits
)
SELECT
    agent_id,
    visit_date,
    COUNT(DISTINCT timeframe) AS timeframes_covered,
    CASE
        WHEN COUNT(DISTINCT timeframe) = 5 THEN 'fully_compliant'
        WHEN COUNT(DISTINCT timeframe) >= 3 THEN 'partially_compliant'
        ELSE 'non_compliant'
    END AS compliance_status
FROM visit_timeframes
WHERE timeframe != 'outside_hours'
GROUP BY agent_id, visit_date;
```

#### Edge Cases

| Scenario | Handling |
|---|---|
| Agent starts shift late | Only count timeframes within agent's actual shift hours |
| Half-day / leave | Adjust denominator proportionally |
| Visits outside 08:00–18:00 | Excluded from timeframe count; flagged as `outside_hours` |
| Visit exactly at boundary (e.g., 10:00:00) | Assigned to the starting timeframe (TF2 in this case) |

---

### Visit Legitimacy Criteria

A visit is considered **legitimate (verified)** if it meets ALL of the following criteria:

| Criterion | Threshold | Rationale |
|---|---|---|
| **GPS coordinates present** | `visit_gps_lat IS NOT NULL` | Cannot verify without GPS |
| **GPS accuracy acceptable** | `accuracy_meters <= 50` | Unreliable beyond 50m |
| **Minimum duration** | `visit_duration >= X minutes` | Too short = not a real visit |
| **Reasonable travel speed** | Implied speed from previous visit `<= 120 km/h` | Physically impossible speeds indicate data issues |
| **Not a duplicate** | Time gap from previous visit `>= Y seconds` | Rapid successive visits indicate gaming |

<!-- 🔧 UPDATE: Replace X, Y with your actual thresholds -->

```sql
-- Pseudocode for visit legitimacy
SELECT
    visit_id,
    CASE
        WHEN visit_gps_lat IS NULL THEN 'no_gps'
        WHEN accuracy_meters > 50 THEN 'low_accuracy'
        WHEN visit_duration_sec < (X * 60) THEN 'too_short'
        WHEN implied_speed_kmh > 120 THEN 'impossible_speed'
        WHEN seconds_since_prev_visit < Y THEN 'rapid_successive'
        ELSE 'verified'
    END AS legitimacy_status
FROM enriched_visits;
```

---

## Core Metrics

### Visit Metrics

#### Total Visits

| Property | Value |
|---|---|
| **Name** | `total_visits` |
| **Description** | Total number of visit events logged by an agent on a given day |
| **Formula** | `COUNT(visit_id) WHERE agent_id = ? AND visit_date = ?` |
| **Grain** | Agent × Day |
| **Source** | `collection_visit` |
| **Notes** | Includes all visits regardless of legitimacy status |

#### Verified Visits

| Property | Value |
|---|---|
| **Name** | `verified_visits` |
| **Description** | Number of visits that passed all legitimacy criteria |
| **Formula** | `COUNT(visit_id) WHERE legitimacy_status = 'verified'` |
| **Grain** | Agent × Day |
| **Source** | `collection_visit` + GPS validation |
| **Notes** | This is the **primary visit metric** for performance evaluation |

#### Flagged Visits

| Property | Value |
|---|---|
| **Name** | `flagged_visits` |
| **Description** | Number of visits that failed one or more legitimacy criteria |
| **Formula** | `total_visits - verified_visits` |
| **Grain** | Agent × Day |
| **Source** | Derived |
| **Notes** | Breakdown by flag reason available in detail table |

#### Visit Verification Rate

| Property | Value |
|---|---|
| **Name** | `visit_verification_rate` |
| **Description** | Percentage of total visits that are GPS-verified |
| **Formula** | `verified_visits / NULLIF(total_visits, 0) * 100` |
| **Grain** | Agent × Day |
| **Source** | Derived |
| **Notes** | Target: ≥ 80% (configurable) |

---

### GPS Quality Metrics

#### Average GPS Accuracy

| Property | Value |
|---|---|
| **Name** | `avg_gps_accuracy_m` |
| **Description** | Average GPS accuracy radius in meters across all pings for the day |
| **Formula** | `AVG(accuracy_meters) WHERE agent_id = ? AND DATE(recorded_at) = ?` |
| **Grain** | Agent × Day |
| **Source** | `gps_10s_pings` |
| **Notes** | Lower is better. Values > 50m indicate poor GPS quality |

#### GPS Coverage Rate

| Property | Value |
|---|---|
| **Name** | `gps_coverage_rate` |
| **Description** | Percentage of expected working hours with GPS signal |
| **Formula** | `(hours_with_gps_signal / expected_working_hours) * 100` |
| **Grain** | Agent × Day |
| **Source** | `gps_10s_pings` |
| **Notes** | Low coverage may indicate GPS disabling or device issues |

#### Total Distance Traveled

| Property | Value |
|---|---|
| **Name** | `total_distance_km` |
| **Description** | Total distance traveled by agent based on GPS trajectory |
| **Formula** | `SUM(haversine_distance(point_n, point_n+1))` |
| **Grain** | Agent × Day |
| **Source** | `gps_10s_pings` (or `fpa_gps_records` if pre-computed) |
| **Notes** | Filtered to exclude GPS noise (jumps > 1km in 10s) |

---

### Performance Metrics

#### Collection Success Rate

| Property | Value |
|---|---|
| **Name** | `collection_success_rate` |
| **Description** | Percentage of verified visits that resulted in successful collection |
| **Formula** | `COUNT(visit_id WHERE visit_result = 'collected' AND legitimacy = 'verified') / NULLIF(verified_visits, 0) * 100` |
| **Grain** | Agent × Day |
| **Source** | `collection_visit` + GPS validation |
| **Notes** | Only counts verified visits to avoid inflating rate with fake visits |

#### Average Visit Duration

| Property | Value |
|---|---|
| **Name** | `avg_visit_duration_min` |
| **Description** | Average time spent per verified visit in minutes |
| **Formula** | `AVG(visit_duration_sec / 60.0) WHERE legitimacy = 'verified'` |
| **Grain** | Agent × Day |
| **Source** | `collection_visit` or GPS-derived dwell time |
| **Notes** | GPS-derived dwell time is more reliable than self-reported duration |

#### Average Inter-Visit Distance

| Property | Value |
|---|---|
| **Name** | `avg_inter_visit_dist_km` |
| **Description** | Average geographic distance between consecutive verified visits |
| **Formula** | `AVG(haversine_distance(visit_n, visit_n+1)) WHERE both visits verified` |
| **Grain** | Agent × Day |
| **Source** | `collection_visit` GPS coordinates |
| **Notes** | Very low values may indicate agent visiting same area repeatedly |

---

### Compliance Metrics

#### Timeframes Covered

| Property | Value |
|---|---|
| **Name** | `timeframes_covered` |
| **Description** | Number of the 5 defined timeframes with at least 1 verified visit |
| **Formula** | `COUNT(DISTINCT timeframe) WHERE legitimacy = 'verified'` |
| **Grain** | Agent × Day |
| **Source** | Derived from `collection_visit` |
| **Notes** | Core metric for the 5-Timeframe Rule |

#### Compliance Status

| Property | Value |
|---|---|
| **Name** | `is_compliant` |
| **Description** | Whether the agent meets all business rules for the day |
| **Formula** | `timeframes_covered >= 3 AND verified_visits >= X AND flagged_visit_rate <= Y%` |
| **Grain** | Agent × Day |
| **Source** | Derived |
| **Notes** | Composite boolean flag; thresholds configurable |

---

## Composite Scores

### Legitimacy Score

> **Purpose**: A single 0–100 score summarizing the overall quality and legitimacy of an agent's daily field activity.

#### Components & Weights

| Component | Weight | Calculation | Score Range |
|---|---|---|---|
| **Visit Verification Rate** | 30% | `verified_visits / total_visits * 100` | 0–100 |
| **Timeframe Compliance** | 25% | `timeframes_covered / 5 * 100` | 0–100 |
| **Avg Visit Duration** | 20% | Normalized against benchmark | 0–100 |
| **GPS Coverage** | 15% | `hours_with_gps / working_hours * 100` | 0–100 |
| **Distance Reasonableness** | 10% | Deviation from regional median | 0–100 |

<!-- 🔧 UPDATE: Adjust weights and components to match your actual scoring model -->

#### Formula

```
legitimacy_score = (
    visit_verification_rate * 0.30
  + timeframe_score * 0.25
  + duration_score * 0.20
  + gps_coverage_score * 0.15
  + distance_score * 0.10
)
```

#### Interpretation

| Score Range | Label | Action |
|---|---|---|
| 80–100 | 🟢 Excellent | No action needed |
| 60–79 | 🟡 Acceptable | Monitor; provide coaching if trending down |
| 40–59 | 🟠 Concerning | Review with supervisor; increase QC sampling |
| 0–39 | 🔴 Critical | Immediate investigation required |

---

## Metric Catalog Summary

| Metric | Category | Grain | Type | Formula Reference |
|---|---|---|---|---|
| `total_visits` | Visit | Agent × Day | Count | [→](#total-visits) |
| `verified_visits` | Visit | Agent × Day | Count | [→](#verified-visits) |
| `flagged_visits` | Visit | Agent × Day | Count | [→](#flagged-visits) |
| `visit_verification_rate` | Visit | Agent × Day | Percentage | [→](#visit-verification-rate) |
| `avg_gps_accuracy_m` | GPS Quality | Agent × Day | Average | [→](#average-gps-accuracy) |
| `gps_coverage_rate` | GPS Quality | Agent × Day | Percentage | [→](#gps-coverage-rate) |
| `total_distance_km` | GPS Quality | Agent × Day | Sum | [→](#total-distance-traveled) |
| `collection_success_rate` | Performance | Agent × Day | Percentage | [→](#collection-success-rate) |
| `avg_visit_duration_min` | Performance | Agent × Day | Average | [→](#average-visit-duration) |
| `avg_inter_visit_dist_km` | Performance | Agent × Day | Average | [→](#average-inter-visit-distance) |
| `timeframes_covered` | Compliance | Agent × Day | Count | [→](#timeframes-covered) |
| `is_compliant` | Compliance | Agent × Day | Boolean | [→](#compliance-status) |
| `legitimacy_score` | Composite | Agent × Day | Score (0–100) | [→](#legitimacy-score) |

---

<p align="center"><a href="../README.md">← Back to README</a></p>
