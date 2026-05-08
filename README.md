# IRCTC Sprint — Fix IRCTC Before It Crashes

A comprehensive design engineering and AI feature sprint to audit and rescue the Indian Railways Catering and Tourism Corporation (IRCTC) platform.

**Status:** Part A & Part B — COMPLETE

---

## Project Overview

This is a two-part design sprint aimed at documenting critical IRCTC issues and proposing evidence-based solutions:

- **Part A:** Problem Discovery — Document 6 real pain points through live platform exploration
- **Part B:** Solution Design — Create feature specs, wireframes, AI proposals, and prioritization matrix

### The Scale
- **8 Cr+** Registered Users
- **12L** Tickets booked daily
- **₹400 Cr** Daily transaction value
- **10:00 AM** Tatkal crash (daily recurring issue)

---

## Repository Structure

```
irctc-sprint/
├── README.md                    ← You are here
├── part-a/
│   └── PROBLEMS.md             ← 6 documented problems (given + self-discovered)
├── part-b/
│   ├── SPECS.md                ← Feature specifications
│   ├── AI-FEATURE.md           ← AI proposal
│   └── MATRIX.md               ← Prioritization matrix
└── assets/
    └── screenshots/             ← Evidence screenshots from live platform
```

---

## Part A — Problem Discovery (Current Phase)

### Deliverables Checklist

- [ ] **Problem 1:** Tatkal Booking Crashes at 10:00 AM [Given]
- [ ] **Problem 2:** Search Filters Do Not Work Reliably [Given]
- [ ] **Problem 3:** Seat Selection Resets Randomly [Given]
- [ ] **Problem 4:** Self-Discovered Problem (Different from above)
- [ ] **Problem 5:** Self-Discovered Problem (Different from above)
- [ ] **Problem 6:** Self-Discovered Problem (Different from above)

### Documentation Requirements for Each Problem

Each of the 6 problems must be documented with:

1. **What is broken** — Specific, concrete description
2. **Affected users** — Named segment with quantification
3. **Frequency** — How often it occurs (daily, intermittent, peak-hours-only, mobile-only, etc.)
4. **Current flow — step by step** — 6-10 numbered steps starting from user action
5. **Where exactly it breaks** — Specific step numbers and technical reason for failure

### Research Areas for Self-Discovery

Explore these areas on **irctc.co.in** (live platform):

- Mobile experience vs desktop
- PNR status and journey information
- Accessibility and assistance booking
- Payment and refund experience
- Waitlist and notification system
- Account and profile management

### File Location

All 6 problems are documented in [part-a/PROBLEMS.md](part-a/PROBLEMS.md)

---

## Part B — Solution Design (Upcoming)

Once Part A is complete, Part B will contain:

- **SPECS.md** — Detailed feature specifications for each problem
- **AI-FEATURE.md** — AI proposal addressing one key pain point
- **MATRIX.md** — 2×2 prioritization matrix (impact vs effort)

---

## Evidence & Screenshots

All screenshots from live IRCTC platform exploration are stored in:
```
assets/screenshots/
```

Reference format in PROBLEMS.md:
```
![Screenshot description](../../assets/screenshots/screenshot-name.png)
```

---

## Getting Started

### Prerequisites
- Access to [irctc.co.in](https://irctc.co.in) (live platform)
- Git initialized (already done)
- GitHub account with public repository