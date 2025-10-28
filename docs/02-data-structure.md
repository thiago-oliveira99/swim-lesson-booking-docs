# Data Structure Documentation - ACTUAL IMPLEMENTATION

## Overview

This document describes the complete database schema for the swim lesson booking application **as actually implemented in Bubble.io**. All field types, relationships, and purposes are documented based on the live application.

**Last Updated:** October 28, 2025  
**Implementation:** Production Bubble.io Application

---

## Table of Contents

- [Data Type Summary](#data-type-summary)
- [Core Data Types](#core-data-types)
- [Supporting Data Types](#supporting-data-types)
- [Billing and Payment Data Types](#billing-and-payment-data-types)
- [Relationships Diagram](#relationships-diagram)
- [Field Type Conventions](#field-type-conventions)

---

## Data Type Summary

| Data Type | Purpose | Key Relationships |
|-----------|---------|-------------------|
| **User** | Authentication and user accounts | → InstructorProfile, → Children |
| **InstructorProfile** | Instructor-specific information | ← User, → AvailabilityRules, → Pools |
| **Child** | Children enrolled in lessons | ← User (parent), → Service |
| **Pool** | Swimming pool locations | ← InstructorProfile |
| **Services** | Lesson packages and pricing | → Bookings, → Children |
| **AvailabilityRule** | Instructor's recurring schedule | ← InstructorProfile, → AppointmentSlots |
| **AppointmentSlot** | Specific bookable time slots | ← AvailabilityRule, ↔ Booking |
| **Booking** | Parent's reservation | ← User (parent), ← Child, ← AppointmentSlot |
| **Timeslot** | Time selection options for UI | None (helper data) |
| **Billing** | Invoices and payment tracking | → Bookings, → Child, → InstructorProfile |
| **Invoice** | Simplified invoice records | → Bookings, ← User |
| **BookingSession** | Booking flow state management | (TBD) |

---

## Core Data Types

### User

**Purpose:** Built-in Bubble user type for authentication and authorization

| Field Name | Type | Required | Description |
|------------|------|----------|-------------|
| email | text | Yes | User's email (built-in, unique) |
| name | text | No | User's full name |
| phone_number | text | No | Contact phone number |
| profile_picture | image | No | User's profile photo |
| type | Type (option set) | No | User role: "parent" or "instructor" |
| children | List of Childs | No | Parent's enrolled children |
| instructor_profile | InstructorProfile | No | If user is instructor, links to profile |

**Privacy:** Privacy rules applied (see Privacy documentation)

**Business Rules:**
- Email must be unique (Bubble enforces)
- If `instructor_profile` is not empty → User is an instructor
- If `instructor_profile` is empty → User is a parent
- `type` field provides explicit role designation (optional, for clarity)

---

### InstructorProfile

**Purpose:** Additional information for users who teach lessons

| Field Name | Type | Required | Description |
|------------|------|----------|-------------|
| user | User | Yes | Links back to User account |
| bio | text | No | Instructor biography/description |
| certifications | text | No | Teaching certifications and qualifications |
| pools | List of Pools | No | Pools where instructor teaches |
| availability_rules | List of AvailabilityRules | No | Instructor's recurring schedules (reverse relationship) |

**Relationships:**
- **One-to-one** with User (user ↔ instructor_profile)
- **Many-to-many** with Pool (instructor can teach at multiple pools)
- **One-to-many** with AvailabilityRule (instructor creates many rules)

**Business Rules:**
- Created when user registers as instructor or upgrades account
- Cannot be deleted if instructor has future bookings
- `pools` list controls which pools appear in availability creation dropdown

---

### Child

**Purpose:** Represents children enrolled in swim lessons

| Field Name | Type | Required | Description |
|------------|------|----------|-------------|
| parent | User | Yes | Parent/guardian user account |
| name | text | Yes | Child's full name |
| age | number | No | Child's current age (years) |
| service | Services | No | Current/most recent service enrolled in |

**Relationships:**
- **Many-to-one** with User (many children to one parent)
- **Many-to-one** with Services (child's current service level)

**Business Rules:**
- `age` should be calculated from date_of_birth (if DOB field added)
- `service` tracks child's current enrollment level
- Cannot be hard-deleted if child has booking history
- Parent can have multiple children under one account

**Privacy:**
- Only parent can view/edit their children
- Instructors can see children in their bookings only

---

### Pool

**Purpose:** Swimming pool locations where lessons are conducted

| Field Name | Type | Required | Description |
|------------|------|----------|-------------|
| name | text | Yes | Pool name (e.g., "City Aquatic Center") |
| address | text | No | Street address |
| city | text | No | City |
| state | text | No | State/province |
| zip_code | text | No | Postal code |
| phone | text | No | Pool contact phone number |
| instructor | User | No | **⚠️ Should be List of InstructorProfiles** |
| is_active | yes/no | Yes | Whether pool is currently available |

**Business Rules:**
- `is_active = no` hides pool from availability creation
- Cannot be deleted if used in active AvailabilityRules
- Address fields used for location display and map links

**Known Issue:**
- `instructor` field should be `List of InstructorProfiles`, not single User
- Current workaround: InstructorProfile has `pools` list (reverse relationship)

---

### Services

**Purpose:** Defines lesson packages, pricing, and duration

| Field Name | Type | Required | Description |
|------------|------|----------|-------------|
| name | text | Yes | Service name (e.g., "4-Week Beginner Package") |
| description | text | No | Detailed description of service |
| price_per_week | number | Yes | Weekly price in dollars |
| deposit_amount | number | No | Upfront deposit required |
| FullPayment | number | No | Total package price (alternative to per-week) |
| duration_mins | number | Yes | Lesson duration in minutes (e.g., 30) |
| duration_seconds | number | No | Duration in seconds (redundant) |
| duration_weeks | number | Yes | Number of weeks in package (e.g., 4) |
| min_weeks | number | No | Minimum weeks parent can book |
| max_weeks | number | No | Maximum weeks parent can book |
| is_active | yes/no | Yes | Whether service is currently offered |
| status | text | No | Service status (active, archived, etc.) |
| stripe_price_id | text | No | Stripe integration: price object ID |

**Business Rules:**
- `duration_weeks` controls multi-week booking behavior
- `price_per_week` × `duration_weeks` = total package cost
- `min_weeks` and `max_weeks` provide booking range flexibility
- `is_active = no` hides service from booking flow
- Stripe integration via `stripe_price_id` for payment processing

**Pricing Models:**
1. **Per-week:** `price_per_week` = $35, `duration_weeks` = 4 → Total: $140
2. **Full package:** `FullPayment` = $130 (discount for prepay)
3. **Deposit + balance:** `deposit_amount` = $50, remaining due at first lesson

---

### AvailabilityRule

**Purpose:** Defines instructor's recurring availability schedule

| Field Name | Type | Required | Description |
|------------|------|----------|-------------|
| instructor | InstructorProfile | Yes | Instructor this rule belongs to |
| pool | Pool | Yes | Location where lessons occur |
| service | Services | Yes | Service offered during this availability |
| start_date | date | Yes | First date of availability |
| end_date | date | Yes | Last date of availability (inclusive) |
| start_time | number | Yes | Daily start time (minutes from midnight, e.g., 600 = 10 AM) |
| end_time | number | Yes | Daily end time (minutes from midnight, e.g., 1080 = 6 PM) |
| start_minutes | number | Yes | Same as start_time (used in calculations) |
| end_minutes | number | Yes | Same as end_time (used in calculations) |
| start_time_display | text | No | Formatted start time "10:00 AM" (pre-calculated) |
| end_time_display | text | No | Formatted end time "6:00 PM" (pre-calculated) |
| days_of_week_numbers | List of numbers | Yes | Days when available [1=Mon, 2=Tue, ..., 7=Sun] |
| days_of_week_display | text | No | Formatted days "Mon, Wed, Fri" (pre-calculated) |
| is_active | yes/no | Yes | Whether rule actively generates slots |
| status | AvailabilityStatus | No | Option set status (if used instead of is_active) |
| weeks | number | No | Duration in weeks (optional helper field) |

**Relationships:**
- **Many-to-one** with InstructorProfile
- **Many-to-one** with Pool
- **Many-to-one** with Services
- **One-to-many** with AppointmentSlot (generates many slots)

**Time Storage Explanation:**

**Why Both start_time AND start_minutes?**
```
start_time = 600       → Primary field: 10:00 AM in minutes from midnight
start_minutes = 600    → Same value, used in slot count calculation
end_time = 1080        → Primary field: 6:00 PM in minutes from midnight
end_minutes = 1080     → Same value, used in slot count calculation

In generate_day workflow:
  slot_count = (end_minutes - start_minutes) / 10
  slot_count = (1080 - 600) / 10 = 48 slots
```

**Both fields have the same value but serve different purposes in the code.**

**Days of Week Storage:**

```
UI Selection: User checks Mon, Wed, Fri checkboxes

Stored as:
  days_of_week_numbers = [1, 3, 5] (List of numbers)
  days_of_week_display = "Mon, Wed, Fri" (text for display)

Passed to generate_all:
  days_of_week parameter = "1,3,5" (comma-separated text)
  
Used in generate_all:
  Check if "1,3,5" contains current_date:extract day (1, 2, 3, etc.)
```

**Display Fields:**
- Populated from UI dropdowns at creation time
- Avoid repeated formatting calculations
- Improve page load performance

**Business Rules:**
- `end_date` must be >= `start_date`
- `end_time` must be > `start_time`
- `days_of_week_numbers` must contain at least one day
- Cannot delete rule with active bookings (soft delete: `is_active = no`)
- Slot generation triggered immediately after creation

---

### AppointmentSlot

**Purpose:** Specific bookable time slots generated from AvailabilityRules

| Field Name | Type | Required | Description |
|------------|------|----------|-------------|
| Availability Rule | AvailabilityRule | Yes | Rule that generated this slot |
| slot_date | date | Yes | Date of slot (no time component), e.g., "Jan 15, 2025" |
| start_time | date | Yes | **Full datetime** of slot start, e.g., "Jan 15, 2025 10:00 AM" |
| end_time | date | Yes | **Full datetime** of slot end, e.g., "Jan 15, 2025 10:10 AM" |
| duration_minutes | number | Yes | Slot duration in minutes (typically 10) |
| day_of_week | text | No | Day name (e.g., "Monday") for filtering |
| pool | Pool | Yes | Location (copied from AvailabilityRule) |
| services | Services | Yes | Service offered (copied from AvailabilityRule) |
| instructor | InstructorProfile | Yes | Instructor (copied from AvailabilityRule) |
| is_available | yes/no | Yes | Whether slot can be booked (opposite of is_booked) |
| booking_count | number | Yes | Number of bookings for this slot (0 or 1) |

**Relationships:**
- **Many-to-one** with AvailabilityRule
- **One-to-one** with Booking (via booking_count and search)
- **Many-to-one** with Pool, Services, InstructorProfile (denormalized)

**⚠️ CRITICAL: Time Storage as Dates**

```
Why dates instead of numbers?

start_time: date = "Jan 15, 2025 10:00:00 AM"
end_time: date = "Jan 15, 2025 10:10:00 AM"
slot_date: date = "Jan 15, 2025" (time set to midnight)

Bubble's date type includes both date and time components.

Advantages:
✓ No conversion needed for display
✓ Direct date/time comparisons work
✓ Easier to work with in Bubble workflows
✓ Formatted output: start_time :formatted as "h:mm a" = "10:00 AM"

In create_slot workflow:
  start_time = slot_datetime (passed as date parameter)
  end_time = slot_datetime +minutes: 10
  slot_date = slot_datetime :rounded down to day
```

**Denormalization:**
- `pool`, `services`, `instructor` copied from AvailabilityRule
- Improves query performance (no joins needed)
- Preserves data if AvailabilityRule is deleted

**Business Rules:**
- Generated in 10-minute increments
- `is_available = yes` means no booking exists
- `booking_count` tracks number of bookings (0 or 1 for single-occupancy)
- Cannot be deleted if booked (soft delete by deactivating AvailabilityRule)

**Search Optimization:**
- Index on: `slot_date`, `is_available`, `pool`
- Common query: "Find available slots between dates at specific pool"

---

### Booking

**Purpose:** Parent's reservation of an AppointmentSlot for their child

| Field Name | Type | Required | Description |
|------------|------|----------|-------------|
| appointment_slot | AppointmentSlot | Yes | The slot being booked |
| parent | User | Yes | Parent making the booking |
| child | Child | Yes | Child attending the lesson |
| service_type | Services | Yes | Service package selected |
| instructor | InstructorProfile | Yes | Instructor (denormalized from slot) |
| pool | Pool | Yes | Location (denormalized from slot) |
| start_date | date | Yes | Lesson date (copied from slot) |
| start_time | date | Yes | **Full datetime** of lesson start |
| end_time | date | Yes | **Full datetime** of lesson end |
| end_date | date | No | Same as start_date (for range bookings) |
| status | text | Yes | "confirmed", "cancelled_by_parent", "completed", etc. |
| weeks | number | Yes | Number of weeks in booking package (e.g., 4) |
| booking_group_id | text | Yes | Links related bookings in multi-week packages |
| total_cost | number | Yes | Total price for this booking |
| payment_amount | number | No | Amount paid |
| payment_date | date | No | When payment was received |
| payment_status | payment_status | No | Option set: "paid", "unpaid", "pending", "refunded" |
| stripe_payment_id | text | No | Stripe payment intent ID |

**Relationships:**
- **Many-to-one** with AppointmentSlot
- **Many-to-one** with User (parent)
- **Many-to-one** with Child
- **Many-to-one** with Services
- **Grouped by** booking_group_id (for multi-week packages)

**Multi-Week Booking:**

```
Parent books 4-week package:

Booking #1:
  - start_date = Jan 6
  - weeks = 4
  - booking_group_id = "abc123"
  - appointment_slot = Monday Jan 6, 10 AM slot

Booking #2:
  - start_date = Jan 13
  - weeks = 4
  - booking_group_id = "abc123"
  - appointment_slot = Monday Jan 13, 10 AM slot

Booking #3:
  - start_date = Jan 20
  - weeks = 4
  - booking_group_id = "abc123"
  - appointment_slot = Monday Jan 20, 10 AM slot

Booking #4:
  - start_date = Jan 27
  - weeks = 4
  - booking_group_id = "abc123"
  - appointment_slot = Monday Jan 27, 10 AM slot

All 4 bookings share booking_group_id "abc123"
Query: Search for Bookings where booking_group_id = "abc123"
Result: All 4 weeks of lessons
```

**Status Values:**
- `"confirmed"` - Active booking
- `"pending"` - Awaiting payment confirmation
- `"cancelled_by_parent"` - Parent cancelled
- `"cancelled_by_instructor"` - Instructor cancelled
- `"completed"` - Lesson finished
- `"no_show"` - Child didn't attend

**Payment Tracking:**
- `payment_status` tracks payment state
- `stripe_payment_id` links to Stripe payment
- `payment_amount` may differ from `total_cost` (partial payments, refunds)

**Business Rules:**
- Cannot book if child has conflicting slot at same time
- Cannot book slot where `is_available = no`
- Cancellation within 24 hours may forfeit payment
- All bookings in group must be cancelled together (optional rule)

---

## Supporting Data Types

### Timeslot

**Purpose:** Helper data type for time selection dropdowns in UI

| Field Name | Type | Required | Description |
|------------|------|----------|-------------|
| display | text | Yes | Human-readable time "10:00 AM" |
| minutes | number | Yes | Minutes from midnight (600) |

**Usage:**
```
Timeslot records in database:
  {display: "6:00 AM", minutes: 360}
  {display: "6:30 AM", minutes: 390}
  {display: "7:00 AM", minutes: 420}
  ...
  {display: "9:00 PM", minutes: 1260}

Dropdown options:
  Display to user: "10:00 AM"
  Value passed to workflow: 600 (minutes)

In create AvailabilityRule workflow:
  start_time = Timeslot dropdown's value's minutes
  start_time_display = Timeslot dropdown's value's display
```

**Pre-Population:**
Timeslot records created once during app setup:
- 6:00 AM to 10:00 PM
- 30-minute increments (or 15-minute for flexibility)
- Total: ~33 records

**Benefits:**
- Consistent time selection across app
- No time parsing needed
- Easy to change available times (add/remove records)

---

## Billing and Payment Data Types

### Billing

**Purpose:** Comprehensive invoice and billing management

| Field Name | Type | Required | Description |
|------------|------|----------|-------------|
| parent | User | Yes | Parent being billed |
| child | Child | Yes | Child the billing is for |
| instructor | InstructorProfile | Yes | Instructor providing service |
| bookings | List of Appointments | No | **Note: References deleted type** |
| services | List of Serviceses | Yes | Services included in bill |
| issue_date | date | Yes | When invoice was created |
| due_date | date | Yes | Payment due date |
| amount_due | number | Yes | Total amount owed |
| amount_paid | number | Yes | Amount paid so far |
| balance | number | Yes | Remaining balance (calculated) |
| discount_amount | number | No | Any discounts applied |
| coupon_code | text | No | Promotional code used |
| payment_method | text | No | "card", "cash", "check", etc. |
| status | text | Yes | "unpaid", "partial", "paid", "overdue" |
| description / notes | text | No | Invoice notes or description |
| stripe_invoice_id | text | No | Stripe invoice object ID |
| stripe_payment_intent_id | text | No | Stripe payment intent ID |

**Business Rules:**
- `balance = amount_due - amount_paid - discount_amount`
- Status auto-updates based on balance and due_date
- Can apply coupons at billing time
- Links to multiple bookings for package billing

**Stripe Integration:**
- `stripe_invoice_id` syncs with Stripe invoice
- `stripe_payment_intent_id` tracks payment processing

---

### Invoice

**Purpose:** Simplified invoice tracking (alternative to Billing)

| Field Name | Type | Required | Description |
|------------|------|----------|-------------|
| parent | User | Yes | Parent being invoiced |
| bookings | List of Bookings | Yes | Bookings included in invoice |
| invoice_number | text | Yes | Unique invoice identifier |
| issue_date | date | Yes | Invoice creation date |
| due_date | date | Yes | Payment due date |
| amount | number | Yes | Total invoice amount |
| status | text | Yes | "unpaid", "paid", "overdue", "cancelled" |
| stripe_invoice_id | text | No | Stripe integration ID |

**Difference from Billing:**
- **Invoice:** Simple, one-record-per-invoice
- **Billing:** Complex, tracks payments, discounts, multiple services

**Usage:**
- Use **Invoice** for straightforward billing
- Use **Billing** for complex payment plans, discounts, multiple children

---

### BookingSession

**Purpose:** Manages booking flow state (session data)

**Fields:** Not visible in screenshots, may include:
- Current step in booking wizard
- Selected slots (temporary)
- Form data before submission

**Usage:**
- Stores temporary data during multi-step booking
- Cleared after booking completion
- Prevents data loss if user navigates away

---

## Relationships Diagram

```
User
 ├─→ InstructorProfile (1:1)
 │    ├─→ AvailabilityRule (1:many)
 │    │    └─→ AppointmentSlot (1:many)
 │    └─→ Pool (many:many)
 │
 └─→ Child (1:many)
      └─→ Booking (1:many)
           └─→ AppointmentSlot (many:1)

Services
 ├─→ AvailabilityRule (many:1)
 ├─→ AppointmentSlot (many:1)
 ├─→ Booking (many:1)
 └─→ Child (many:1)

Pool
 ├─→ AvailabilityRule (many:1)
 └─→ AppointmentSlot (many:1)

Booking Group (virtual, via booking_group_id)
 └─→ Booking (1:many) - Links multi-week bookings
```

---

## Field Type Conventions

### Date vs. Number for Times

**In AvailabilityRule:**
- Times stored as **numbers** (minutes from midnight)
- Example: `start_time = 600` means 10:00 AM

**In AppointmentSlot and Booking:**
- Times stored as **dates** (full datetime)
- Example: `start_time = Jan 15, 2025 10:00:00 AM`

**Rationale:**
- AvailabilityRule defines recurring patterns (abstract time)
- AppointmentSlot/Booking are concrete instances (specific datetime)

### Text vs. List for Days of Week

**Storage:**
- `days_of_week_numbers`: List of numbers [1, 3, 5]
- Passed to workflows as: Text "1,3,5"

**Why both?**
- List: Easy to loop through in UI
- Text: Easy to check containment in workflows

### Display Fields

**Pattern:** `field_name_display` stores pre-formatted values
- `start_time_display = "10:00 AM"`
- `days_of_week_display = "Mon, Wed, Fri"`

**Purpose:** Performance optimization, avoid repeated formatting

---

## Data Integrity

### Soft Deletes
Never hard-delete records with dependencies:
- AvailabilityRule: Set `is_active = no`
- Services: Set `is_active = no`
- Pool: Set `is_active = no`

### Denormalization
Copy data to avoid breaking relationships:
- AppointmentSlot copies: pool, services, instructor
- Booking copies: instructor, pool, start_time, end_time

**Trade-off:**
- Pro: Data preserved if parent record deleted
- Con: Updates don't propagate (intentional)

---

## Next Steps

For workflow documentation showing how this data is created and modified, see:
- [04-workflows.md](./04-workflows.md) - UI interactions
- [05-api-workflows.md](./05-api-workflows.md) - Backend processes
- [07-business-logic.md](./07-business-logic.md) - Business rules

For troubleshooting data issues, see:
- [09-troubleshooting.md](./09-troubleshooting.md)
