# IRCTC Feature Specifications — Part B

Design Engineering Sprint | 6 Solutions to Critical Problems

---

# Feature Spec 1: Tatkal Virtual Queue System

## Problem Statement

At 10:00 AM daily, 20-40 lakh concurrent users attempt Tatkal bookings simultaneously, overwhelming the IRCTC backend. The system provides zero feedback about request status. Users see infinite spinners, then HTTP 502 errors. They refresh repeatedly, compounding server load. Session drops and payment uncertainty result. Tatkal completion rate during peak: ~40%.

**Reference:** Part A, Problem 1 — Tatkal Booking Crash at 10:00 AM

---

## Proposed Solution

Implement a virtual queue system with live user feedback. At 10:00 AM, instead of hammering the server directly, users enter a visible queue. The UI displays their queue position (e.g., "#4,281"), estimated wait time (e.g., "~9 minutes"), and real-time progress bar. When their position is called, they have 90 seconds to complete seat selection and payment. After 90 seconds without action, their slot expires and re-enters the queue.

**User sees:**
- Live countdown to queue opening
- Queue position with estimated wait
- Progress animation
- Clear next-step instructions
- Queue position updates every 5 seconds

---

## Technical Implementation Plan

### System Components Affected
- Frontend (React/Vue component for queue UI)
- Backend queue service (new microservice or Redis-based queue)
- WebSocket server for real-time position updates
- Database (queue state persistence)
- Payment gateway (slot expiry coordination)

### New Data Requirements

**New Queue Table/Collection:**
```
queue_entry {
  id: UUID
  user_id: int
  queue_position: int
  entered_at: timestamp
  slot_assigned_at: timestamp
  slot_expiry_at: timestamp
  status: enum [waiting, assigned, completed, expired]
  train_search_params: json (stored search criteria)
  seat_preference: json (pre-selected seats)
}
```

### API Changes

**New Endpoints:**

1. `POST /tatkal/queue/enter`
   - Request: `{ user_id, search_params, seat_preferences }`
   - Response: `{ queue_position, estimated_wait_seconds, queue_id }`

2. `GET /tatkal/queue/status/{queue_id}`
   - Response: `{ position, wait_time_remaining, slot_assigned, slot_expiry }`

3. `WebSocket /tatkal/queue/live/{queue_id}`
   - Streaming updates: `{ position, wait_time, status_change }`

4. `POST /tatkal/booking/confirm-slot`
   - Request: `{ queue_id, seat_selection, passenger_details }`
   - Response: `{ booking_confirmed, ticket_id }`

### Frontend State Changes

**New React Context:**
```
QueueContext {
  queueId: string
  position: number
  waitTime: number
  slotAssigned: boolean
  slotExpiry: timestamp
  selectedSeats: array
  isConnected: boolean (WebSocket)
}
```

**New Component:** `TatkalQueueScreen.jsx`
- Displays queue countdown
- Real-time position updates via WebSocket
- Pre-fills passenger details from cache
- Auto-opens seat selection when slot assigned

### Third-Party Services
- Redis or Apache Kafka for queue management (message broker)
- WebSocket provider (native or Socket.io)

### Database Changes
- Create `tatkal_queue` table
- Create index on `queue_position` and `user_id` for fast lookups

---

## Success Metrics

- Tatkal booking completion rate: 40% → 70% during peak hours
- Session drop rate during 10 AM: 35% → 5%
- Payment success rate: 65% → 85%
- User abandonment before payment: 45% → 15%
- Average booking time: 45 seconds → 90 seconds (includes queue wait)
- Server error rate during Tatkal: 15% → <1%

---

## Edge Cases & Constraints

1. **Queue overflow:** If queue grows beyond 1 million, implement secondary queues by train route
2. **Slot expiry during payment:** User loses slot mid-payment. Show clear message: "Slot expired. Returning to queue. Your data is saved."
3. **Network disconnection:** WebSocket reconnects automatically; position preserved
4. **Railway backend unavailability:** Queue pauses; users see "Temporary hold. Try again in 30 seconds."
5. **Payment gateway lag:** Extend slot expiry to 120 seconds if payment processing slow
6. **Mobile network drop:** Queue position cached locally; resumes when connection restored

---

---

# Feature Spec 2: Search Filter Persistence & Live Sync

## Problem Statement

Search filters on train results frequently fail to apply correctly. Users select "Available Only" but still see waitlisted trains. Filters reset when users navigate back. This forces manual scanning of 20-40 trains repeatedly, adding 8-15 minutes to the booking process. Affects all search users; hits hardest on first-time users and elderly users who need specific filters.

**Reference:** Part A, Problem 2 — Search Filters Do Not Work Reliably

---

## Proposed Solution

Persist filter state in URL query parameters and localStorage. When availability data refreshes from the backend, reapply filters client-side automatically. Display applied filters prominently so users know what is active. Add "Clear Filters" button. When users navigate back from train details, filters remain active and visible.

**User sees:**
- Applied filters displayed as blue chips/tags
- Trains update in real-time as new data arrives
- Filters persist across back navigation
- Visual indicator: "Showing 8 of 40 trains"
- One-click "Clear All Filters"

---

## Technical Implementation Plan

### System Components Affected
- Frontend (React filter component)
- Redux/Zustand state management
- Browser localStorage API
- URL query string management
- Backend search API (minimal changes)

### New Data Requirements

**Browser localStorage schema:**
```
{
  filters: {
    class: ["Sleeper", "AC First Class"],
    quota: ["General"],
    availability: ["Available"],
    departure_time: ["06:00-12:00"],
    train_type: ["Express", "Rajdhani"]
  },
  search_params: {
    from: "New Delhi",
    to: "Mumbai Central",
    date: "2026-05-15"
  }
}
```

### URL Structure

```
/search?from=New+Delhi&to=Mumbai&date=2026-05-15&class=Sleeper&class=AC&quota=General&available_only=true
```

### Frontend State Changes

**New Redux Slice:**
```
searchFiltersSlice {
  activeFilters: object
  availableFilterOptions: object
  filteredTrainCount: number
  totalTrainCount: number
}
```

**New Component:** `FilterPersistence.jsx`
- Reads URL query params on mount
- Listens to filter changes
- Updates URL without page reload
- Saves to localStorage
- Syncs across browser tabs

### Backend API Changes (Minimal)

No new endpoints needed. Existing `/search` endpoint already returns complete train data. Frontend applies filters client-side.

### Real-time Filter Re-application

```
When availability data updates (every 30 seconds):
1. Receive fresh train list from backend
2. Check if any filters are active
3. Re-apply filters to new data
4. Update UI with filtered results
5. Show: "Results updated 2 seconds ago"
```

### Third-Party Services
None required.

---

## Success Metrics

- Filter trust score: 40% → 85% (survey: "Do you trust the filters?")
- Time to select train: 12 minutes → 4 minutes
- Back navigation filter retention: 0% → 95%
- User filter application rate: 30% → 65%
- Search repeat rate: 45% → 10%

---

## Edge Cases & Constraints

1. **Conflicting filters:** If "Available Only" + specific date have no trains, show: "No trains available with these filters. Clear and try again."
2. **Filter-induced no results:** Display suggestion: "Broaden filters to see results" with one-click buttons
3. **Stale filter state:** If user searches different route while old filters active, clear old filters automatically
4. **Mobile screen space:** Collapse filter chips into "3 active filters" pill; expand on tap
5. **Railway backend API doesn't support some filters:** Show disabled state with tooltip: "This filter temporarily unavailable"

---

---

# Feature Spec 3: Persistent Seat Selection with State Locking

## Problem Statement

Users select specific seats (lower berths for elderly, specific sides for comfort) during booking. The selection disappears when proceeding to passenger details. The system assigns "Auto" or a different seat instead. Affects 30-40% of all bookings involving seat preferences. Disproportionately impacts elderly travelers and families.

**Reference:** Part A, Problem 3 — Seat Selection Resets Randomly

---

## Proposed Solution

Lock selected seats server-side immediately after user selection. Store seat selection in a dedicated session state that survives navigation. Pass seat selection explicitly to backend before payment. Add visual confirmation: "Lower berth reserved for your elderly passenger" before proceeding.

**User sees:**
- Selected seats highlighted and marked as "Reserved for you"
- Confirmation screen before proceeding: "You've selected Lower Berth 24"
- Seat state preserved across passenger details form
- Clear error message if seat becomes unavailable: "This berth was released. Choose another."

---

## Technical Implementation Plan

### System Components Affected
- Frontend seat map component (React)
- Backend seat booking service
- Database seat allocation table
- Session management
- Mobile-specific fixes

### New Data Requirements

**New Seat Reservation Table:**
```
seat_reservation {
  id: UUID
  user_session_id: string
  train_id: int
  coach_id: string
  seat_number: string
  reserved_at: timestamp
  reservation_expiry: timestamp (15 minutes)
  status: enum [reserved, confirmed, expired, released]
  pax_preference: string (e.g., "elderly", "child", "disabled")
}
```

**Session state addition:**
```
{
  booking_session: {
    train_id: int,
    reserved_seats: [
      { seat_id: "24", coach: "A1", pax_type: "elderly" }
    ],
    reserved_at: timestamp
  }
}
```

### API Changes

**New/Modified Endpoints:**

1. `POST /seat/reserve`
   - Request: `{ user_id, train_id, seat_id, pax_type }`
   - Response: `{ reservation_id, expiry_timestamp, success: true }`

2. `GET /seat/reservation-status/{reservation_id}`
   - Response: `{ seat_id, status, expiry_time_remaining }`

3. `POST /booking/confirm-seats`
   - Request: `{ reservation_id, passenger_details }`
   - Response: `{ confirmed: true, ticket_id }`

### Frontend State Changes

**React Context:**
```
SeatContext {
  selectedSeats: array
  reservationId: string
  reservationExpiry: timestamp
  paxPreferences: object
  seatMapState: object
}
```

**New Component:** `SeatSelectionLock.jsx`
- Calls `/seat/reserve` immediately on click
- Displays reservation expiry countdown
- Shows "Reserved for you" badge
- Handles reservation expiry gracefully

### Mobile-Specific Fixes

**Problem:** Mobile component re-renders lose state during keyboard dismiss
**Solution:**
- Use `useRef` hook to persist state beyond component lifecycle
- Store selected seats in sessionStorage immediately
- Add `useMemo` to prevent unnecessary seat map re-renders
- Disable auto-keyboard-close on selection

### Database Changes
- Create `seat_reservation` table with indexes on `user_session_id`, `train_id`, `reservation_expiry`
- Add `reserved_seats` JSON field to `booking_session` table

---

## Success Metrics

- Seat selection persistence: 75% → 98%
- User frustration with seat change: High → Minimal
- Seat selection accuracy (user gets chosen seat): 65% → 92%
- Mobile seat selection success: 55% → 88%
- Booking completion after seat selection: 70% → 88%

---

## Edge Cases & Constraints

1. **Reservation expiry during form fill:** Show countdown timer + "Expiring in 3 minutes" warning. Auto-refresh if user taps a button.
2. **Seat becomes unavailable after reservation:** Show clear error: "This berth was just booked by another passenger. Choose from available options." with updated seat map.
3. **User selects multiple seats for family:** Reserve all seats together. If any single seat fails, release all and show error.
4. **Mobile keyboard dismiss causes component unmount:** Store selected seats in sessionStorage; restore on remount.
5. **Backend seat lock expires before payment:** Extend lock at key checkpoints (after passenger details, before payment).
6. **Conflicting seat selections (two users choosing same seat simultaneously):** First-come-first-served. Loser gets "seat unavailable" message with refresh option.

---

---

# Feature Spec 4: Real-Time Payment Status Indicator

## Problem Statement

After UPI payment authorization, users see indefinite loading spinners with zero feedback about payment status. They don't know if payment succeeded, failed, or is processing. This creates panic and duplicate payment attempts. Payment uncertainty affects user trust and creates refund chaos. Estimated impact: 30-40% of failed bookings originate from payment status confusion.

**Reference:** Part A, Problem 4 — Payment Status Confusion After UPI Payment

---

## Proposed Solution

Implement real-time payment status polling with live progress indicators. Instead of a blank spinner, show step-by-step feedback: "Authorization sent → Waiting for bank → Payment confirmed → Processing ticket." If status is uncertain after 30 seconds, show clear action: "Confirm payment status?" with option to check or retry.

**User sees:**
- Live multi-step progress: "Processing: Step 2 of 4"
- Status message updates: "Payment confirmed by your bank"
- Countdown timer: "Confirming within 60 seconds"
- If delayed: Clear CTA: "Check payment status" or "Try a different method"
- Never: Blank spinner with zero context

---

## Technical Implementation Plan

### System Components Affected
- Frontend payment component (React)
- Backend payment reconciliation service
- Payment gateway integration layer
- Database payment tracking
- WebSocket server for real-time updates

### New Data Requirements

**Enhanced Payment Transaction Table:**
```
payment_transaction {
  id: UUID
  user_id: int
  booking_id: int
  gateway_transaction_id: string
  amount: float
  currency: string
  status: enum [initiated, authorized, processing, confirmed, failed, refunded]
  status_updated_at: timestamp
  status_history: array [
    { status, timestamp, reason }
  ]
  payment_method: enum [upi, card, netbanking]
  gateway_response: json
  user_notification_sent: boolean
  created_at: timestamp
}
```

### API Changes

**New Endpoints:**

1. `POST /payment/initiate`
   - Request: `{ user_id, booking_id, amount, method }`
   - Response: `{ transaction_id, gateway_redirect_url }`

2. `GET /payment/status/{transaction_id}`
   - Response: `{ status, step, message, last_updated_seconds_ago }`

3. `WebSocket /payment/live/{transaction_id}`
   - Streaming: `{ status, step: 1-4, message, remaining_seconds }`

4. `POST /payment/confirm-manual`
   - Request: `{ transaction_id }`
   - Response: `{ confirmed: true, ticket_id }` or `{ confirmed: false, reason }`

### Frontend State Changes

**React Context:**
```
PaymentContext {
  transactionId: string
  status: enum
  currentStep: 1-4
  statusMessage: string
  pollInterval: number (2 seconds)
  wsConnected: boolean
  nextActionCTA: string
}
```

**New Component:** `PaymentStatusTracker.jsx`
- Establishes WebSocket connection on mount
- Falls back to polling if WebSocket unavailable
- Displays step-by-step progress
- Handles timeout gracefully
- Shows "Check Status" CTA after 30 seconds without update

### Payment Polling Strategy

```
Frontend polling logic:
1. User completes UPI authorization
2. Frontend shows "Step 1: Authorization sent"
3. Poll /payment/status every 2 seconds
4. Update UI with new status
5. If status unchanged for 30 seconds → show "Check status?" CTA
6. If status = "confirmed" → redirect to ticket page
7. If status = "failed" → show retry option
8. Max poll attempts: 120 (4 minutes)
```

### Backend Payment Reconciliation

```
Backend logic:
1. Receive callback from payment gateway
2. Update payment_transaction.status
3. Trigger booking confirmation if status = "confirmed"
4. Notify frontend via WebSocket
5. If no callback received after 45 seconds → mark as "processing"
6. Retry gateway status check every 10 seconds
```

### Third-Party Services
- Primary payment gateway API (Razorpay, Instamojo, etc.)
- WebSocket provider for real-time updates
- Redis for transaction state caching

---

## Success Metrics

- Payment status certainty: 30% → 95% (users know their status)
- Panic refund attempts: 25% → <5%
- Duplicate payment attempts: 18% → <2%
- Payment success rate: 65% → 87%
- User satisfaction with payment flow: 3.2/5 → 4.6/5
- Booking completion after payment page: 55% → 78%

---

## Edge Cases & Constraints

1. **Gateway callback never arrives:** After 45 seconds, show: "Payment status unclear. Check your bank account. If amount deducted, contact support." Generate support ticket automatically.

2. **User closes browser during payment:** Store transaction ID in localStorage. On return, restore transaction status from backend.

3. **Slow network connection:** Extend polling timeout from 2 to 5 seconds. Show: "Slow connection. Status updates may be delayed."

4. **Gateway API timeout:** Fallback to "last known status" and show: "Unable to reach bank. Try again in 30 seconds."

5. **Payment succeeds but booking fails:** Show: "Payment confirmed but booking encountered error. Retry?" Ensure idempotency.

6. **UPI decline after authorization:** Show: "Payment declined by bank. Reason: [reason]. Try another method."

7. **Multiple payment attempts in flight:** Lock payment section. Only one attempt can proceed simultaneously.

---

---

# Feature Spec 5: Session Persistence with Auto-Save

## Problem Statement

Users are logged out abruptly during booking without warning. All passenger data, seat selections, and journey preferences are lost. Users must restart from scratch. Session timeout is set to 5-10 minutes, which is too short for multi-passenger bookings. This creates high abandonment during high-load periods when server responsiveness is slow.

**Reference:** Part A, Problem 5 — Session Timeout During Booking

---

## Proposed Solution

Implement automatic session refresh while user is actively booking. If inactivity detected, show 60-second warning: "Session expiring in 60 seconds. Click to stay logged in." Auto-save passenger data to a recoverable session state every 30 seconds. If session expires, allow recovery: "Your booking session expired but we saved your data. Continue booking?"

**User sees:**
- "Session active for 18 more minutes" indicator
- No sudden logout during booking
- If expiry approaching: "Stay logged in?" prompt (1 tap)
- If logout happens: "Resume your booking" option with pre-filled data

---

## Technical Implementation Plan

### System Components Affected
- Frontend session management (React hooks)
- Backend JWT/session token refresh logic
- Redis session store
- Browser sessionStorage API
- WebSocket heartbeat system

### New Data Requirements

**New Recoverable Session Table:**
```
recoverable_booking_session {
  id: UUID
  user_id: int
  session_token: string
  booking_state: json {
    search_params: object,
    selected_train: object,
    selected_seats: array,
    passenger_details: array,
    seat_preferences: object,
    saved_at: timestamp
  }
  expires_at: timestamp
  recovery_attempts: int
  created_at: timestamp
}
```

### API Changes

**New Endpoints:**

1. `POST /session/refresh`
   - Request: `{ current_token }`
   - Response: `{ new_token, expires_in_seconds }`

2. `GET /session/recoverable-state`
   - Response: `{ booking_state, expired_at, can_recover: boolean }`

3. `POST /session/recover`
   - Request: `{ recovery_session_id }`
   - Response: `{ booking_state, new_token }`

### Frontend State Changes

**React Hooks:**

```javascript
useSessionManagement() {
  // Auto-refresh token every 5 minutes
  // Show warning at 1 minute before expiry
  // Save booking state every 30 seconds
  // Detect inactivity and warn
}

useAutoSaveBookingState() {
  // Saves to sessionStorage and backend every 30 seconds
  // Triggers on: passenger detail change, seat selection, etc.
}
```

**New Component:** `SessionWarning.jsx`
- Displays countdown to expiry
- "Stay logged in?" button
- "Save and logout" option for users who want to pause

### Session Timeout Logic

```
Default settings:
- Session timeout: 30 minutes (instead of 5-10)
- Auto-refresh threshold: 15 minutes
- Inactivity warning: 1 minute before expiry
- Booking state auto-save: Every 30 seconds
- Maximum recovery window: 24 hours
```

### Database Changes
- Create `recoverable_booking_session` table
- Add indexes on `user_id`, `expires_at`, `created_at`
- Add cleanup job to delete expired sessions after 24 hours

---

## Success Metrics

- Session dropout during booking: 35% → <5%
- Users completing booking after warning: 0% → 85%
- Booking completion rate: 70% → 85%
- Session timeout related support tickets: 200/day → <10/day
- Average time to complete booking: Unchanged (users not rushed)
- User trust in session stability: 2.5/5 → 4.5/5

---

## Edge Cases & Constraints

1. **Token refresh fails (backend unreachable):** Show: "Connection issue. Session will expire in X minutes. Complete booking now." Allow offline completion.

2. **User inactive for 30 minutes (beyond booking):** Clear recoverable state. Don't auto-resume if user abandoned booking.

3. **Multiple tabs open:** Share session state across tabs using `localStorage` events. If session lost in one tab, reflect in all.

4. **User logs out manually:** Delete recoverable session. Don't prompt recovery on re-login.

5. **Recovery session already used:** Show: "This session was already used. Start fresh booking."

6. **Booking state corrupted in storage:** Fallback to pre-booking page. Let user start over from search.

7. **User device goes offline:** Cache booking state locally. Resume when online.

---

---

# Feature Spec 6: Mobile-Optimized Responsive Booking Form

## Problem Statement

IRCTC mobile booking becomes laggy and unstable during passenger detail entry and payment. Keyboard interactions trigger unexpected layout shifts. Form becomes unresponsive. Scroll position jumps. This affects a large percentage of mobile traffic, disproportionately impacting Android users and users on slow networks. Mobile booking completion rate: ~45% vs desktop 75%.

**Reference:** Part A, Problem 6 — Mobile Booking Flow Freezes and Scroll Glitches

---

## Proposed Solution

Rebuild mobile booking forms with performance-first architecture. Implement virtual scrolling for long forms. Optimize re-renders using React.memo and useMemo. Use CSS containment to isolate layout impact. Pre-render keyboard layouts to avoid reflow. Test on low-end devices (Android 5+, 2GB RAM). Measure and optimize interaction-to-paint time below 100ms.

**User sees:**
- Smooth keyboard interactions without layout jumps
- Fast input response (no lag between tap and text appearing)
- Form remains responsive even on slow devices
- Scroll position preserved when keyboard opens/closes
- Clear visual hierarchy on narrow screens

---

## Technical Implementation Plan

### System Components Affected
- Frontend passenger form component (React)
- CSS and layout system
- Mobile-specific event handling
- Performance monitoring
- Browser DevTools integration

### Frontend Optimization Strategy

**1. Component Structure:**
```
PassengerForm (parent, memoized)
├── PassengerCard (memo, independent)
│   ├── NameInput (memo, virtual scroll)
│   ├── AgeInput (memo)
│   └── GenderSelect (memo)
├── SeatPreferenceSection (memo)
└── SubmitButton (memo)
```

**2. Prevent Unnecessary Re-renders:**
```javascript
// Use React.memo for child components
const PassengerCard = React.memo(({passenger, onChange}) => {...})

// Use useMemo for derived data
const memoizedValidation = useMemo(
  () => validatePassenger(passenger),
  [passenger]
)

// Use useCallback for event handlers
const handleNameChange = useCallback(
  (e) => onChange({...passenger, name: e.target.value}),
  [passenger, onChange]
)
```

**3. CSS Containment:**
```css
.passenger-card {
  contain: layout style paint;
  /* Isolates layout recalculations */
}

.form-input {
  contain: layout;
  /* Prevents input from affecting siblings */
}
```

**4. Virtual Scrolling (for 5+ passengers):**
```javascript
// Import windowing library
import { FixedSizeList } from 'react-window'

// Only render visible passengers, not all
<FixedSizeList
  height={600}
  itemCount={passengers.length}
  itemSize={200}
>
  {PassengerCard}
</FixedSizeList>
```

**5. Input Optimization:**
```javascript
// Debounce onChange events
const debouncedChange = useMemo(
  () => debounce((value) => onChange(value), 300),
  []
)

// Prevent keyboard from pushing content
.form-container {
  position: relative;
  padding-bottom: env(safe-area-inset-bottom);
}
```

### Mobile Testing Matrix

**Test on these devices:**
- Android 5.x (2GB RAM, 800x480)
- Android 7.x (3GB RAM, 1080x1920)
- Android 11+ (6GB RAM, 2400x1080)
- iPhone SE (2GB RAM, 375x667)
- iPhone 12+ (4GB+ RAM, 1170x2532)

**Measure these metrics:**
- Input interaction to text appearance: <100ms
- Form scroll to full page paint: <200ms
- Keyboard open to form repositioning: <300ms
- Page navigation time: <500ms

### Performance Monitoring

**Add to Production:**
```javascript
// Log interaction-to-paint time
const observer = new PerformanceObserver((list) => {
  list.getEntries().forEach((entry) => {
    if (entry.duration > 100) {
      console.warn('Slow interaction:', entry)
      sendToAnalytics(entry)
    }
  })
})
```

### Database/API Changes
None required. Form structure unchanged; only frontend optimization.

---

## Success Metrics

- Mobile booking completion rate: 45% → 70%
- Form interaction lag: High → <100ms
- Layout shift during keyboard: Frequent → Rare (<1%)
- Mobile abandonment rate: 35% → 15%
- Page speed score (Lighthouse): 35 → 85
- User satisfaction with mobile: 2.8/5 → 4.2/5
- Android-specific completion: 40% → 68%

---

## Edge Cases & Constraints

1. **Extremely long passenger list (15+ people):** Use virtual scrolling to render only visible passengers. Cache offscreen data.

2. **Very slow network (2G):** Debounce form submissions. Show offline capability: "Form saved locally. Complete when online."

3. **Low device memory (2GB RAM):** Reduce DOM complexity. Lazy-load passenger cards. Clear old form data from memory.

4. **Concurrent keyboard + scroll:** Lock scroll while keyboard open. Unlock on keyboard dismiss.

5. **Third-party scripts (analytics, ads):** Defer non-critical scripts until after form interaction. Use async loading.

6. **Browser DevTools open (mobile):** Performance may appear worse than reality. Measure without DevTools.

---

---

# Wireframe Descriptions

## Wireframe 1 — Tatkal Queue Screen (Mobile)

```
─────────────────────────────────────
[ IRCTC Logo ]          [Profile icon]
─────────────────────────────────────

TATKAL BOOKING OPENS IN
┌───────────────────────┐
│   04 : 23 : 11        │ ← Live countdown
└───────────────────────┘

YOUR QUEUE POSITION
┌─────────────────────────────────┐
│  #4,281 OUT OF 840,500          │
│  Est. wait: 9 minutes           │
│  ████████░░░░░░░░░░  Progress   │
└─────────────────────────────────┘

TRAIN DETAILS (Saved)
✓ 12622 Tamil Nadu Express
✓ AC Sleeper | 2 Passengers
✓ Seats: Lower Berth Saved

[ Change Train ]  [ View Passengers ]

─────────────────────────────────────
WHAT HAPPENS NEXT:
1. Wait for your position
2. Your turn: You'll get 90 secs
3. Select seat → Enter details → Pay
4. Ticket confirmed
─────────────────────────────────────

Tip: Your preferences are saved.
```

---

## Wireframe 2 — Search Results with Persistent Filters

```
─────────────────────────────────────
[ New Delhi ]   →   [ Mumbai ]
[ 15 May 2026 ]      [ 1 Passenger ]
[ SEARCH ]
─────────────────────────────────────

ACTIVE FILTERS (Clear all)
[x Sleeper]  [x Available]  [x Rajdhani]

Showing 8 of 40 trains

Train 1: 12622 Tamil Nadu Express
Dep: 06:00 | Dur: 12h 30m
Sleeper: Available 4 | ₹450
Class: ✓ AC First | AC 2-Tier | Sleeper

Train 2: 16504 Intercity Express
Dep: 08:30 | Dur: 14h 15m
Sleeper: WL 2 (Filtered out)

[Results updated 10 seconds ago]
─────────────────────────────────────
```

---

## Wireframe 3 — Seat Selection with Lock Indicator

```
Berth Selection — Coach A1

[BERTH MAP]
┌─────────────────────────┐
│  1(L) 2(U)  3(L) 4(U)   │
│  5(L) 6(U)  7(L) 8(U)   │
│  9(L) 10(U) 11(L) 12(U) │
│  13(L) 14(U) 15(L) 16(U)│
└─────────────────────────┘

Color Legend:
■ Available  ■ Booked  ■ Selected  ■ Reserved for you

Selected: Lower Berth 24
Reserved for: Elderly passenger
Duration: 15 minutes remaining

[Change Selection]  [Confirm & Continue]
─────────────────────────────────────
```

---

## Wireframe 4 — Payment Status Tracker

```
PAYMENT IN PROGRESS

Step 1: Authorization Sent ✓
Step 2: Processing (Current)
Step 3: Bank Confirmation ⏳
Step 4: Ticket Confirmation ⏳

Status: "Your bank is processing..."
Confirm within: 45 seconds

[Refresh Status]
[Try Different Method]

─────────────────────────────────────
Issue? Your payment may still be
processing. Check status in 30 seconds.
```

---

## Wireframe 5 — Session Warning

```
SESSION EXPIRING SOON

Your session will expire in
[ 60 seconds ]

What happens:
- Logout automatically
- Your booking form will be saved
- You can resume within 24 hours

[ Stay Logged In ]  [ Save & Logout ]
```

---

## Wireframe 6 — Mobile Form (Optimized)

```
PASSENGER DETAILS

Passenger 1
┌─────────────────────────┐
│ Full Name               │
│ [__________________]    │
│                         │
│ Age                     │
│ [__]                    │
│                         │
│ Gender                  │
│ [Male ▼]                │
└─────────────────────────┘

Passenger 2
[ + Add Another Passenger ]

─────────────────────────
[ Save Details ]  [ Continue ]
```

---

# Summary

- **6 Feature Specs:** Fully detailed with problem, solution, technical plan, metrics
- **6 Wireframes:** Text-based mid-fidelity layouts showing before/after
- **All linked to Part A:** Every spec references specific Part A problem
- **Production-ready:** Include edge cases, constraints, success metrics
- **Mobile-first:** Optimization for Indian users on slower networks

Next: AI Feature Spec + 2×2 Matrix + Peer Review Updates
