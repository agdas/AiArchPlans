# CancelBooking Mutation — Implementation Reference

## Repos Involved

| Repo | What Changes | Why |
|------|-------------|-----|
| **odyssey-voyage-II-server** | Schema + resolver + datasource | Define the mutation, business logic, DB update, refund |
| **odyssey-voyage-II-client** | UI component + GraphQL mutation | Cancel button, confirmation UX, Apollo cache update |

> **elasticsearch** is not involved — it's unrelated to the Airlock booking app.

## Current State

- No cancel flow exists today
- `BookingStatus` enum only has: `CURRENT`, `COMPLETED`, `UPCOMING`
- `createBooking` resolver (`monolith/resolvers.js:153`) deducts funds via `paymentsAPI.subtractFunds()`
- Booking stores `totalPrice` which is needed for refund calculation
- Client uses `BOOK_STAY` mutation pattern in `src/components/BookStay.js:27`

## Implementation Pieces

### 1. Schema Changes (`monolith/schema.graphql`)

- Add `CANCELLED` to the `BookingStatus` enum (line 299)
- Add `cancelBooking(bookingId: ID!): CancelBookingResponse!` to `type Mutation` (line 32)
- Add `CancelBookingResponse` type implementing `MutationResponse` interface:
  - `code: Int!`
  - `success: Boolean!`
  - `message: String!`
  - `booking: Booking`

### 2. Server Resolver (`monolith/resolvers.js`)

Follow the `createBooking` pattern (line 153) but reverse the flow:

1. **Auth check** — verify the user is the guest who made the booking (or host)
2. **Fetch booking** — get booking by ID, validate it belongs to the user
3. **Status validation** — only `UPCOMING` bookings can be cancelled (not `CURRENT` or `COMPLETED`)
4. **Refund via payments API** — call `dataSources.paymentsAPI.addFunds()` to refund the `totalPrice`
5. **Update booking status** — set status to `CANCELLED` in `dataSources.bookingsDb`
6. **Return response** — follow `MutationResponse` pattern with code/success/message

### 3. Bookings Datasource (`monolith/datasources/bookings.js`)

- Add `cancelBooking(bookingId)` or `updateBookingStatus(bookingId, status)` method
- Add `getBookingById(bookingId)` if not already present (needed to look up `totalPrice` for refund)

### 4. Client Mutation (new component or extend existing)

- New `CANCEL_BOOKING` gql mutation following the `BOOK_STAY` pattern (`BookStay.js:27`):
  ```graphql
  mutation CancelBooking($bookingId: ID!) {
    cancelBooking(bookingId: $bookingId) {
      success
      message
      booking { id status }
    }
  }
  ```
- `useMutation` hook with `refetchQueries` to refresh bookings list

### 5. Client UI (`src/components/Bookings.js` + `src/pages/bookings.js`)

- Cancel button on each upcoming booking in the `Booking` component (`Bookings.js:25`)
- Confirmation dialog before executing
- Optimistic UI update or refetch of `HOST_BOOKINGS` / `upcomingGuestBookings` queries
- Handle loading/error/success states

## Edge Cases & Complexity

| Concern | Detail |
|---------|--------|
| **Refund policy** | Full refund vs. partial refund vs. no refund based on cancellation window (days before check-in) |
| **Race condition** | Guest cancels while host is reviewing bookings |
| **Host notifications** | Should the host be notified of a cancellation? |
| **Listing availability** | Cancelled dates must become bookable again (affects `datesToExclude` logic in `BookStay.js:56`) |
| **Reviews** | A cancelled booking should not allow reviews (affects `Booking` type resolvers at `resolvers.js:418`) |

## End-to-End Flow

```
Guest clicks "Cancel" → Confirmation dialog
  → CANCEL_BOOKING mutation
    → Server: auth check → validate UPCOMING status
    → Server: paymentsAPI.addFunds(totalPrice) → refund
    → Server: bookingsDb.updateStatus(id, CANCELLED)
  → Client: refetch bookings queries → update UI
```
