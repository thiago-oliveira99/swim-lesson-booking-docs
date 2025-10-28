# API Workflow Documentation - ACTUAL IMPLEMENTATION

## Overview

This document details all backend API workflows in the swim lesson booking application **as actually implemented**. These workflows handle:
- Recursive slot generation from instructor availability
- Multi-week booking creation for parents
- Automated scheduling and processing

**Last Updated:** October 28, 2025  
**Status:** ✅ Matches production Bubble.io implementation

---

## Table of Contents

- [API Workflow Fundamentals](#api-workflow-fundamentals)
- [Slot Generation Workflows](#slot-generation-workflows)
  - [generate_all](#generate_all)
  - [generate_day](#generate_day)
  - [create_slot](#create_slot)
- [Booking Creation Workflows](#booking-creation-workflows)
  - [book_lessons](#book_lessons)
  - [create_booking_for_week](#create_booking_for_week)
- [Complete Flow Examples](#complete-flow-examples)
- [Performance and Limits](#performance-and-limits)
- [Troubleshooting](#troubleshooting)

---

## API Workflow Fundamentals

### What Are API Workflows?

API workflows in Bubble are backend processes that:
- ✅ Run on Bubble's servers (not in user's browser)
- ✅ Can be scheduled to run at specific times
- ✅ Can call themselves recursively for complex operations
- ✅ Can be exposed as HTTP API endpoints (optional)
- ✅ Have 60-second timeout limit (vs 10 seconds for regular workflows)
- ✅ Continue running even if user closes browser

### When to Use API Workflows

**✅ Use API workflows for:**
- Recursive operations (generating slots for many days)
- Long-running processes that would timeout in UI
- Batch operations on large datasets
- Multi-step processes (multi-week bookings)
- Background tasks that don't need user feedback

**❌ Don't use API workflows for:**
- Simple instant operations (use regular workflows)
- Operations that need immediate UI updates
- Single data manipulations

---

## Slot Generation Workflows

### Architecture Overview

```
Instructor clicks "Generate Slots"
          ↓
    [Regular Workflow]
    Create AvailabilityRule
          ↓
    Schedule generate_all ← API WORKFLOW STARTS HERE
          ↓
    ┌─────────────────────┐
    │   generate_all      │  Check each date: Jan 1, Jan 2, Jan 3...
    │   (Date Iterator)   │  Does date match days_of_week?
    └─────────────────────┘
          ↓ (if match)
    ┌─────────────────────┐
    │   generate_day      │  Create list: [0, 10, 20, 30, ... 470]
    │  (Time Calculator)  │  One number per 10-minute slot
    └─────────────────────┘
          ↓ (for each number)
    ┌─────────────────────┐
    │   create_slot       │  Create AppointmentSlot with
    │  (Slot Creator)     │  start_time = date + offset
    └─────────────────────┘
```

---

## generate_all

**Purpose:** Master controller that iterates through dates and schedules slot generation for matching days

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `rule` | AvailabilityRule | Yes | The availability rule to generate slots from |
| `days_of_week` | text | Yes | **Comma-separated text** like "1,3,5" (NOT a list!) |
| `current_date` | date | Yes | The date being processed (recursion counter) |

**⚠️ CRITICAL:** `days_of_week` is passed as **TEXT** ("1,3,5"), not as a list!

### Workflow Steps

```
Step 1: Schedule API Workflow "generate_day"
  Only when: days_of_week contains current_date:extract day:formatted as 1028.58
  
  Parameters:
    - rule: rule
    - target_date: current_date
  
  Scheduled date: Current date/time
  
  Logic:
    - Extract day number from current_date (1=Mon, 2=Tue, etc.)
    - Format as 1028.58 (numeric format)
    - Check if days_of_week text contains that number
    - Example: current_date = Jan 15 (Monday = 1)
              days_of_week = "1,3,5"
              "1,3,5" contains "1" → TRUE → Schedule generate_day

Step 2: Schedule API Workflow "generate_all" (RECURSION)
  Only when: current_date < rule's end_date
  
  Parameters:
    - rule: rule
    - days_of_week: days_of_week
    - current_date: current_date +days: 1
  
  Scheduled date: Current date/time
  
  Logic:
    - Move to next day (current_date + 1)
    - Call generate_all again for that day
    - Stop when current_date reaches end_date
```

### How It Works

**Example:** Generate slots for Mon/Wed/Fri, Jan 1-7, 2025

```
Call 1: generate_all(rule, "1,3,5", Jan 1)
  - Jan 1 = Monday (1)
  - "1,3,5" contains "1" → TRUE
  - Schedule generate_day(rule, Jan 1)
  - Schedule generate_all(rule, "1,3,5", Jan 2)

Call 2: generate_all(rule, "1,3,5", Jan 2)
  - Jan 2 = Tuesday (2)
  - "1,3,5" contains "2" → FALSE
  - Skip generate_day
  - Schedule generate_all(rule, "1,3,5", Jan 3)

Call 3: generate_all(rule, "1,3,5", Jan 3)
  - Jan 3 = Wednesday (3)
  - "1,3,5" contains "3" → TRUE
  - Schedule generate_day(rule, Jan 3)
  - Schedule generate_all(rule, "1,3,5", Jan 4)

... continues until Jan 7 ...

Result:
  - generate_day scheduled for: Jan 1, Jan 3, Jan 5 (Mon, Wed, Fri)
  - Skipped: Jan 2, Jan 4, Jan 6, Jan 7 (Tue, Thu, Sat, Sun)
```

### Why Recursion?

**Without recursion (BAD):**
```
Search for dates between Jan 1 and Jan 31 → 31 dates
Filter where day is in [1,3,5]
For each date:
  Create slots... [TIMEOUT after ~1000 slots]
```

**With recursion (GOOD):**
```
Process 1 date → Schedule next date
Each API call processes only 1 day = ~48 slots
31 API calls × 48 slots = ~1,488 slots
Each call finishes in <1 second = NO TIMEOUT
```

---

## generate_day

**Purpose:** For a single date, create a list of time offsets and schedule create_slot for each

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `rule` | AvailabilityRule | Yes | The availability rule |
| `target_date` | date | Yes | The specific date to generate slots for |

### Workflow Steps

```
Step 1: Make Numeric List SSA
  This action: Generate a numeric list and send it to the plugin's server-side action
  
  Number of items: rule's end_minutes - rule's start_minutes / 10 :floor
  Start at: 0
  Increment by: 10
  
  Example:
    - rule's start_minutes = 600 (10:00 AM)
    - rule's end_minutes = 1080 (6:00 PM)
    - Duration: 1080 - 600 = 480 minutes (8 hours)
    - Slots: 480 / 10 = 48
    - List created: [0, 10, 20, 30, 40, ... 460, 470]
    
  Why this works:
    - Each number represents minutes offset from start_time
    - 0 = exactly at start_time (10:00 AM)
    - 10 = 10 minutes after start_time (10:10 AM)
    - 20 = 20 minutes after start_time (10:20 AM)
    - ... and so on

Step 2: Schedule API Workflow "create_slot" on a list
  List to run on: Result of step 1 (Make Numeric List...)'s Make Numeric List Results
  
  API Workflow: create_slot
  
  Parameters:
    - slot_datetime: target_date:rounded down to day +minutes: rule's start_minutes +minutes: This number
    - rule: rule
    - minute_offset: This number
  
  Scheduled date: Current date/time
  
  Logic:
    - For each number in the list [0, 10, 20, 30...]
    - Calculate exact datetime: target_date + start_minutes + offset
    - Example: Jan 15 + 600 minutes + 0 = Jan 15, 10:00 AM
               Jan 15 + 600 minutes + 10 = Jan 15, 10:10 AM
               Jan 15 + 600 minutes + 20 = Jan 15, 10:20 AM
```

### Example Calculation

**Given:**
- `target_date` = January 15, 2025 (12:00 AM)
- `rule.start_minutes` = 600 (10:00 AM)
- `rule.end_minutes` = 1080 (6:00 PM)

**Step 1 creates:**
```
[0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100, ... 470]
Total: 48 numbers
```

**Step 2 calculates slot_datetime for each:**

```
Number = 0:
  slot_datetime = Jan 15 :rounded down to day +minutes: 600 +minutes: 0
  slot_datetime = Jan 15, 12:00 AM + 600 min + 0 min
  slot_datetime = Jan 15, 10:00 AM

Number = 10:
  slot_datetime = Jan 15, 12:00 AM + 600 min + 10 min
  slot_datetime = Jan 15, 10:10 AM

Number = 20:
  slot_datetime = Jan 15, 12:00 AM + 600 min + 20 min
  slot_datetime = Jan 15, 10:20 AM

... continues for all 48 numbers ...

Number = 470:
  slot_datetime = Jan 15, 12:00 AM + 600 min + 470 min
  slot_datetime = Jan 15, 12:00 AM + 1070 min
  slot_datetime = Jan 15, 5:50 PM
```

### Why Use "Make Numeric List SSA"?

**SSA = Server-Side Action** (Plugin from Floppy)

**Without plugin (BAD):**
```
Can't create dynamic lists in API workflows
Would need to hardcode 48 separate create_slot calls
Inflexible - can't change duration without rebuilding workflow
```

**With plugin (GOOD):**
```
✓ Dynamically generates list based on start/end times
✓ Works with any duration (4 hours, 8 hours, 12 hours)
✓ Clean, maintainable workflow
✓ One action instead of dozens
```

---

## create_slot

**Purpose:** Create a single AppointmentSlot record with specific datetime

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `slot_datetime` | date | Yes | **Full datetime** of slot (e.g., "Jan 15, 2025 10:00 AM") |
| `rule` | AvailabilityRule | Yes | The availability rule |
| `minute_offset` | number | Yes | Offset in minutes (for debugging/tracking) |

### Workflow Steps

```
Step 1: Create a new AppointmentSlot
  Type: AppointmentSlot
  
  Fields:
    start_time = slot_datetime
    end_time = slot_datetime +minutes: 10
    Availability Rule = rule
    pool = rule's pool
    services = rule's service
    is_available = "yes"
    instructor = Search for InstructorProfiles:first item
    slot_date = slot_datetime :rounded down to day
    booking_count = 0
```

### Field-by-Field Explanation

**start_time (date):**
```
Value: slot_datetime
Example: Jan 15, 2025 10:00:00 AM
Type: DATE (not number!)
Why: Full datetime for easy display and comparison
```

**end_time (date):**
```
Value: slot_datetime +minutes: 10
Example: Jan 15, 2025 10:10:00 AM
Type: DATE (not number!)
Why: 10-minute slots (can be changed to 30, 60, etc.)
```

**slot_date (date):**
```
Value: slot_datetime :rounded down to day
Example: Jan 15, 2025 12:00:00 AM (midnight)
Type: DATE with time set to 00:00:00
Why: For filtering by date without time component
Use: "Show me all slots on Jan 15" → filter by slot_date
```

**pool, services, instructor (relationships):**
```
Values: Copied from rule
Why: Denormalization - preserves data if rule deleted
Also: Faster queries (no joins needed)
```

**is_available (yes/no):**
```
Value: "yes"
Why: New slots start as available
Changed to "no" when booked
```

**booking_count (number):**
```
Value: 0
Why: Tracks how many bookings reference this slot
Typically 0 or 1 (single occupancy)
Could be 2+ for group lessons
```

### Why Times Are Dates (Not Numbers)

**Original plan was:**
```
start_time: number = 600 (minutes from midnight)
```

**Actual implementation:**
```
start_time: date = "Jan 15, 2025 10:00 AM"
```

**Why the change?**

| Using Numbers | Using Dates |
|---------------|-------------|
| Need to convert for display | Direct display: :formatted as "h:mm a" |
| Need to calculate full datetime | Already have full datetime |
| Need to store date separately | Date included in datetime |
| More flexible | Easier to work with in Bubble |

**Bubble's date type includes both date AND time**, so there's no reason to store them separately!

---

## Booking Creation Workflows

### Architecture Overview

```
Parent clicks "Book Lessons"
          ↓
    [Regular Workflow]
    Schedule book_lessons
          ↓
    ┌─────────────────────────────┐
    │     book_lessons            │  Entry point - validates booking
    │     (Entry Point)           │  Gets weeks from service
    └─────────────────────────────┘
          ↓
    ┌─────────────────────────────┐
    │  create_booking_for_week    │  Week 0: Book Jan 6 slot
    │  (Recursive Booking)        │  Create Booking #1
    └─────────────────────────────┘
          ↓
    ┌─────────────────────────────┐
    │  create_booking_for_week    │  Week 1: Book Jan 13 slot
    │  (RECURSION)                │  Create Booking #2
    └─────────────────────────────┘
          ↓
    ┌─────────────────────────────┐
    │  create_booking_for_week    │  Week 2: Book Jan 20 slot
    │  (RECURSION)                │  Create Booking #3
    └─────────────────────────────┘
          ↓
    ┌─────────────────────────────┐
    │  create_booking_for_week    │  Week 3: Book Jan 27 slot
    │  (RECURSION)                │  Create Booking #4
    └─────────────────────────────┘
          ↓
        STOP (current_week >= total_weeks)
```

---

## book_lessons

**Purpose:** Entry point for booking - validates and starts the recursive booking process

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `parent` | User | Yes | Parent making the booking |
| `child` | Child | Yes | Child attending lessons |
| `service` | Services | Yes | Service package selected |
| `base_slot` | AppointmentSlot | Yes | First slot to book (e.g., Monday Jan 6, 10 AM) |
| `weeks` | number | Yes | Number of weeks to book (from service.duration_weeks) |

### Workflow Steps

```
Step 1: Schedule API Workflow "create_booking_for_week"
  Only when: base_slot is not empty 
         AND base_slot's is_available is "yes" 
         AND weeks > 0 
         AND weeks ≤ 52
  
  Parameters:
    - parent: parent
    - child: child
    - service: service
    - base_slot: base_slot
    - current_week: 0
    - total_weeks: weeks
    - instructor: base_slot's instructor
    - pool: base_slot's pool
    - booking_group_id: parent's unique id :extract Current date/time:extract UNIX
    - target_date: base_slot's slot_date
  
  Scheduled date: Current date/time
```

### Business Logic

**Validation checks:**
1. ✅ `base_slot is not empty` - User selected a slot
2. ✅ `is_available = "yes"` - Slot not already booked
3. ✅ `weeks > 0` - At least 1 week
4. ✅ `weeks ≤ 52` - Maximum 52 weeks (1 year)

**booking_group_id generation:**
```
parent's unique id :extract Current date/time:extract UNIX

Example:
  parent.unique_id = "1234567890123x4567890"
  Current date/time = Jan 1, 2025 10:30:00 AM
  UNIX timestamp = 1735729800
  
  Result: "12345678901234567890_1735729800"
  
Why: Unique identifier linking all bookings in the multi-week package
```

---

## create_booking_for_week

**Purpose:** Create one booking for a specific week, then recursively call itself for the next week

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `parent` | User | Yes | Parent user |
| `child` | Child | Yes | Child |
| `service` | Services | Yes | Service type |
| `base_slot` | AppointmentSlot | Yes | Original slot (for reference) |
| `current_week` | number | Yes | Which week we're processing (0, 1, 2, 3...) |
| `total_weeks` | number | Yes | Total weeks in package (e.g., 4) |
| `instructor` | InstructorProfile | Yes | Instructor |
| `pool` | Pool | Yes | Pool location |
| `booking_group_id` | text | Yes | Links all bookings together |
| `target_date` | date | Yes | Date to book for this week |

### Workflow Steps

```
Step 1: Create a new Booking
  Only when: Search for AppointmentSlots:first item is not empty
  
  Search criteria for AppointmentSlot:
    - slot_date = target_date (the specific date for this week)
    - instructor = instructor
    - pool = pool
    - is_available = "yes"
    - services = service
    - (Finds the matching slot for this week)
  
  Create Booking:
    child = child
    service_type = service
    instructor = instructor
    pool = pool
    status = "Arbitrary text" (likely "confirmed")
    weeks = total_weeks
    booking_group_id = booking_group_id
    total_cost = service's price_per_week
    start_date = Search for AppointmentSlots:first item's slot_date
    start_time = Search for AppointmentSlots:first item's start_time
    end_time = Search for AppointmentSlots:first item's end_time
    end_date = Search for AppointmentSlots:first item's slot_date
    appointment_slot = Search for AppointmentSlots:first item

Step 2: Make changes to AppointmentSlot
  Only when: Result of step 1 (Create a new Booking...) is not empty
  
  Thing to change: Result of step 1's appointment_slot
  
  Fields:
    is_available = "no"
    booking_count = Result of step 1's appointment_slot's booking_count + 1

Step 3: Schedule API Workflow "create_booking_for_week" (RECURSION)
  Only when: current_week + 1 < total_weeks
         AND Result of step 1 (Create a new Booking...) is not empty
         AND current_week < 100
  
  Parameters:
    - parent: parent
    - child: child
    - service: service
    - base_slot: base_slot
    - current_week: current_week + 1
    - total_weeks: total_weeks
    - instructor: instructor
    - pool: pool
    - booking_group_id: booking_group_id
    - target_date: base_slot's slot_date :+(days): (current_week +1) × 7
  
  Scheduled date: Current date/time
```

### How It Works - Complete Example

**Scenario:** Parent books 4-week package starting Monday, Jan 6, 10:00 AM

```
=== CALL 1: create_booking_for_week (Week 0) ===

Parameters:
  - current_week = 0
  - total_weeks = 4
  - target_date = Jan 6, 2025

Step 1: Search for AppointmentSlot
  Criteria:
    - slot_date = Jan 6
    - is_available = "yes"
    - Same instructor, pool, service
  
  Found: AppointmentSlot #1
    - start_time: Jan 6, 2025 10:00 AM
    - end_time: Jan 6, 2025 10:10 AM
  
  Create Booking #1:
    - child: Johnny
    - start_time: Jan 6, 2025 10:00 AM
    - weeks: 4
    - booking_group_id: "abc123"
    - total_cost: $35

Step 2: Update AppointmentSlot #1
  - is_available = "no"
  - booking_count = 1

Step 3: Check recursion
  - current_week + 1 = 1
  - total_weeks = 4
  - 1 < 4? YES
  - Schedule create_booking_for_week (Week 1)
    - target_date = Jan 6 + (1 × 7 days) = Jan 13

=== CALL 2: create_booking_for_week (Week 1) ===

Parameters:
  - current_week = 1
  - total_weeks = 4
  - target_date = Jan 13, 2025

Step 1: Search for AppointmentSlot
  Found: AppointmentSlot #2 (Jan 13, 10:00 AM)
  
  Create Booking #2:
    - start_time: Jan 13, 2025 10:00 AM
    - booking_group_id: "abc123" (same as Booking #1)
    - total_cost: $35

Step 2: Update AppointmentSlot #2
  - is_available = "no"
  - booking_count = 1

Step 3: Check recursion
  - 1 + 1 = 2
  - 2 < 4? YES
  - Schedule Week 2 (Jan 20)

=== CALL 3: create_booking_for_week (Week 2) ===
[Same process for Jan 20]

=== CALL 4: create_booking_for_week (Week 3) ===
[Same process for Jan 27]

Step 3: Check recursion
  - 3 + 1 = 4
  - 4 < 4? NO
  - STOP RECURSION

=== RESULT ===

4 Bookings created:
  1. Jan 6, 10:00 AM - booking_group_id: "abc123"
  2. Jan 13, 10:00 AM - booking_group_id: "abc123"
  3. Jan 20, 10:00 AM - booking_group_id: "abc123"
  4. Jan 27, 10:00 AM - booking_group_id: "abc123"

4 AppointmentSlots marked unavailable
Total cost: $35 × 4 = $140
```

### Why Search Instead of Create?

**You might wonder:** Why search for slots? Why not just create them?

**Answer:** Slots are already created by generate_all/generate_day/create_slot!

```
Week 1: generate_all creates slots for Jan 1 - Jan 31
Result: ~624 slots created (including Jan 6, 13, 20, 27 at 10 AM)

Week 2: Parent books
  - Search finds existing Jan 6 slot → Book it
  - Search finds existing Jan 13 slot → Book it
  - Search finds existing Jan 20 slot → Book it
  - Search finds existing Jan 27 slot → Book it
```

**If we created slots during booking:**
```
❌ Duplicate slots (instructor's slots + parent's slots)
❌ No availability checking (would create even if already booked)
❌ Instructor's availability rules wouldn't be respected
```

---

## Complete Flow Examples

### Example 1: Generate Slots for Mon/Wed/Fri, 10 AM - 6 PM, 4 weeks

**Setup:**
- Start date: Jan 1, 2025 (Wednesday)
- End date: Jan 28, 2025 (Tuesday)
- Days: Monday (1), Wednesday (3), Friday (5)
- Hours: 10:00 AM (600) - 6:00 PM (1080)
- Duration: 480 minutes = 48 slots per day

**Execution:**

```
=== Instructor Clicks "Generate Slots" ===

Regular workflow creates AvailabilityRule:
  - start_date: Jan 1
  - end_date: Jan 28
  - start_time: 600
  - end_time: 1080
  - start_minutes: 600
  - end_minutes: 1080
  - days_of_week_numbers: [1, 3, 5]

Regular workflow schedules: generate_all(rule, "1,3,5", Jan 1)

=== API Workflows Begin ===

generate_all(rule, "1,3,5", Jan 1):
  - Jan 1 = Wednesday (3)
  - "1,3,5" contains "3" → YES
  - Schedule: generate_day(rule, Jan 1)
  - Schedule: generate_all(rule, "1,3,5", Jan 2)

generate_day(rule, Jan 1):
  - Create list: [0, 10, 20, ... 470] (48 numbers)
  - Schedule: create_slot for each number
  
create_slot (48 times):
  - Creates 48 AppointmentSlots for Jan 1

generate_all(rule, "1,3,5", Jan 2):
  - Jan 2 = Thursday (4)
  - "1,3,5" contains "4" → NO
  - Skip generate_day
  - Schedule: generate_all(rule, "1,3,5", Jan 3)

generate_all(rule, "1,3,5", Jan 3):
  - Jan 3 = Friday (5)
  - "1,3,5" contains "5" → YES
  - Schedule: generate_day(rule, Jan 3)
  - Schedule: generate_all(rule, "1,3,5", Jan 4)

... continues until Jan 28 ...

=== Results ===

Matching dates in range:
  - Jan 1 (Wed), Jan 3 (Fri), Jan 6 (Mon), Jan 8 (Wed), Jan 10 (Fri)
  - Jan 13 (Mon), Jan 15 (Wed), Jan 17 (Fri), Jan 20 (Mon), Jan 22 (Wed)
  - Jan 24 (Fri), Jan 27 (Mon)
  
Total: 12 days × 48 slots = 576 slots created

API calls made:
  - generate_all: 28 calls (one per day in range)
  - generate_day: 12 calls (one per matching day)
  - create_slot: 576 calls (one per slot)
  - Total: 616 API workflow calls

Time to complete: ~10-15 minutes (Bubble processes with built-in delays)
```

### Example 2: Book 4-Week Package

**Setup:**
- Parent: John Smith
- Child: Emma (age 8)
- Service: "4-Week Beginner Package" ($35/week)
- Selected slot: Monday, Jan 6, 10:00 AM

**Execution:**

```
=== Parent Clicks "Book Lessons" ===

Regular workflow:
  - Validates: slot is available
  - Schedules: book_lessons(parent, child, service, base_slot, 4)

=== book_lessons ===

Checks:
  ✓ base_slot is not empty
  ✓ is_available = "yes"
  ✓ weeks = 4 (within 1-52 range)

Generates booking_group_id:
  - parent.unique_id = "123456"
  - UNIX timestamp = 1736168400
  - booking_group_id = "123456_1736168400"

Schedules: create_booking_for_week(
  current_week: 0,
  total_weeks: 4,
  target_date: Jan 6,
  booking_group_id: "123456_1736168400"
)

=== create_booking_for_week (Week 0) ===

Searches for AppointmentSlot:
  - slot_date = Jan 6
  - is_available = "yes"
  - instructor = same
  - pool = same
  - service = same

Found: Slot at Jan 6, 10:00 AM

Creates Booking #1:
  - child: Emma
  - start_time: Jan 6, 2025 10:00 AM
  - end_time: Jan 6, 2025 10:10 AM
  - weeks: 4
  - booking_group_id: "123456_1736168400"
  - total_cost: $35
  - status: "confirmed"

Updates slot:
  - is_available = "no"
  - booking_count = 1

Checks recursion:
  - current_week + 1 = 1
  - 1 < 4? YES

Schedules: create_booking_for_week(
  current_week: 1,
  target_date: Jan 13
)

=== create_booking_for_week (Week 1) ===

[Same process for Jan 13]

=== create_booking_for_week (Week 2) ===

[Same process for Jan 20]

=== create_booking_for_week (Week 3) ===

Searches for AppointmentSlot:
  - slot_date = Jan 27
  - is_available = "yes"

Found: Slot at Jan 27, 10:00 AM

Creates Booking #4:
  - start_time: Jan 27, 2025 10:00 AM
  - booking_group_id: "123456_1736168400"
  - total_cost: $35

Updates slot:
  - is_available = "no"
  - booking_count = 1

Checks recursion:
  - current_week + 1 = 4
  - 4 < 4? NO
  - STOP RECURSION ✓

=== Results ===

4 Bookings created for Emma:
  1. Jan 6, 10:00 AM
  2. Jan 13, 10:00 AM
  3. Jan 20, 10:00 AM
  4. Jan 27, 10:00 AM

All linked by booking_group_id: "123456_1736168400"

Total cost: $35 × 4 = $140

4 AppointmentSlots now marked unavailable

Emma is enrolled in 4-week beginner swim lessons!
```

---

## Performance and Limits

### Bubble API Workflow Limits

**Free Plan:**
- ❌ No API workflows available

**Paid Plans:**
- ✅ 60-second timeout per workflow call
- ✅ Unlimited total calls per day
- ⚠️ Rate limiting applies (varies by plan)
- ✅ Can run thousands of workflows per hour

### Our Implementation Performance

**Slot Generation:**
```
Scenario: Generate 4 weeks of Mon/Wed/Fri slots (10 AM - 6 PM)
  - Days: 12
  - Slots per day: 48
  - Total slots: 576

API Calls:
  - generate_all: 28
  - generate_day: 12
  - create_slot: 576
  - Total: 616 calls

Time: 10-15 minutes
Why: Bubble adds small delays between calls to prevent server overload
```

**Multi-Week Booking:**
```
Scenario: Book 8-week package
  - Weeks: 8
  - API Calls: 9 (book_lessons + 8 × create_booking_for_week)
  - Time: <1 minute
  - Why: Fast because just searching/creating, no heavy processing
```

### Optimization Strategies

**✅ Already Optimized:**
1. **Recursion prevents timeouts** - Each call processes small amount
2. **Parallel processing** - Multiple days can generate simultaneously
3. **Minimal database queries** - Efficient searches
4. **Denormalized data** - No complex joins needed

**🔄 Could Be Improved:**
1. **Batch create slots** - Instead of 1 call per slot, create 10 at once
2. **Skip existing slots** - Check if slots already exist before generating
3. **Incremental generation** - Generate only 1-2 weeks at a time
4. **Caching** - Store frequently accessed data

**⚠️ Don't Optimize Unless Needed:**
- Current system works well
- Premature optimization adds complexity
- Only optimize if hitting limits or user complaints

### Rate Limiting

Bubble implements automatic rate limiting:

**What it does:**
- Prevents too many workflows from running simultaneously
- Queues excess workflows
- Runs them when capacity available

**What you see:**
- Slightly longer generation times during high usage
- No errors - just delays
- Consistent, reliable execution

**When it matters:**
- Multiple instructors generating slots simultaneously
- High traffic periods (registration opening)
- Bulk operations (admin generating for all instructors)

**How to handle:**
- User feedback: "Generating slots, this may take a few minutes..."
- Progress indicators (if possible)
- Email notification when complete

---

## Troubleshooting

### Issue: Slots not generating

**Symptoms:**
- Instructor clicks "Generate Slots"
- No slots appear
- No error message

**Check:**
1. ✅ `days_of_week_numbers` is not empty in AvailabilityRule
2. ✅ `start_date` < `end_date`
3. ✅ At least one date in range matches `days_of_week`
4. ✅ generate_all was scheduled (check Logs → Scheduler)

**Common causes:**
- Days of week doesn't match any dates in range
  - Example: Selected only Saturday (6), but range is Mon-Fri
- Start date = end date (needs at least 1 day range)
- End date is in the past

**Fix:**
```
Check AvailabilityRule:
  - days_of_week_numbers: [1, 3, 5] (not empty)
  - start_date: Jan 1, 2025
  - end_date: Jan 31, 2025 (after start_date)
  
Verify at least one match:
  - Jan 1 = Wednesday (3) → 3 in [1,3,5]? YES ✓
```

### Issue: Slots generate slowly

**Symptoms:**
- Slots eventually appear
- Takes 10-20 minutes
- All slots correct

**This is normal!**
- Bubble rate limits API workflows
- Adds small delays between calls
- Prevents server overload

**Not a bug, but can improve UX:**
```
Add loading message:
  "Generating slots... This may take 10-15 minutes. 
   You'll receive an email when complete."

Or show progress:
  "Generated slots for 3 of 12 days..."
```

### Issue: Multi-week booking creates fewer than expected

**Symptoms:**
- Parent books 4-week package
- Only 2 bookings created
- No error message

**Check:**
1. ✅ Are slots available for all weeks?
   - Week 1: Jan 6 slot available? 
   - Week 2: Jan 13 slot available?
   - Week 3: Jan 20 slot available?
   - Week 4: Jan 27 slot available?

**Common causes:**
- Someone else booked the slot first (race condition)
- Slots weren't generated for future dates
- Instructor cancelled availability for some weeks

**How to diagnose:**
```
Check AppointmentSlots:
  - slot_date = Jan 13
  - is_available = "yes" or "no"?
  - If "no" → Already booked (expected)
  - If doesn't exist → Slots not generated (BUG)

Check Booking records:
  - booking_group_id = "123456_1736168400"
  - Count records
  - Should match weeks requested
```

**Fix:**
```
If slots missing:
  - Instructor needs to extend their availability
  - Regenerate slots for missing dates

If slots exist but unavailable:
  - Expected behavior (first come, first served)
  - Parent needs to select different time
```

### Issue: Duplicate slots created

**Symptoms:**
- Multiple identical slots (same date, time, instructor)
- is_available = "yes" for both
- booking_count = 0 for both

**This shouldn't happen with current implementation!**

**Possible causes:**
- AvailabilityRule created twice by accident
- generate_all called twice
- Bug in conditional logic

**How to check:**
```
Count AppointmentSlots:
  - slot_date = Jan 6
  - start_time = 10:00 AM
  - instructor = John
  
If count > 1 → Duplicate

Check AvailabilityRules:
  - Same instructor, pool, dates
  - Were two rules created?

Check Logs:
  - Was generate_all scheduled twice?
```

**Fix:**
```
Short-term:
  - Manually delete duplicate slots
  - Check which is the "real" one (created first)

Long-term:
  - Add constraint: Only one active AvailabilityRule per instructor/pool/time
  - Add validation: Prevent overlapping rules
```

### Issue: Booking created but slot still shows as available

**Symptoms:**
- Booking exists in database
- AppointmentSlot.is_available = "yes" (should be "no")
- Another parent can book the same slot

**This is a BUG!**

**Check:**
```
Booking record:
  - appointment_slot = ?
  - Is it linked correctly?

AppointmentSlot record:
  - booking_count = ?
  - Should be 1, might be 0

Workflow logs:
  - Did Step 2 of create_booking_for_week run?
  - "Make changes to AppointmentSlot"
```

**Common causes:**
- Step 2 conditional failed (Result of step 1 was empty)
- Privacy rules prevented update
- Workflow timed out before Step 2

**Fix:**
```
Check conditional in Step 2:
  Only when: Result of step 1 is not empty
  
If step 1 failed → Booking wasn't created → Step 2 didn't run

If booking exists but step 2 didn't run:
  - Manually update slot:
    - is_available = "no"
    - booking_count = 1
  - Review workflow logs for errors
  - Check privacy rules on AppointmentSlot
```

---

## Best Practices

### DO:
✅ Use recursion for large batch operations  
✅ Add conditions to prevent infinite loops  
✅ Log errors for debugging  
✅ Denormalize frequently accessed data  
✅ Schedule workflows during low-traffic times when possible  
✅ Provide user feedback ("Processing...")  
✅ Test with small datasets first  

### DON'T:
❌ Create workflows without stop conditions  
❌ Process thousands of records in one call  
❌ Ignore rate limits  
❌ Forget error handling  
❌ Assume instant execution  
❌ Make workflows too complex (break into steps)  
❌ Expose sensitive workflows as public APIs  

---

## Related Documentation

- [02-data-structure.md](./02-data-structure.md) - Database schema
- [04-workflows.md](./04-workflows.md) - Front-end workflows
- [07-business-logic.md](./07-business-logic.md) - Business rules
- [09-troubleshooting.md](./09-troubleshooting.md) - Common issues

---

**Questions or issues?** Check the troubleshooting section or review the workflow logs in your Bubble app (Logs → Scheduler).
