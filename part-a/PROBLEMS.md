# IRCTC Problem Audit Report

> Design Engineering & AI Feature Sprint — Part A
>
> Platform Audited: IRCTC Web + Mobile Experience
>
> Audit Type: UX, Reliability, Scalability & System Flow Analysis
>
> Prepared For: Product Engineering Assignment
>
> Prepared by: Lokeswara Reddy Muthumula

---

# Executive Summary

IRCTC is one of the largest public-facing digital platforms in India, handling millions of train searches and ticket bookings daily. Despite its massive scale and critical role in Indian transportation, the platform suffers from recurring issues involving server crashes, unstable booking flows, payment uncertainty, poor mobile responsiveness, and weak state management.

This audit documents 6 major user-facing problems observed across the booking journey. Each problem includes:

- What is broken
- Affected users
- Frequency analysis
- Current user flow
- Exact failure point
- Root cause analysis

The objective of this report is to identify high-impact usability and engineering failures that affect booking reliability and user trust.

---

# Severity Levels

| Severity | Meaning |
|---|---|
| CRITICAL | Prevents booking/payment completion |
| HIGH | Major friction causing booking failure or abandonment |
| MEDIUM | Significant usability inefficiency |
| LOW | Minor inconvenience |

---

# Problem 1 — Tatkal Booking Crash at 10:00 AM

## Severity
CRITICAL

## What is Broken

The IRCTC server becomes extremely slow or crashes exactly at 10:00 AM when Tatkal quota booking opens. Users experience HTTP 502 errors, session timeouts, CAPTCHA resets, failed OTP verification, and payment uncertainty.

Even users who successfully reach the payment page often lose their session before completion.

---

## Affected Users

- Tatkal booking users across India
- Emergency travelers
- Daily commuters
- Tier-2 and Tier-3 city passengers
- Users dependent on urgent railway travel

Estimated concurrent traffic during Tatkal opening:
20–40 lakh active users.

---

## Frequency

Occurs daily during Tatkal booking windows:

- 10:00 AM for AC classes
- 11:00 AM for non-AC classes

The issue has existed for years and is widely reported.

---

## Current User Flow — Step by Step

1. User opens IRCTC around 9:50 AM
2. User logs into account
3. User searches for train
4. User selects Tatkal quota
5. Availability shows “Available 12” before opening
6. User fills passenger details quickly
7. User clicks “Book Now” exactly at 10:00 AM
8. Loading spinner appears with no progress feedback
9. Page freezes for 15–45 seconds
10. User receives HTTP 502 / timeout / CAPTCHA reset
11. User refreshes page
12. Session expires and user is logged out
13. User logs back in
14. Tatkal quota becomes waitlisted or unavailable
15. User checks bank statement in panic to verify payment status

---

## Where Exactly It Breaks

The failure occurs between steps 7–12.

The backend receives massive concurrent booking requests with insufficient queue handling and poor request throttling. The frontend provides no queue position or progress visibility, causing users to repeatedly refresh and amplify server load.

---

## Root Cause Analysis

- No virtual queueing system
- Weak concurrency handling
- Backend overload during peak traffic
- No real-time booking progress UI
- Session instability under high load
- Repeated retries worsening server pressure

---

# Problem 2 — Search Filters Reset or Show Incorrect Results

## Severity
HIGH

## What is Broken

Search filters on the train results page frequently fail to apply correctly. Trains marked as “Waitlisted” continue appearing even after selecting “Available Only.” Filters also reset automatically when users navigate back.

This forces users to manually scan large train lists repeatedly.

---

## Affected Users

- All train search users
- Senior citizens
- First-time users
- Mobile users
- Users comparing multiple train classes and quotas

Estimated impact:
All users searching among 20–40 train options.

---

## Frequency

Occurs inconsistently but increases significantly during high traffic periods.

Observed failure rate:
Approximately 30–40% during repeated searches.

---

## Current User Flow — Step by Step

1. User enters source station
2. User enters destination station
3. User selects travel date
4. User clicks “Search Trains”
5. Results page loads with 20–40 trains
6. User selects “Sleeper Class” filter
7. User selects “Available Only” filter
8. Page reloads
9. Waitlisted trains still appear
10. User opens train details
11. User goes back to results page
12. Filters reset to default values
13. User manually reapplies all filters again
14. User scans trains manually due to low trust in filters

---

## Where Exactly It Breaks

The failure occurs between steps 7–12.

The frontend loses filter state during data refresh and back navigation. Availability data updates independently from the filter layer, creating stale and inconsistent UI results.

---

## Root Cause Analysis

- Weak frontend state persistence
- Client-side filtering on stale cached data
- Live availability refresh not synchronized with filters
- Missing URL/query-state preservation
- Poor navigation state handling

---

# Problem 3 — Seat Selection Resets Randomly

## Severity
HIGH

## What is Broken

Users selecting preferred berths or seats often lose their selected seat while proceeding to passenger details. The system automatically changes the seat to “Auto” or assigns another berth.

This especially impacts families and elderly passengers.

---

## Affected Users

- Families booking together
- Elderly passengers requiring lower berths
- Disabled passengers
- Mobile users
- Users booking long-distance journeys

Estimated affected journeys:
30–40% of bookings involve berth preference selection.

---

## Frequency

Occurs intermittently.

Higher occurrence observed on mobile devices.

Approximate occurrence:
15–25% of seat selection sessions.

---

## Current User Flow — Step by Step

1. User selects train and quota
2. User proceeds to seat selection page
3. Seat map loads
4. User selects preferred lower berth
5. Selected seat turns highlighted
6. User clicks “Proceed”
7. Passenger details form opens
8. Seat preference changes to “Auto”
9. Original berth disappears from selection
10. User returns to seat map
11. Previously selected berth now appears unavailable
12. User continues booking with unwanted berth assignment

---

## Where Exactly It Breaks

The failure occurs between steps 5–8.

The seat selection state is not reliably preserved during route transitions. Mobile rendering and backend seat synchronization issues trigger state resets.

---

## Root Cause Analysis

- Weak frontend state synchronization
- Race conditions during berth allocation
- Mobile component re-render issues
- Inconsistent seat-locking mechanism
- Delayed backend seat confirmation

---

# Problem 4 — Payment Status Confusion After UPI Payment

## Severity
CRITICAL

## What is Broken

After successful UPI authorization, IRCTC frequently fails to provide immediate transaction confirmation. Users see indefinite loading spinners without knowing whether payment succeeded, failed, or is pending.

This creates panic and duplicate payment attempts.

---

## Affected Users

- UPI payment users
- Mobile banking users
- Users booking during peak traffic hours
- First-time digital payment users

Estimated impact:
Affects a large percentage of mobile bookings.

---

## Frequency

Occurs intermittently during peak booking periods.

More common during:

- Tatkal booking windows
- Heavy traffic periods
- Mobile browser sessions

---

## Current User Flow — Step by Step

1. User selects train
2. User enters passenger details
3. User proceeds to payment page
4. User selects UPI payment
5. User enters UPI ID
6. User approves payment in banking app
7. User returns to IRCTC page
8. Endless loading spinner appears
9. No confirmation message shown
10. User refreshes page in panic
11. User checks bank account for deduction
12. Ticket confirmation remains unclear
13. User retries payment fearing failure

---

## Where Exactly It Breaks

The failure occurs between steps 7–10.

The payment gateway callback status is not communicated clearly to the frontend. The UI lacks real-time transaction state visibility.

---

## Root Cause Analysis

- Weak payment reconciliation flow
- No real-time payment polling
- Missing transaction progress indicators
- Poor retry-safe payment architecture
- Delayed gateway callback handling

---

# Problem 5 — Session Timeout During Booking

## Severity
HIGH

## What is Broken

Users are automatically logged out during booking sessions without warning. Passenger information and booking progress are completely lost.

This forces users to restart the booking process from the beginning.

---

## Affected Users

- Senior citizens
- Slow typists
- First-time users
- Mobile users
- Users booking for multiple passengers

Estimated impact:
High among users taking longer to fill forms.

---

## Frequency

Occurs frequently after periods of inactivity or during high server load.

Approximate timeout range:
5–10 minutes.

---

## Current User Flow — Step by Step

1. User logs into account
2. User searches for train
3. User selects train and class
4. User fills passenger details slowly
5. User reviews journey information
6. User clicks “Continue”
7. Session expired popup appears
8. User redirected to login page
9. Passenger data is lost
10. User logs in again
11. Booking flow restarts from beginning

---

## Where Exactly It Breaks

The failure occurs between steps 6–8.

The session expires without proactive warning or auto-refresh. Booking state is not preserved locally or server-side.

---

## Root Cause Analysis

- Aggressive session timeout policy
- No autosave functionality
- Missing token refresh mechanism
- Weak booking state persistence
- No inactivity warning system

---

# Problem 6 — Mobile Booking Flow Freezes and Scroll Glitches

## Severity
HIGH

## What is Broken

The IRCTC mobile booking flow becomes laggy and unstable during passenger detail entry and payment stages. Keyboard interactions trigger layout shifts, freezing, and accidental taps.

This significantly slows mobile bookings.

---

## Affected Users

- Android users
- Low-end smartphone users
- Users on slower mobile networks
- Users booking via mobile browsers

Estimated impact:
Affects a major percentage of mobile traffic.

---

## Frequency

Occurs frequently during:

- High traffic periods
- Long booking forms
- Payment stage transitions

Observed consistently on lower-performance devices.

---

## Current User Flow — Step by Step

1. User opens IRCTC mobile website
2. User searches for train
3. User selects train and quota
4. Passenger details form loads
5. User taps input field
6. Mobile keyboard opens
7. Layout shifts unexpectedly
8. Scroll position jumps
9. Form freezes temporarily
10. User taps wrong field accidentally
11. Page becomes laggy during input
12. User refreshes or abandons booking

---

## Where Exactly It Breaks

The failure occurs between steps 6–10.

Frontend rendering becomes unstable during keyboard interactions and dynamic form updates on mobile devices.

---

## Root Cause Analysis

- Heavy DOM rendering
- Poor mobile optimization
- Weak responsive layout handling
- Excessive frontend re-renders
- Inefficient form rendering logic

---

# Cross-System Findings

| Area | Observation | Severity |
|---|---|---|
| Scalability | Tatkal traffic crashes booking pipeline | CRITICAL |
| Session Management | Booking state frequently lost | HIGH |
| Payment Reliability | Weak payment confirmation visibility | CRITICAL |
| Mobile UX | Major responsiveness issues | HIGH |
| State Management | Filters and seat selection frequently reset | HIGH |
| Accessibility | Elderly and first-time users struggle heavily | MEDIUM |

---

# Conclusion

IRCTC operates at enormous national scale but suffers from major reliability, usability, and system architecture problems that directly affect millions of users.

The most critical failures occur during:

- Tatkal booking
- Payment processing
- Session continuity
- Mobile booking interactions

These issues reduce user trust, increase booking abandonment, and create transaction anxiety.

Future engineering efforts should prioritize:

- Scalable booking infrastructure
- Resilient payment architecture
- Mobile-first optimization
- Transparent user feedback systems
- Persistent booking state management

---

# End of Audit Report

