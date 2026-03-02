# CancelBooking Mutation — Implementation Plan

## Context

The Airlock booking app has no cancellation flow. The `BookingStatus` enum only has `CURRENT`, `COMPLETED`, `UPCOMING`. When a guest books via `createBooking`, funds are deducted and a booking is created — but there's no way to cancel or get a refund. This plan adds a full cancel-and-refund flow across the server and client repos.

## Repos & Files

**Server (`odyssey-voyage-II-server`) — 6 files (3 pairs, monolith/ + final/monolith/):**

| File | Change |
|------|--------|
| `monolith/schema.graphql` | Add `CANCELLED` enum value, `cancelBooking` mutation, `CancelBookingResponse` type |
| `monolith/datasources/bookings.js` | Add `updateBookingStatus()` method, fix `isListingAvailable()` to exclude CANCELLED |
| `monolith/resolvers.js` | Add `cancelBooking` resolver |
| `final/monolith/schema.graphql` | Same (identical file) |
| `final/monolith/datasources/bookings.js` | Same (identical file) |
| `final/monolith/resolvers.js` | Same (identical file) |

**Client (`odyssey-voyage-II-client`) — 2 files:**

| File | Change |
|------|--------|
| `src/pages/trips.js` | Add `id` and `totalPrice` to `GUEST_TRIPS` query |
| `src/components/Trips.js` | Add `CANCEL_BOOKING` mutation, cancel button, AlertDialog confirmation, toast feedback |

---

## Step 1: Add CANCELLED to BookingStatus Enum

**Files:** `monolith/schema.graphql` (line 299), `final/monolith/schema.graphql`

```graphql
enum BookingStatus {
  CURRENT
  COMPLETED
  UPCOMING
  CANCELLED
}
```

No DB migration needed — the Sequelize model uses `DataTypes.STRING` with no enum constraint.

## Step 2: Add cancelBooking Mutation + Response Type to Schema

**Files:** `monolith/schema.graphql`, `final/monolith/schema.graphql`

Add to `type Mutation` block (after `createBooking`):
```graphql
"Cancels an upcoming booking and refunds the guest"
cancelBooking(bookingId: ID!): CancelBookingResponse!
```

Add response type (after `CreateBookingResponse`):
```graphql
"The response after cancelling a booking."
type CancelBookingResponse implements MutationResponse {
  code: Int!
  success: Boolean!
  message: String!
}
```

Follows the `MutationResponse` interface pattern used by all other mutations. Single `bookingId` arg — no input type needed (matches `submitGuestReview(bookingId: ID!, ...)` pattern).

## Step 3: Add updateBookingStatus to BookingsDb Datasource

**Files:** `monolith/datasources/bookings.js`, `final/monolith/datasources/bookings.js`

Add method to `BookingsDb` class:
```javascript
async updateBookingStatus({ bookingId, status }) {
  const booking = await this.db.Booking.findByPk(bookingId);
  if (!booking) {
    throw new Error('Booking not found');
  }
  booking.status = status;
  await booking.save();
  return booking.dataValues;
}
```

Reuses existing `findByPk` pattern from `getBooking()`. General-purpose for future status transitions.

## Step 4: Fix isListingAvailable to Exclude CANCELLED Bookings

**Files:** `monolith/datasources/bookings.js`, `final/monolith/datasources/bookings.js`

Current `isListingAvailable` finds ALL bookings with overlapping dates regardless of status. Cancelled bookings would still block dates. Add `Op.in` filter:

```javascript
const { between, or, in: opIn } = this.db.Sequelize.Op;
// Add to where clause:
status: { [opIn]: ['UPCOMING', 'CURRENT'] },
```

Uses `Op.in` (not `Op.or`) to avoid key collision with the existing date range `[or]`.

## Step 5: Implement cancelBooking Resolver

**Files:** `monolith/resolvers.js`, `final/monolith/resolvers.js`

Add after `createBooking` resolver (line 203). Flow:

1. **Auth check** — `if (!userId) throw AuthenticationError()` (matches all mutations)
2. **Fetch booking** — `dataSources.bookingsDb.getBooking(bookingId)` (existing method)
3. **Ownership check** — `booking.guestId !== userId` → 403
4. **Status guard** — `booking.status !== "UPCOMING"` → 400
5. **Refund** — `dataSources.paymentsAPI.addFunds({ userId, amount: booking.totalCost })` (existing method, same API used by `addFundsToWallet` resolver)
6. **Update status** — `dataSources.bookingsDb.updateBookingStatus({ bookingId, status: "CANCELLED" })`
7. **Return** `{ code: 200, success: true, message: "..." }`

Refund happens before status update — if refund fails, booking stays UPCOMING (no partial state). Mirrors `createBooking` which does payment first, then creates record.

### Resolver Code

```javascript
cancelBooking: async (_, { bookingId }, { dataSources, userId }) => {
  if (!userId) throw AuthenticationError();

  const booking = await dataSources.bookingsDb.getBooking(bookingId);
  if (!booking) {
    return { code: 404, success: false, message: "Booking not found." };
  }
  if (booking.guestId !== userId) {
    return { code: 403, success: false, message: "You do not have permission to cancel this booking." };
  }
  if (booking.status !== "UPCOMING") {
    return { code: 400, success: false, message: "Only upcoming bookings can be cancelled." };
  }

  try {
    await dataSources.paymentsAPI.addFunds({ userId, amount: booking.totalCost });
  } catch (e) {
    return { code: 400, success: false, message: "We couldn't process your refund. Please try again later." };
  }

  try {
    await dataSources.bookingsDb.updateBookingStatus({ bookingId, status: "CANCELLED" });
    return { code: 200, success: true, message: "Your booking has been cancelled and a full refund has been issued." };
  } catch (err) {
    return { code: 400, success: false, message: err.message };
  }
},
```

## Step 6: Add id and totalPrice to GUEST_TRIPS Query

**File:** `src/pages/trips.js`

The `GUEST_TRIPS` query is currently missing `id` (unlike `PAST_GUEST_TRIPS` which has it). Add `id` and `totalPrice`:

```javascript
upcomingGuestBookings {
  id              // NEW — needed for cancel mutation
  totalPrice      // NEW — shown in refund confirmation
  checkInDate
  checkOutDate
  ...
```

## Step 7: Add Cancel Button + Confirmation Dialog to Trips.js

**File:** `src/components/Trips.js`

**New imports:** `useRef`, `useState`, `useMutation`, `AlertDialog` + related Chakra components, `useToast`

**New mutation constant:**
```javascript
export const CANCEL_BOOKING = gql`
  mutation CancelBooking($bookingId: ID!) {
    cancelBooking(bookingId: $bookingId) { success message }
  }
`;
```

**In `Trip` component:**
- `useMutation(CANCEL_BOOKING, { variables, refetchQueries: [{query: GUEST_TRIPS}], onCompleted, onError })`
- Cancel button appears only for `UPCOMING` trips (not `CURRENT`) — extends existing status ternary
- `AlertDialog` confirmation shows listing title and refund amount
- `useToast` for success/error feedback (appropriate since the card disappears from list after refetch)
- Loading spinners on both the button and dialog confirm button to prevent double-submission

---

## Implementation Order

1. Server schema (Steps 1-2) — both `monolith/` and `final/monolith/`
2. Server datasource (Steps 3-4) — both directories
3. Server resolver (Step 5) — both directories
4. **Test server** via GraphQL Playground before touching client
5. Client query fix (Step 6)
6. Client UI (Step 7)
7. End-to-end verification

## Verification

1. **Schema:** Run server → check GraphQL Playground for `cancelBooking` mutation
2. **Auth guard:** Cancel without login → `AuthenticationError`
3. **Ownership:** Cancel another guest's booking → 403
4. **Status guard:** Cancel a COMPLETED/CURRENT booking → 400
5. **Happy path:** Cancel UPCOMING booking → 200, status = CANCELLED, wallet refunded
6. **Date availability:** Cancelled dates become bookable again
7. **UI flow:** Cancel button → dialog → confirm → toast → trip disappears from list
8. **Loading states:** Spinners during mutation execution
9. **Error handling:** Network failure → error toast, dialog closes
