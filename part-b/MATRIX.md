# IRCTC Prioritization Matrix — Part B

## 2×2 Impact vs Effort Analysis

---

## Scoring Framework

### Impact Score (0-10)

**Factors:**
- Users affected (from Part A frequency)
- Severity of problem (Critical = 9-10, High = 6-8, Medium = 4-5)
- Revenue impact (₹ tickets booked affected)
- Booking completion impact (does it increase/decrease completion?)

### Effort Score (0-10)

**Factors:**
- Number of system components touched (1-2 = Low, 3-5 = Medium, 5+ = High)
- New infrastructure needed (new service, database, etc.)
- Dependencies on Railway backend API
- Testing & deployment complexity
- Risk of breaking existing flows

---

## Scoring Table — Each Feature

| Feature | Users Affected | Severity | Booking Impact | Impact Score | Components | Infrastructure | Effort Score |
|---------|---|---|---|---|---|---|---|
| 1. Tatkal Queue | 40L daily | CRITICAL | +30% completion | 9.5 | 5+ | New queue service, Redis, WebSocket | 8 |
| 2. Filter Persistence | 8Cr searches | HIGH | +15% completion | 7.5 | 3 | localStorage, URL state | 3 |
| 3. Seat Lock | 3.6L daily | HIGH | +22% completion | 8 | 4 | DB table, session state, reservation logic | 5 |
| 4. Payment Status | 80% payments | CRITICAL | +22% completion | 9 | 5 | Polling, WebSocket, reconciliation | 7 |
| 5. Session Persist | All users | HIGH | +15% completion | 7 | 4 | sessionStorage, auto-save, recovery | 4 |
| 6. Mobile Optimize | 45% mobile traffic | HIGH | +25% completion | 8.5 | 3 | CSS, React memo, virtual scroll | 6 |

---

## 2×2 Matrix Visualization

```
IMPACT (User Value)
   10  ┌─────────────────────────────────────┐
       │  MAJOR PROJECTS                     │
       │  (Do, but plan carefully)           │
    8  │     ★ Tatkal (9.5,8)                │
       │     ★ Payment (9,7)                 │
       │                                     │
    6  │  QUICK WINS      │ Mobile (8.5,6)  │
       │  (Do first)      │ Seat Lock (8,5) │
       │ Filters (7.5,3)  │                 │
    4  │ Session (7,4)    │ FILL-INS        │
       │                  │ (Do if capacity)│
    2  │                  │ │               │
       │                  │ │               │
    0  └─────────────────────────────────────┘
       0  2  4  6  8  10
         EFFORT (Technical Complexity)
```

---

## Placement Details — All 6 Features

---

### Quadrant 1: QUICK WINS (High Impact / Low Effort)

#### Feature 2: Search Filter Persistence (Impact: 7.5, Effort: 3)

**3-Sentence Justification:**

Filter persistence touches only frontend state management — no new backend infrastructure, no database changes, just smart use of localStorage and URL parameters. It directly reduces user friction during the most common IRCTC flow (search), affecting all 8 crore registered users. Implementation is straightforward React work that can ship in 1-2 sprints with minimal risk.

**Why here:**
- Only 3 system components (frontend, localStorage, URL)
- No new backend infrastructure
- Impacts 100% of search users
- Low risk: Pure frontend change
- High value: 8-15 minutes saved per search

**Do this: WEEK 1**

---

#### Feature 5: Session Persistence with Auto-Save (Impact: 7, Effort: 4)

**3-Sentence Justification:**

Session timeout affects every single user during booking but is solvable with client-side auto-save and token refresh logic — no new infrastructure, just smarter session management. The implementation is contained to frontend hooks and a recovery state table (single DB schema addition). Results: users no longer lose 10+ minutes of form entry if they're temporarily inactive, directly reducing abandonment.

**Why here:**
- 4 components (React hooks, sessionStorage, single DB table, backend refresh logic)
- No complex infrastructure
- Affects 100% of users
- Implementation is well-understood pattern
- High value: prevents abandonment from timeout

**Do this: WEEK 1-2**

---

### Quadrant 2: MAJOR PROJECTS (High Impact / High Effort)

#### Feature 1: Tatkal Virtual Queue System (Impact: 9.5, Effort: 8)

**3-Sentence Justification:**

Tatkal crashes are CRITICAL — 40 lakh users lose tickets daily at 10 AM because the system cannot handle load spikes and provides zero visibility. The solution requires a new microservice (queue system), Redis, WebSocket infrastructure, and careful orchestration with the payment pipeline. But the ROI is exceptional: if Tatkal completion rate goes from 40% to 70%, that's 40 lakh additional successful bookings per day, worth ₹2-3 crore in daily revenue. This is a must-do.

**Why here:**
- 5+ system components (queue service, Redis, WebSocket, DB, payment gateway)
- Requires new infrastructure and careful testing
- Affects 40 lakh peak concurrent users daily
- CRITICAL severity: System crash
- Massive impact: +30% completion during Tatkal

**Timeline: 4-6 weeks, parallel with Payment Status fix**

---

#### Feature 4: Real-Time Payment Status Indicator (Impact: 9, Effort: 7)

**3-Sentence Justification:**

Payment uncertainty is CRITICAL — users don't know if their ₹2000 payment succeeded after authorization, causing panic retries and refund chaos. The fix requires payment reconciliation logic, real-time polling/WebSocket, and status tracking infrastructure but is well-established in fintech. Reducing duplicate payment attempts from 18% to <2% directly improves user trust and reduces payment support overhead.

**Why here:**
- 5 components (payment gateway, reconciliation, polling, WebSocket, DB)
- Requires careful orchestration with payment provider
- Affects 80%+ of booking traffic (all payment attempts)
- CRITICAL severity: Payment failure
- High impact: Reduces duplicate attempts, builds trust

**Timeline: 3-4 weeks, can parallelize with Tatkal**

---

#### Feature 3: Persistent Seat Selection (Impact: 8, Effort: 5)

**3-Sentence Justification:**

Seat selection resetting affects 30-40% of bookings (families, elderly) and directly reduces completion rates. The solution requires server-side seat locking, a new reservation table, and mobile-specific fixes. Implementation is moderate effort but straightforward: reserve seats, track in session state, confirm at payment. The payoff: families no longer lose seat preferences mid-booking, increasing completion by ~20%.

**Why here:**
- 4 components (frontend, seat lock service, DB, session state)
- Requires new seat reservation table and logic
- Affects 30-40% of all bookings directly
- HIGH severity: Affects specific user segments (elderly, families)
- Moderate complexity: Well-understood pattern

**Timeline: 3 weeks, after Tatkal queuing**

---

#### Feature 6: Mobile-Optimized Booking Form (Impact: 8.5, Effort: 6)

**3-Sentence Justification:**

Mobile represents 45% of IRCTC traffic but has only 45% completion rate vs 75% on desktop — that's 30,000+ lost bookings daily due to lag and layout jank. Fixing requires frontend optimization: React.memo, CSS containment, virtual scrolling, and careful testing on low-end devices. High-impact but medium-effort: mostly refactoring existing code for performance, no new APIs or infrastructure.

**Why here:**
- 3 components (frontend React, CSS, performance monitoring)
- No new backend infrastructure
- Affects 45% of all traffic
- HIGH severity: Major UX regression
- High impact: +25% mobile completion

**Timeline: 3 weeks, parallel with other frontend work**

---

### Quadrant 3: FILL-INS (Low Impact / Low Effort)

**Status:** All 6 features are High Impact. No features fall into this quadrant.

---

### Quadrant 4: TIME SINKS (Low Impact / High Effort)

**Status:** All 6 features are either High Impact or justified by effort. No features fall into this quadrant.

---

## Recommended Implementation Order

### Phase 1 (Weeks 1-2) — Quick Wins + Foundation

1. **Filter Persistence** (1-2 weeks, low risk, high user value)
2. **Session Auto-Save** (1-2 weeks, foundational for booking reliability)

**Why first:** Builds momentum with quick wins, improves core search/booking flow confidence.

---

### Phase 2 (Weeks 3-6) — Critical Fixes + Core Infrastructure

3. **Tatkal Queue** (4-6 weeks, most complex, highest impact)
4. **Payment Status** (3-4 weeks, parallelize with Tatkal)

**Why parallel:** Both are payment-critical. Build queue system while hardening payment flow.

---

### Phase 3 (Weeks 7-9) — Polish & Specifics

5. **Seat Lock** (3 weeks, depends on session state from Phase 1)
6. **Mobile Optimization** (3 weeks, parallelize with Seat Lock)

**Why later:** Builds on Phase 1-2 infrastructure; lower risk with Phase 1 foundation solid.

---

### Phase 4 (Weeks 10-12) — Deployment & Monitoring

- AI Waitlist Predictor (4-6 weeks, separate team)
- Comprehensive testing across all 6 features
- Canary deployment to 5% traffic
- Full rollout with monitoring

---

## Impact & Effort Scoring Details

### Impact Score Breakdown

**Tatkal Queue (9.5):**
- Users: 40L peak (highest)
- Severity: CRITICAL (9/10)
- Daily revenue impact: ₹2-3 crore
- Booking completion: +30%

**Payment Status (9):**
- Users: 80% of all bookings
- Severity: CRITICAL (9/10)
- Daily revenue impact: ₹80L (lost refunds + failed payments)
- Booking completion: +22%

**Mobile Optimize (8.5):**
- Users: 45% of traffic
- Severity: HIGH (8/10)
- Booking completion: +25% for mobile users
- Daily impact: 30,000+ additional bookings

**Seat Lock (8):**
- Users: 30-40% of bookings
- Severity: HIGH (8/10)
- Booking completion: +22% for affected segment
- Segment: Families, elderly, high-value bookings

**Filter Persistence (7.5):**
- Users: 8 crore (all users)
- Severity: HIGH (7/10)
- Booking completion: +15% (search flow)
- Daily impact: Massive (all users search multiple times)

**Session Persist (7):**
- Users: 100% of all bookings
- Severity: HIGH (7/10)
- Booking completion: +15%
- Prevents frustration from sudden logout

---

### Effort Score Breakdown

**Tatkal Queue (8):**
- Components: 5+ (queue, Redis, WebSocket, DB, payment)
- Infrastructure: New microservice required
- Testing: Extensive load testing needed
- Dependencies: Payment gateway integration
- Risk: High (can destabilize payment flow if incorrect)

**Payment Status (7):**
- Components: 5 (gateway, polling, WebSocket, reconciliation, DB)
- Infrastructure: WebSocket server, polling logic
- Testing: Must test all payment gateway edge cases
- Dependencies: Payment provider API stability
- Risk: Medium (payment is critical path)

**Mobile Optimize (6):**
- Components: 3 (React, CSS, monitoring)
- Infrastructure: None
- Testing: Device testing matrix (5+ devices)
- Dependencies: None
- Risk: Low (frontend-only change)

**Seat Lock (5):**
- Components: 4 (frontend, lock service, DB, session)
- Infrastructure: Seat reservation table
- Testing: Concurrency testing (two users selecting same seat)
- Dependencies: Session management
- Risk: Low-Medium (clear pattern, well-understood)

**Session Persist (4):**
- Components: 4 (hooks, sessionStorage, DB, refresh)
- Infrastructure: Single recovery table
- Testing: Session expiry scenarios
- Dependencies: Token refresh logic
- Risk: Low (proven pattern)

**Filter Persist (3):**
- Components: 3 (frontend, localStorage, URL)
- Infrastructure: None
- Testing: Browser state testing
- Dependencies: None
- Risk: Minimal (frontend-only)

---

## Final Roadmap

```
WEEK 1-2    [Filter Persist] [Session Auto-Save]
              └─────────────────────────────────┘
                   QUICK WINS
                   (Low risk, high value)

WEEK 3-6    [Tatkal Queue ────────────────────→]
              [Payment Status ───────────────→]
              └──────────────────────────────┘
                   MAJOR PROJECTS
                   (High impact, requires planning)

WEEK 7-9    [Seat Lock] [Mobile Optimize]
              [────────────────────────────]
                   POLISH & SPECIFICS
                   (Depends on Phase 1-2)

WEEK 10-12  [AI Predictor (parallel, separate team)]
              [Testing + Canary + Full Rollout]
              [──────────────────────────────]
                   LAUNCH READINESS
```

---

## Success Criteria for Phase 1 → Phase 2 Transition

Before proceeding to Tatkal Queue and Payment Status work:

- [ ] Filter persistence reduces search time by 50% (verified in testing)
- [ ] Session auto-save saves 90% of in-flight bookings (no more logout losses)
- [ ] Zero critical bugs found during UAT
- [ ] Queue service architecture reviewed by Payment team
- [ ] Payment reconciliation logic prototype completed

---

## End of Matrix
