# Learning Practice: Black-Box Testing, Part 2
**Student:** Victoria Salomon 

**Application:** North Vacation Booking  
**Topics:** Decision Table Testing and State Transition Testing  
**Testing approach:** Test design based on requirements only; no application code was provided.

## Step 1 — Identify Conditions and Actions

### Conditions

 ID | Condition | Meaning |
|---|---|---|
| C1 | Valid dates | Check-in is before checkout. |
| C2 | Valid duration | Checkout minus check-in is 1–31 nights inclusive. Checkout is not a reserved night. |
| C3 | Valid guest count | There are 1–12 guests inclusive. |
| C4 | All nights available | Every requested night, from check-in up to but excluding checkout, is available. |
| C5 | More than five guests | Guest count is greater than five; an extra nightly fee applies. |

### Actions

| ID | Action | Meaning |
|---|---|---|
| A1 | Create reservation | Create a reservation in the **Pending** state. |
| A2 | Calculate standard price | Charge $200 per night plus a one-time $50 cleaning fee. |
| A3 | Add extra guest fee | Charge $25 per night for each guest above five. |
| A4 | Raise `ValueError` | Reject invalid dates, duration, or guest count. |
| A5 | Raise `UnavailableError` | Reject a request if any requested night is unavailable. |

For valid bookings:

```text
Total = (nights × $200) + $50 + (max(guests − 5, 0) × nights × $25)
```

# Step 2 — Decision Table

**Legend:** `T` = true, `F` = false, `-` = condition does not affect this rule, `X` = action occurs; blank = action does not occur.

| Conditions and actions | R1 | R2 | R3 | R4 | R5 | R6 | R7 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Conditions** | | | | | | | |
| C1: Dates valid | T | T | F | T | T | T | T |
| C2: Duration valid | T | T | - | F | T | T | T |
| C3: Guest count valid | T | T | - | - | F | T | T |
| C4: All nights available | T | T | - | - | - | F | T |
| C5: More than five guests | F | T | - | - | - | - | F |
| **Actions** | | | | | | | |
| A1: Create Pending reservation | X | X | | | | | X |
| A2: Calculate standard price | X | X | | | | | X |
| A3: Add extra guest fee | | X | | | | | |
| A4: Raise `ValueError` | | | X | X | X | | |
| A5: Raise `UnavailableError` | | | | | | X | |

**Rule descriptions:**

- **R1:** Valid booking for five or fewer guests; standard price only.
- **R2:** Valid booking for more than five guests; apply extra guest fee.
- **R3:** Invalid date order; raise `ValueError`.
- **R4:** Invalid duration; raise `ValueError`.
- **R5:** Invalid guest count; raise `ValueError`.
- **R6:** Valid booking details but at least one night unavailable; raise `UnavailableError`.
- **R7:** Minimum valid booking (one night, one guest); create Pending reservation.

Assumption: Each invalid test case focuses on one error at a time. Since the requirements do not specify which error is checked first, we test them separately. When the checkout date is before the check-in date, the reservation is already invalid, so the number of nights does not need to be evaluated separately.

## Step 3: Create Decision Table Test Cases

Dates are in **2026**. All requested nights are available unless stated otherwise.

| Test ID | Rule | Check-in | Checkout | Nights | Guests | Availability | Base Price | Additional Fees | Expected Price / Exception |
|---|---|---|---|---:|---:|---|---:|---|---|
| DT01 | R1 | Nov 1 | Nov 4 | 3 | 4 | All available | $600 | $50 cleaning | **$650** |
| DT02 | R2 | Nov 1 | Nov 5 | 4 | 7 | All available | $800 | $50 cleaning + $200 extra guests | **$1,050** |
| DT03 | R3 | Nov 5 | Nov 3 | Invalid (−2) | 4 | Irrelevant | — | — | `ValueError` |
| DT04 | R4 | Nov 1 | Dec 3 | 32 | 4 | Irrelevant | — | — | `ValueError` |
| DT05 | R5 | Nov 1 | Nov 4 | 3 | 13 | Irrelevant | — | — | `ValueError` |
| DT06 | R6 | Nov 1 | Nov 4 | 3 | 4 | Nov 2 unavailable | — | — | `UnavailableError` |
| DT07 | R7 | Nov 1 | Nov 2 | 1 | 1 | All available | $200 | $50 cleaning | **$250** |

**Price checks:**

- DT01: `(3 × $200) + $50 = $650`.
- DT02: `(4 × $200) + $50 + ((7 − 5) × 4 × $25) = $1,050`.
- DT07: `(1 × $200) + $50 = $250`.
## Step 4: Identify Reservation States and Events

- Identify five possible reservation states.
- Identify five events that may cause the reservation to change states.

| State | Description |
|---|---|
| **Pending** | A valid reservation has been created and is awaiting payment. |
| **Confirmed** | Payment succeeded. |
| **Checked In** | The guest checked in on the scheduled check-in date. |
| **Completed** | The guest checked out after checkout. Terminal state. |
| **Cancelled** | Payment failed or an allowed cancellation occurred. Terminal state. |

### Events

| Event | Description |
|---|---|
| Payment succeeds | Pending → Confirmed. |
| Payment fails | Pending → Cancelled. |
| Guest cancels | Pending or Confirmed → Cancelled. |
| Guest checks in | Confirmed → Checked In, on the check-in date. |
| Guest checks out | Checked In → Completed, after checkout. |

## Step 5: Create a State Transition Table

Document the expected result of each event from each relevant state.

| ID | Current state | Event | Expected state | Expected result | Type |
|---|---|---|---|---|---|
| T01 | Pending | Payment succeeds | Confirmed | Reservation is confirmed. | Valid |
| T02 | Pending | Payment fails | Cancelled | Reservation is cancelled. | Valid |
| T03 | Pending | Guest cancels | Cancelled | Reservation is cancelled. | Valid |
| T04 | Confirmed | Guest cancels | Cancelled | Reservation is cancelled. | Valid |
| T05 | Confirmed | Guest checks in on check-in date | Checked In | Guest is checked in. | Valid |
| T06 | Checked In | Guest checks out after checkout | Completed | Reservation is completed. | Valid |
| T07 | Pending | Guest attempts to check in | Pending | `InvalidStateError`; no state change. | Invalid |
| T08 | Confirmed | Guest attempts to check out | Confirmed | `InvalidStateError`; no state change. | Invalid |
| T09 | Cancelled | Guest attempts to check in | Cancelled | `InvalidStateError`; no state change. | Invalid |
| T10 | Completed | Guest attempts to cancel | Completed | `InvalidStateError`; no state change. | Invalid |
| T11 | Checked In | Guest attempts to cancel | Checked In | `InvalidStateError`; no state change. | Invalid |

## Step 6: Create State Transition Test Cases

Create test cases that cover:

- Every valid state transition, for a minimum of six test cases
- At least three invalid state transitions
- At least one complete reservation sequence
- At least one cancellation sequence

You must create at least 11 state transition test cases in total.

| Test ID | Starting state | Event | Expected state | Expected result | Transition type |
|---|---|---|---|---|---|
| ST01 | Pending | Payment succeeds | Confirmed | Reservation confirmed. | Valid |
| ST02 | Pending | Payment fails | Cancelled | Reservation cancelled. | Valid |
| ST03 | Pending | Guest cancels | Cancelled | Reservation cancelled. | Valid |
| ST04 | Confirmed | Guest cancels | Cancelled | Reservation cancelled. | Valid |
| ST05 | Confirmed | Guest checks in on check-in date | Checked In | Check-in accepted. | Valid |
| ST06 | Checked In | Guest checks out after checkout | Completed | Reservation completed. | Valid |
| ST07 | Pending | Guest attempts to check in | Pending | Raise `InvalidStateError`. | Invalid |
| ST08 | Confirmed | Guest attempts to check out | Confirmed | Raise `InvalidStateError`. | Invalid |
| ST09 | Cancelled | Guest attempts to check in | Cancelled | Raise `InvalidStateError`. | Invalid |
| ST10 | Completed | Guest attempts to cancel | Completed | Raise `InvalidStateError`. | Invalid |
| ST11 | Checked In | Guest attempts to cancel | Checked In | Raise `InvalidStateError`. | Invalid |
| ST12 | Pending | Payment succeeds → Check in → Check out | Completed | Complete valid lifecycle. | Complete sequence |
| ST13 | Pending | Payment succeeds → Guest cancels | Cancelled | Cancellation after confirmation. | Cancellation sequence |

