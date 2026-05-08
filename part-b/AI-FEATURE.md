# IRCTC AI Feature Proposal — Part B

---

# AI Feature: Waitlist Confirmation Probability Predictor

## Problem It Solves

Reference: Part A, Problem 5 (Session timeout) and Problem 4 (Payment uncertainty)

Users booking waitlisted tickets have zero visibility into confirmation probability. They pay for WL tickets without knowing if they'll get confirmed. This uncertainty causes abandonment and distrust. Additionally, many users don't understand the difference between WL1, WL15, and WL250 in terms of actual confirmation likelihood. The AI solution: a real-time probability model that estimates the likelihood of waitlist confirmation based on historical train data, route, date, passenger count, and class.

---

## Model Choice: Gradient Boosting (XGBoost)

**Why this model:**
- Handles non-linear relationships between features (date, class, route, passenger count, historical patterns)
- Fast inference (<100ms) suitable for real-time API calls
- Can be trained offline and deployed as a lightweight model
- Works well with structured tabular data (historical bookings)
- Explainable: Can show which factors most influence the prediction

**Alternatives considered:**
- Neural networks: Too slow for inference, overkill for tabular data
- Simple linear regression: Insufficient to capture complex booking patterns
- Rule-based system: Cannot adapt to new patterns, requires manual updates

---

## Training Data & Sources

**Historical Data Required (4 years):**

1. **Booking History:**
   - Train route, class, quota, date
   - Number of WL bookings on that train
   - Number of WL confirmations
   - Time taken for confirmation (1 hour, 1 day, 3 days)
   - Passenger count

2. **Cancellation Data:**
   - Daily cancellation rates by route/class
   - Peak vs off-season cancellation patterns
   - Festival season anomalies

3. **Seasonal Patterns:**
   - Tourist season vs regular season
   - Summer vacation periods
   - Festival dates (Diwali, Holi, Christmas)

4. **External Features:**
   - Day of week (weekends vs weekdays)
   - Advance booking period (booked 60 days ago vs 10 days ago)
   - Competitor traffic (MakeMyTrip, other bookings)

**Data Collection:**
- Source: IRCTC historical booking database
- Volume: 50+ million booking records
- Training set: 40 million, Validation: 5 million, Test: 5 million

**Feature Engineering:**

```python
features = {
    'route_id': int,  # e.g., Delhi-Mumbai = 101
    'class': categorical,  # ['AC', 'Sleeper', 'General']
    'quota': categorical,  # ['General', 'Tatkal', 'Senior', 'Divyang']
    'wl_position': int,  # 1-500
    'passenger_count': int,  # 1-6
    'booking_advance_days': int,  # days before departure
    'day_of_week': int,  # 0-6 (Mon-Sun)
    'is_festival_period': bool,
    'is_holiday': bool,
    'historical_confirmation_rate': float,  # 0.0-1.0
    'avg_cancellation_rate_this_route': float,
    'is_peak_season': bool,
    'train_type': categorical,  # ['Express', 'Rajdhani', 'Local']
}

target = 'confirmed_within_24_hours'  # 1 if confirmed, 0 if not
```

---

## Model Output: What User Sees

**On Booking Page — After selecting WL ticket:**

```
WAITLIST #47 — Your Confirmation Chances

Predicted Confirmation Probability: 72%

What this means:
- Based on historical data for this train/class/date
- Similar bookings in the past: 72 out of 100 got confirmed
- Typical confirmation time: 6-18 hours
- You'll receive SMS alert when confirmed

Confidence level: HIGH ✓
(Prediction based on 8,000+ similar bookings)

Factors increasing your chances:
✓ 47 people ahead of you (WL depth manageable)
✓ Tourist season (high cancellations expected)
✓ 4-day advance booking (cancellations still possible)

Factors reducing your chances:
✗ Sleeper class (higher demand than expected)

[ Book Anyway ] [ Try Another Option ]
```

---

## API Integration

**New Endpoint:**

```
POST /prediction/wl-confirmation-probability

Request:
{
  train_id: 12622,
  class: "Sleeper",
  quota: "General",
  wl_position: 47,
  passenger_count: 2,
  departure_date: "2026-05-20",
  booking_advance_days: 4
}

Response:
{
  probability: 0.72,
  confidence: 0.89,
  typical_confirmation_time_hours: 12,
  similar_bookings_count: 8000,
  contributing_factors: {
    positive: ["WL depth manageable", "tourist season"],
    negative: ["high demand class"]
  },
  prediction_id: "uuid-for-tracking"
}
```

**Implementation:**
- Model hosted as microservice (FastAPI or Flask)
- Inference model: XGBoost model file (50MB)
- Response time: <100ms
- Caching: Cache predictions for same train+class+date+position for 5 minutes

---

## Technical Implementation

**Architecture:**

```
User selects WL ticket
    ↓
Frontend calls GET /prediction/wl-probability
    ↓
Backend API (FastAPI)
    ↓
ML Model Service (XGBoost inference)
    ↓
Returns probability + explanation
    ↓
Frontend displays prediction with visualization
```

**Deployment:**
- Model: Docker container on separate microservice
- Server: AWS SageMaker or self-hosted
- Load: Expected 500K predictions/day during peak hours
- Scale: Auto-scale instances based on queue length

**Model Retraining:**
- Frequency: Monthly
- Process: Automated pipeline on Airflow
- New data: Last month's bookings + confirmations
- Validation: A/B test new model against current model before deployment

---

## How to Show Uncertainty & Fallback

**When confidence is LOW (<60%):**
```
WAITLIST #142 — Confirmation Uncertain

Predicted Probability: 45%

Not enough historical data for this specific scenario.
But here's what we know:
- General class on this route: 65% confirmation
- Holiday season: Usually higher confirmations
- Your position: Deep, but not impossible

[ Proceed ] [ Look for other options ]
```

**When model is unavailable:**
```
WAITLIST #47

Typical confirmation (based on historical average):
Sleeper class on this route: 68% usually confirmed

Real-time probability unavailable. Booking anyway?
[ Yes ] [ No ]
```

**Fallback strategy:**
- Show historical statistics instead of live prediction
- Let user proceed even if model fails
- Log failures for debugging
- Never block booking due to ML service outage

---

## Success Metrics

- User understanding of WL probability: 20% → 85% (survey: "Do you understand your chances?")
- Confidence in WL booking decision: Low → High
- WL booking completion rate: 50% → 72%
- WL confirmation abandonment: (Users changing mind after seeing low probability) Expected: 15%
- Prediction accuracy on test set: 78%+
- Prediction latency: <100ms
- Model precision on "confirmed" predictions: 80%+
- Model recall on "not confirmed": 75%+

---

## Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Model predicts 95%, user doesn't confirm | Low user trust | Show confidence intervals; explain uncertainty |
| Model inaccurate during festivals | User distrust | Retrain monthly; A/B test before deploy |
| ML service goes down | Booking interrupted | Fallback to historical statistics |
| Model biased by past patterns | Unfair to minority routes | Monitor fairness metrics; override if biased |
| User books too many WLs based on high prob | Overbooking scenario | Show: "You have 3 WL bookings. Confirm one soon." |

---

## Why This AI, Why Now

This AI feature directly addresses uncertainty from Part A. Users currently feel anxious about WL bookings because there's zero information. A probability model turns data debt into user empowerment. The model is:

- **Technically feasible:** XGBoost is proven, industry-standard
- **Data-driven:** Built on 50M+ real bookings
- **User-centric:** Explains reasoning in plain language
- **Gracefully degrading:** Falls back to statistics if model fails
- **Deployable:** Can be built, tested, and shipped in 8-12 weeks

---

# End of AI Feature Proposal
