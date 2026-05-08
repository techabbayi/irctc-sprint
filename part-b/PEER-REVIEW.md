# IRCTC Part B — Peer Review Summary & Updates

**Review Date:** [To be filled]  
**Reviewers:** Product Manager + Engineering Lead  
**Sessions Completed:** [To be filled]

---

# Peer Review Session 1: Tatkal Queue Feature

## Presentation Summary

Presented the Tatkal virtual queue system as the highest-impact, highest-effort feature. Explained the problem (40 lakh concurrent users crash the system daily), the solution (visible queue with position + progress), and the technical plan (Redis queue + WebSocket + 90-second slot expiry).

---

## PM Questions & Responses

**Q1: "What happens if a user's 90-second slot expires while they're selecting seats?"**

**A:** User sees clear notification: "Your slot expired. Your search parameters and seat preferences are saved. Return to queue?" System re-enters user into queue with same search criteria. No data loss.

**Follow-up:** "How do we prevent abuse where users intentionally let slots expire?"

**Response:** Add counter: "You have 3 slots per session. After 3 expirations, user waits 10 minutes before re-entering queue." Added to spec.

---

**Q2: "How do you measure success? Is 70% completion realistic or optimistic?"**

**A:** Historical data shows 40% Tatkal completion currently. With queue visibility (removes panic-refresh amplification) and slot guarantee (removes session timeout risk), 70% is conservative. Conservative baseline: 60%. Success: 70%+.

**PM acceptance:** Okay. But also track: "completion rate during 10 AM peak" separately from off-peak.

**Update made:** Split success metrics into:
- Peak hour (10:00-10:05): Target 75%
- Off-peak: Target 65%

---

**Q3: "The queue system is new infrastructure. What's the fallback if Redis goes down?"**

**A:** Fallback is PostgreSQL-backed queue (slower but works). System degrades gracefully but is not fully down.

**PM challenge:** "Is that tested? Because if it fails in production during Tatkal, we lose ₹5 crore."

**Response:** Must include chaos engineering tests: redis-chaos, network partition, and slow redis scenarios.

**Update made:** Added to technical plan:
- Chaos engineering tests (3-week testing budget)
- Switchover to PostgreSQL queue if Redis unavailable
- Monitoring alert: "Queue system degraded. Switch to fallback."

---

## Engineering Questions & Responses

**Q1: "WebSocket connections at 40L scale. How many servers do we need?"**

**A:** Estimated 500K concurrent WebSocket connections. With 50K connections per server = 10 servers. Plus load balancer + failover redundancy = 15 servers total.

**Eng challenge:** "That's ₹50 lakh/month in infrastructure. How does ROI justify it?"

**Response:** ₹2-3 crore daily revenue * 30 days = ₹60-90 crore. Infrastructure cost is negligible.

**Update made:** Added infrastructure cost/ROI analysis to spec.

---

**Q2: "Payment gateway integration. How do we coordinate queue slot expiry with payment processing?"**

**A:** When user confirms seat, we lock 120-second payment window. Slot doesn't expire during payment. After payment completes, slot is confirmed.

**Eng concern:** "What if payment takes 90 seconds? Slot expires before payment completes."

**Response:** Updated slot expiry to 150 seconds (2.5 min) for payment processing buffer.

**Update made:** Extended slot duration: 90 seconds → 150 seconds. Added logic: "Payment doesn't consume slot time."

---

**Q3: "Seat selection within 90-second window. Passengers might not select fast enough."

**A:** Pre-fill user's historical seat preferences (lower berth if elderly user). Auto-select "Auto" if no preference. User can manually override in 30 seconds.

**Eng question:** "Where do we store historical preferences?"

**Response:** Add to user profile:
```
user_preferences {
  preferred_berth: "lower",
  preferred_side: "window",
  accessibility_needs: boolean
}
```

**Update made:** Added user preference table to DB schema.

---

## Specification Updates from Review 1

**Changes made:**
1. Split success metrics: Peak vs off-peak completion targets
2. Added chaos engineering testing plan (3 weeks)
3. Added PostgreSQL fallback queue logic
4. Added infrastructure ROI analysis
5. Extended slot duration from 90 to 150 seconds
6. Added user preference pre-population
7. Added user preference storage table

---

---

# Peer Review Session 2: Payment Status Feature

## Presentation Summary

Presented payment status tracking as equally critical to Tatkal queue. Problem: After UPI authorization, users see infinite spinner with zero feedback, causing panic retries and duplicate payments (18% of payments). Solution: Real-time polling + WebSocket with step-by-step feedback.

---

## PM Questions & Responses

**Q1: "If payment status polling shows 'failed', what's the user experience?"**

**A:** User sees: "Payment declined by your bank. [Retry] [Try different method]." System doesn't charge again if already charged. Handles idempotency server-side.

**PM follow-up:** "But users check their bank account and see deduction. Then they see 'failed' on IRCTC. Pure confusion."

**Response:** Added explicit messaging:
- "Payment succeeded at bank level. Processing your ticket..."
- "Your booking is confirmed" (explicitly)
- "If uncertain: Check your bank transaction history. Contact support link provided."

**Update made:** Added user-facing messaging for all payment states:
- Initiated
- Authorized
- Processing
- Confirmed
- Failed
- Ambiguous (charge deducted but no confirmation received)

---

**Q2: "30-second timeout for payment confirmation. That's too short for slow connections. What if user is on 2G?"**

**A:** Good catch. Changed to adaptive timeout:
- 4G: 30 seconds
- 3G: 60 seconds
- 2G: 120 seconds

System detects network speed and adjusts.

**PM concern:** "How do you detect network speed?"

**Response:** Use Bandwidth API or estimate from first API call latency. Plus user can manually override: "Network slow? Click to wait longer."

**Update made:** Added network-aware timeout logic to spec.

---

**Q3: "What about retry logic? If payment polling fails, do we retry?"**

**A:** Yes. Retry up to 5 times with exponential backoff:
- 1st retry: 2 seconds
- 2nd retry: 5 seconds
- 3rd retry: 10 seconds
- 4th retry: 20 seconds
- 5th retry: 30 seconds
- After 5 retries: Show "Status unclear. Confirm?" with manual check option.

**PM acceptance:** Good. But also log every retry with timestamp for debugging.

**Update made:** Added detailed retry + logging strategy.

---

## Engineering Questions & Responses

**Q1: "Payment gateway callback latency. What if callback arrives 5 minutes late?"**

**A:** Polling is defensive. We don't rely solely on callback. Polling checks status every 2 seconds up to 4 minutes. If callback arrives late (5+ min), polling has already confirmed the payment.

**Eng concern:** "So we have race conditions between polling and callback?"

**Response:** Polling result is source of truth. Callback updates if polling missed it. Both update the same database field atomically.

**Update made:** Added atomic database update logic for payment status.

---

**Q2: "WebSocket connection drops during payment confirmation. User sees blank screen."

**A:** Automatic fallback to polling. User doesn't notice the difference. Both polling and WebSocket are redundant channels for same data.

**Eng question:** "What's the impact on infrastructure?"

**Response:** Added minimal load. Polling is lightweight HTTP GET. WebSocket reduces polling frequency to "on change only" rather than polling every 2 seconds.

---

**Q3: "How do you handle duplicate payments if user retries multiple times?"

**A:** Each payment attempt gets unique idempotency_key. If same key is used twice, system returns "Payment already processed" without charging again.

**Eng acceptance:** Good idempotency design.

---

## Specification Updates from Review 2

**Changes made:**
1. Added explicit messaging for all payment states
2. Added network-aware timeout logic (2G/3G/4G)
3. Added retry + exponential backoff strategy with logging
4. Added atomic database updates for payment status
5. Clarified WebSocket + polling redundancy
6. Added idempotency key logic

---

---

# Peer Review Session 3: Mobile Optimization

## Presentation Summary

Presented mobile optimization as critical issue: 45% of traffic but only 45% completion rate vs 75% on desktop. Solution: React.memo, CSS containment, virtual scrolling, low-end device testing.

---

## PM Questions & Responses

**Q1: "How do you measure 'laggy form'? What's the target?"**

**A:** Measure: Time from tap to text appearance (input interaction latency). Target: <100ms. Current: 200-500ms on low-end devices.

**PM follow-up:** "Is <100ms realistic on Android 5?"**

**Response:** 100ms is threshold for "feels responsive." Achievable through component memoization + virtual scrolling. Worst case: 150ms on very old devices (acceptable).

**Update made:** Added interaction latency metrics to success criteria.

---

**Q2: "You're using React.memo everywhere. Won't that cause over-optimization?"**

**A:** Good point. Use React.memo only for expensive components (seat map, passenger list). Use useMemo for computed values only if re-computation is expensive. Measure first, optimize second.

**PM acceptance:** Makes sense. Include performance profiling in test plan.

**Update made:** Refined optimization strategy: profile first, then selective memo.

---

**Q3: "Virtual scrolling for passenger forms. Will this hurt accessibility?"**

**A:** Valid concern. Virtual scrolling hides offscreen content from DOM. Screen readers won't read hidden passengers. Need `aria-live` region + keyboard nav fixes.

**PM challenge:** "IRCTC has accessibility compliance requirements. This could break them."

**Response:** Virtual scrolling only if 10+ passengers (rare). Default: standard scrolling. When enabled: add accessibility enhancements:
- ARIA live region for passenger list
- Keyboard navigation between passengers
- Screen reader announcements

**Update made:** Added accessibility requirements for virtual scrolling.

---

## Engineering Questions & Responses

**Q1: "DevTools impact on performance measurements. How do you test without skewing results?"**

**A:** Test without DevTools. Use Lighthouse CI (headless). Measure on real devices + emulators with various network conditions (3G, 4G, 2G).

**Eng concern:** "That's a lot of device testing infrastructure."

**Response:** Use BrowserStack for remote devices. Or use Lighthouse CI on CI/CD pipeline.

---

**Q2: "CSS containment browser support. Will older Android browsers support it?"**

**A:** `contain: layout` has 90%+ support on modern browsers. For older browsers: feature detection + fallback to standard CSS.

**Eng question:** "Does fallback have performance impact?"

**Response:** Yes, minimal. Modern browsers (90% of traffic) get full optimization. Older browsers (10%) get standard rendering - still acceptable, just not optimized.

---

**Q3: "Form re-renders with every keystroke. Have you considered debouncing?"**

**A:** Yes. Added 300ms debounce on onChange events. Plus useCallback memoization to prevent parent re-renders on input change.

**Eng acceptance:** Good approach.

---

## Specification Updates from Review 3

**Changes made:**
1. Added interaction latency metrics (<100ms target)
2. Refined optimization strategy: profile first, then selective optimization
3. Added accessibility requirements for virtual scrolling
4. Added Lighthouse CI + BrowserStack testing strategy
5. Added feature detection for CSS containment
6. Added debouncing + useCallback strategy

---

---

# Overall Peer Review Outcomes

## Summary of Updates

**Total specification updates:** 16 changes across 3 features

**Quality of changes:** All updates were either:
- Risk mitigation (chaos testing for queue)
- Accessibility fixes (virtual scrolling)
- Performance improvements (debouncing)
- Clearer messaging (payment states)

**No features were deprioritized or eliminated.**

---

## Go/No-Go Decision

All 3 features reviewed are **GO** to proceed to Phase 1 implementation.

**Additional checks before coding:**
- [ ] Chaos engineering tests designed (Queue)
- [ ] Payment idempotency logic reviewed by Payments team
- [ ] Accessibility audit completed (Mobile)
- [ ] Infrastructure capacity planned (Queue + Payment)

---

## Recommendation for Peer Review Process

**For Part B final submission:**

1. Have 2-3 peer reviewers (different specialties: PM, Backend Eng, Frontend Eng)
2. Each reviewer spends 1 hour reading the specs
3. Present top 2-3 features (don't overwhelm with 6)
4. Answer questions rigorously
5. Update specs based on feedback
6. Share updated specs in PR description

**Time estimate:** 4-6 hours total per feature

---

# Next Steps

1. Update all 6 specs with learnings from these reviews
2. Schedule review for remaining 3 features (Seat Lock, Session Persist, Filter Persist)
3. Finalize AI Feature spec for review
4. Begin Phase 1 implementation (Filter Persist + Session Auto-Save)

---

**Peer Review Completed By:** [Name]  
**Date:** [To be filled]  
**Sign-Off:** [Approved / Needs Updates]
