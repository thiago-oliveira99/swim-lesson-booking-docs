# Backend Workflow Analysis - Actual Implementation

## Workflow Architecture Overview

Based on screenshots, here's the actual implementation:

---

## 1. SLOT GENERATION SYSTEM

### generate_all (Master Controller)
**Parameters:**
- `rule` (AvailabilityRule)
- `days_of_week` (text) - e.g., "1,3,5"
- `current_date` (date)

**Logic:**
```
Step 1: Schedule generate_day
  Only when: days_of_week contains current_date:extract day:formatted as 1028.58
  
Step 2: Schedule generate_all (recursion)
  Only when: current_date < rule's end_date
  Parameters:
    - current_date = current_date +days: 1
```

**KEY FINDING:** Uses `days_of_week` as TEXT parameter, not list!

---

### generate_day (Day Processor)
**Parameters:**
- `rule` (AvailabilityRule)
- `target_date` (date)

**Logic:**
```
Step 1: Make Numeric List SSA
  Purpose: Create list of time slots
  Number of items: (rule's end_minutes - rule's start_minutes) / 10 :floor
  Start at: 0
  Increment by: 10
  
Step 2: Schedule create_slot on a list
  For each number in list
```

**KEY FINDINGS:**
- Uses `start_minutes` and `end_minutes` fields!
- Creates slots in 10-minute increments
- Uses "Make Numeric List SSA" plugin

---

### create_slot (Slot Creator)
**Parameters:**
- `slot_datetime` (date)
- `rule` (AvailabilityRule)
- `minute_offset` (number)

**Logic:**
```
Step 1: Create AppointmentSlot
  start_time = slot_datetime
  end_time = slot_datetime + minutes: 10
  Availability Rule = rule
  pool = rule's pool
  services = rule's service
  is_available = "yes"
  instructor = Search for InstructorProfiles:first item
  slot_date = slot_datetime:rounded down to day
  booking_count = 0
```

**KEY FINDINGS:**
- `start_time` stored as DATE (slot_datetime)
- `end_time` stored as DATE (slot_datetime + 10 minutes)
- Uses `slot_date` separate from `start_time`
- This explains why times are date type!

---

## 2. BOOKING SYSTEM

### book_lessons (Entry Point)
**Parameters:**
- `parent` (User)
- `child` (Child)
- `service` (Services)
- `base_slot` (AppointmentSlot)
- `weeks` (number)

**Logic:**
```
Step 1: Schedule create_booking_for_week
  Only when: base_slot is not empty 
         AND base_slot's is_available = yes 
         AND weeks > 0 
         AND weeks ≤ 52
```

**KEY FINDINGS:**
- Multi-week booking support!
- `weeks` parameter controls how many weeks to book

---

### create_booking_for_week (Recursive Booking Creator)
**Parameters:**
- `parent` (User)
- `child` (Child)
- `service` (Services)
- `base_slot` (AppointmentSlot)
- `current_week` (number) - defaults to 0
- `total_weeks` (number)
- `instructor` (InstructorProfile)
- `pool` (Pool)
- `booking_group_id` (text)
- `target_date` (date)

**Logic:**
```
Step 1: Create Booking
  Only when: Search for AppointmentSlots:first item is not empty
  
  Fields:
    - child = child
    - service_type = service
    - instructor = instructor
    - pool = pool
    - status = "Arbitrary text"
    - weeks = total_weeks
    - booking_group_id = booking_group_id
    - total_cost = service's price_per_week
    - start_date = Search for AppointmentSlots:first item's slot_date
    - start_time = Search for AppointmentSlots:first item's start_time
    - end_time = Search for AppointmentSlots:first item's end_time
    - end_date = Search for AppointmentSlots:first item's slot_date
    
Step 2: Make changes to AppointmentSlot
  Only when: Result of step 1 is not empty
  
  Thing to change: Result of step 1's appointment_slot
  Fields:
    - is_available = "no"
    - booking_count = appointment_slot's booking_count + 1
    
Step 3: Schedule create_booking_for_week (recursion)
  Only when: current_week + 1 < total_weeks
         AND Result of step 1 is not empty
         AND current_week < 100
  
  Parameters:
    - current_week = current_week + 1
    - target_date = (varies by implementation)
```

**KEY FINDINGS:**
- Searches for AppointmentSlots to book (doesn't create them!)
- Uses `booking_group_id` to link multi-week bookings
- Uses `booking_count` field to track bookings per slot
- Copies slot's start_time, end_time to Booking
- This is why Booking has date-type time fields!

---

## 3. UNDERSTANDING THE TIME STORAGE DESIGN

### Why Times Are Stored as Dates:

**In AppointmentSlot:**
```
start_time: date = "Jan 15, 2025 10:00 AM"
end_time: date = "Jan 15, 2025 10:30 AM"
slot_date: date = "Jan 15, 2025" (no time component)
```

**Purpose:**
- Bubble's date type includes both date and time
- Easier to display: just format the date field
- Easier to compare: date comparison includes time
- No conversion needed from minutes

**In AvailabilityRule:**
```
start_minutes: number = 0 (relative to start_time)
end_minutes: number = 720 (12 hours * 60)
start_time: number = 600 (10:00 AM in minutes from midnight)
end_time: number = 1320 (10:00 PM in minutes from midnight)
```

**Purpose:**
- `start_time`/`end_time`: Define availability window (600 = 10 AM)
- `start_minutes`/`end_minutes`: Calculate duration for slot generation
  - Used in: (end_minutes - start_minutes) / 10 to count slots
  - Example: (720 - 0) / 10 = 72 slots in a 12-hour period

---

## 4. FIELD PURPOSE CLARIFICATION

### AvailabilityRule Fields Explained:

**Time Definition:**
- `start_time` (number): When availability starts (e.g., 600 = 10 AM)
- `end_time` (number): When availability ends (e.g., 1080 = 6 PM)
- `start_minutes` (number): Relative start for calculation (usually 0)
- `end_minutes` (number): Duration in minutes for slot generation
  - Formula: `end_minutes - start_minutes` = total availability duration
  - Example: If teaching 10 AM - 6 PM = 480 minutes
  - So: start_minutes = 0, end_minutes = 480

**Days of Week:**
- `days_of_week` (List of texts): ["Monday", "Wednesday", "Friday"]
- `days_of_week_numbers` (List of numbers): [1, 3, 5]
- `days_of_week_display` (text): "Mon, Wed, Fri"

**Display Helpers:**
- `start_time_display` (text): "10:00 AM"
- `end_time_display` (text): "6:00 PM"
- Pre-formatted for UI to avoid repeated calculations

**Status:**
- `status` (AvailabilityStatus option set): More granular than is_active
  - Possible values: active, inactive, pending, expired?
- `is_active` (yes/no): Simple active/inactive flag

**Weeks:**
- `weeks` (number): How many weeks this rule generates slots for
  - Auto-calculates: end_date = start_date + (weeks * 7 days)
  - OR: Just stores the duration for reference

---

## 5. MULTI-WEEK BOOKING FLOW

```
Parent selects: 4-week package

1. book_lessons called
   - weeks = 4
   - base_slot = Jan 15 Monday 10 AM slot
   
2. create_booking_for_week called (current_week = 0)
   - Search for: Jan 15 Monday 10 AM slot
   - Create Booking #1 for Jan 15
   - Update slot: is_available = no, booking_count = 1
   - Schedule next: create_booking_for_week (current_week = 1, target_date = Jan 22)
   
3. create_booking_for_week called (current_week = 1)
   - Search for: Jan 22 Monday 10 AM slot
   - Create Booking #2 for Jan 22
   - Update slot: is_available = no
   - Schedule next: create_booking_for_week (current_week = 2, target_date = Jan 29)
   
4. create_booking_for_week called (current_week = 2)
   - Search for: Jan 29 Monday 10 AM slot
   - Create Booking #3 for Jan 29
   - Update slot: is_available = no
   - Schedule next: create_booking_for_week (current_week = 3, target_date = Feb 5)
   
5. create_booking_for_week called (current_week = 3)
   - Search for: Feb 5 Monday 10 AM slot
   - Create Booking #4 for Feb 5
   - Update slot: is_available = no
   - Check: current_week + 1 (4) < total_weeks (4)? NO
   - Stop recursion ✓
   
Result: 4 Bookings created, all linked by booking_group_id
```

---

## 6. CRITICAL REALIZATIONS

### Your Implementation is DIFFERENT from Our Discussion!

**Original Design (Documented):**
- Times as numbers (minutes from midnight)
- Simple one-booking-at-a-time
- Basic slot generation

**Your Actual Implementation:**
- Times as dates (full datetime)
- Multi-week booking packages
- More complex slot generation with minute offsets
- Booking groups for related bookings
- Booking count tracking

### Why This Matters:

**You CANNOT change time fields to numbers without breaking everything!**

Your workflows expect:
```
AppointmentSlot.start_time: date
Booking.start_time: date
```

If you change to:
```
AppointmentSlot.start_time: number
```

All these will break:
- create_slot: Sets start_time = slot_datetime (date)
- create_booking_for_week: Copies start_time to Booking
- Any display that formats start_time as date
- Any time comparisons

---

## 7. CORRECT FIELD ANALYSIS

### AvailabilityRule - What to Keep:

✅ **ESSENTIAL FIELDS (DO NOT DELETE):**
1. `start_time` (number) - Availability window start (600 = 10 AM)
2. `end_time` (number) - Availability window end (1080 = 6 PM)
3. `start_minutes` (number) - For calculating slot count (usually 0)
4. `end_minutes` (number) - For calculating slot count (total duration)
5. `days_of_week_numbers` (List of numbers) - [1, 3, 5] for logic
6. `instructor` (InstructorProfile)
7. `pool` (Pool)
8. `service` (Services)
9. `start_date` (date)
10. `end_date` (date)
11. `weeks` (number) - Multi-week support

✅ **USEFUL BUT OPTIONAL:**
- `days_of_week_display` (text) - UI helper
- `start_time_display` (text) - UI helper
- `end_time_display` (text) - UI helper
- `status` (option set) - Better than is_active

❌ **CAN POTENTIALLY DELETE:**
- `days_of_week` (List of texts) - Redundant with days_of_week_numbers
- `is_active` (yes/no) - If using status field instead

⚠️ **MISSING (Should Add):**
- `duration` (number) - Lesson duration (30 min)
- `buffer_time` (number) - Gap between lessons (10 min)

---

## 8. RECOMMENDATIONS

### Option 1: Keep Current Implementation (RECOMMENDED)
**Pros:**
- Everything works
- Multi-week bookings functional
- No risk of breaking

**Cons:**
- Times stored as dates (not ideal but works)
- Some redundant fields

**Action:**
- Document current implementation
- Add missing fields (duration, buffer_time)
- Keep all existing fields
- Update documentation to match reality

### Option 2: Refactor to Numbers
**Pros:**
- Cleaner data model
- Times as numbers (more flexible)

**Cons:**
- MASSIVE refactor required
- High risk of breaking everything
- All workflows must be rewritten
- All display logic must change

**Action:**
- NOT RECOMMENDED for production app
- Only if starting fresh

---

## 9. NEXT STEPS

1. **Keep your current implementation**
2. **Document what you have** (I'll update all docs)
3. **Add missing helpful fields:**
   - AvailabilityRule.duration (number)
   - AvailabilityRule.buffer_time (number)
4. **Optional cleanup:**
   - Remove days_of_week (List of texts) if not used
   - Choose: status OR is_active (not both)

**Bottom line: Your implementation is more advanced than what we discussed. Let's document what you have rather than change it!**
