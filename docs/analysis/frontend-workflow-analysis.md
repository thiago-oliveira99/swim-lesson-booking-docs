# Front-End Workflow Analysis

## Button Workflows from Screenshots

### 1. GENERATE SLOTS BUTTON (Instructor)

**Trigger:** Button "generate-slots-button" is clicked

**Workflow Steps:**

```
Step 1: Set states is_loading... of POOLS
  Purpose: Show loading indicator while generating

Step 2: Create a new AvailabilityRule
  Type: AvailabilityRule
  Fields:
    - pool = Dropdown pool-select-dropdown's value
    - start_minutes = starttimedrop's value's minutes
    - end_minutes = endtimedrop's value's minutes
    - start_date = DateInput start-date-input's value
    - end_date = DateInput end-date-input's value
    - end_time = endtimedrop's value's minutes
    - is_active = "yes"
    - days_of_week_display = Weekday dropdown's value:each item's Display :join with ,
    - start_time_display = starttimedrop's value's display
    - end_time_display = endtimedrop's value's display
    - days_of_week_numbers set list = Weekday dropdown's value:each item's Value
    - instructor = Current User's instructor_profile
    - start_time = starttimedrop's value's minutes

Step 3: Schedule API Workflow generate_all
  Parameters:
    - rule = Result of step 2 (Create a new Avail...)
    - days_of_week = Result of step 2's days_of_week_numbers :join with ","
    - current_date = Result of step 2's start_date
  Scheduled date: Current date/time

Step 4: Set states is_loading... of POOLS
  Purpose: Hide loading indicator

Step 5: Reset relevant inputs
  Purpose: Clear form after submission

Step 6: Display list in RepeatingGroup availability-group
  Purpose: Show newly created availability rule
```

**KEY FINDINGS:**
- Creates AvailabilityRule with ALL display fields populated upfront
- Converts days_of_week_numbers list to comma-separated string for generate_all
- Uses Timeslot data type for dropdown (starttimedrop, endtimedrop)
- Weekday dropdown returns list with Display and Value properties

---

### 2. BOOK LESSONS BUTTON (Parent)

**Trigger:** Button "appointments-details-cancel-button" is clicked

**Workflow Steps:**

```
Step 1: Schedule API Workflow book_lessons
  Parameters:
    - parent = Current User
    - child = Popup appointments-details-popup's child
    - service = Popup appointments-details-popup's service
    - base_slot = Parent group's AppointmentSlot
    - weeks = Popup appointments-details-popup's service's duration_weeks
  Scheduled date: Current date/time

Step 2: Go to page parent_isr_appointments
  Purpose: Navigate to bookings list page
```

**KEY FINDINGS:**
- Parent selects slot from UI
- Service determines number of weeks (service's duration_weeks)
- Creates multi-week booking automatically
- Simple 2-step workflow - backend does the heavy lifting

---

## COMPLETE FLOW ANALYSIS

### Instructor Creates Availability

```
UI ACTION: Instructor fills form and clicks "Generate Slots"
  - Selects pool: "City Pool"
  - Selects days: Mon, Wed, Fri
  - Selects times: 10:00 AM - 6:00 PM
  - Selects dates: Jan 1 - Jan 31

↓

FRONT-END WORKFLOW: generate-slots-button clicked
  Step 1: Show loading spinner
  Step 2: Create AvailabilityRule record
    - pool = City Pool
    - start_time = 600 (10 AM in minutes)
    - end_time = 1080 (6 PM in minutes)
    - start_minutes = 600 (same as start_time)
    - end_minutes = 1080 (same as end_time)
    - days_of_week_numbers = [1, 3, 5]
    - days_of_week_display = "Mon, Wed, Fri"
    - start_time_display = "10:00 AM"
    - end_time_display = "6:00 PM"
    - start_date = Jan 1
    - end_date = Jan 31
    - instructor = Current User's instructor_profile
  Step 3: Schedule generate_all API workflow
    - rule = new AvailabilityRule
    - days_of_week = "1,3,5" (converted from list)
    - current_date = Jan 1

↓

BACKEND WORKFLOW: generate_all is called
  Parameter: days_of_week = "1,3,5" (text, not list!)
  
  Step 1: Check if current_date matches days_of_week
    - Extract day from current_date (1 = Monday)
    - Check if "1,3,5" contains "1" → YES
    - Schedule generate_day for Jan 1
  
  Step 2: Recurse to next day
    - Schedule generate_all with current_date = Jan 2
    - Only when: current_date < end_date

↓

BACKEND WORKFLOW: generate_day is called (for Jan 1)
  Step 1: Make Numeric List
    - Count: (end_minutes - start_minutes) / 10
    - Example: (1080 - 600) / 10 = 48 slots
    - Creates: [0, 10, 20, 30, ... 470]
  
  Step 2: Schedule create_slot on a list
    - For each number in list (minute offset)

↓

BACKEND WORKFLOW: create_slot is called (48 times for Jan 1)
  Parameters:
    - slot_datetime = Jan 1, 10:00 AM (calculated: target_date + start_minutes + minute_offset)
    - rule = AvailabilityRule
    - minute_offset = 0, 10, 20, 30...
  
  Creates AppointmentSlot:
    - start_time = Jan 1, 10:00 AM (DATE type)
    - end_time = Jan 1, 10:10 AM (DATE type)
    - slot_date = Jan 1 (date only)
    - pool = rule's pool
    - services = rule's service
    - instructor = rule's instructor
    - is_available = yes
    - booking_count = 0

↓

RESULT: 
  - Jan 1 (Mon): 48 slots created
  - Jan 2 (Tue): Skipped (not in days_of_week)
  - Jan 3 (Wed): 48 slots created
  - Jan 4 (Thu): Skipped
  - Jan 5 (Fri): 48 slots created
  - Continues until Jan 31
  - Total: ~13 matching days × 48 slots = ~624 slots
```

---

### Parent Books Multi-Week Lesson

```
UI ACTION: Parent browses slots and clicks one
  - Sees: Monday, Jan 6, 10:00 AM
  - Clicks: "Book Now"
  
↓

UI POPUP: Booking details popup opens
  Shows:
    - Selected slot: Monday, Jan 6, 10:00 AM
    - Child: Dropdown (parent selects child)
    - Service: Dropdown (parent selects service)
      - Service shows: "4-Week Beginner Package"
      - Service.duration_weeks = 4

↓

UI ACTION: Parent clicks "Confirm Booking"

↓

FRONT-END WORKFLOW: appointments-details-cancel-button clicked
  Step 1: Schedule book_lessons API workflow
    Parameters:
      - parent = Current User
      - child = Selected child
      - service = Selected service (4-Week Beginner)
      - base_slot = Monday, Jan 6, 10:00 AM slot
      - weeks = 4 (from service.duration_weeks)
  
  Step 2: Navigate to "My Bookings" page

↓

BACKEND WORKFLOW: book_lessons is called
  Parameters:
    - weeks = 4
    - base_slot = Monday, Jan 6, 10:00 AM
  
  Step 1: Schedule create_booking_for_week
    - current_week = 0
    - total_weeks = 4
    - base_slot = Monday, Jan 6, 10:00 AM
    - booking_group_id = unique ID (generated)

↓

BACKEND WORKFLOW: create_booking_for_week (Week 0)
  Step 1: Search for AppointmentSlots
    - Find: Monday, Jan 6, 10:00 AM slot
  
  Step 2: Create Booking
    - child = selected child
    - service_type = 4-Week Beginner
    - start_date = Jan 6
    - start_time = Jan 6, 10:00 AM (DATE)
    - end_time = Jan 6, 10:10 AM (DATE)
    - weeks = 4
    - booking_group_id = unique ID
    - total_cost = service's price_per_week
  
  Step 3: Update AppointmentSlot
    - is_available = "no"
    - booking_count = 1
  
  Step 4: Schedule create_booking_for_week (Week 1)
    - current_week = 1
    - target_date = Jan 13 (7 days later)

↓

BACKEND WORKFLOW: create_booking_for_week (Week 1)
  [Repeats for Monday, Jan 13, 10:00 AM]

↓

BACKEND WORKFLOW: create_booking_for_week (Week 2)
  [Repeats for Monday, Jan 20, 10:00 AM]

↓

BACKEND WORKFLOW: create_booking_for_week (Week 3)
  [Repeats for Monday, Jan 27, 10:00 AM]
  
  After creating:
    - Check: current_week + 1 (4) < total_weeks (4)? NO
    - Stop recursion

↓

RESULT:
  - 4 Bookings created (Jan 6, 13, 20, 27)
  - All linked by booking_group_id
  - 4 AppointmentSlots marked unavailable
  - Parent sees all 4 bookings on "My Bookings" page
```

---

## KEY INSIGHTS FROM ACTUAL IMPLEMENTATION

### 1. Timeslot Data Type
You have a separate **Timeslot** data type used for dropdowns:
```
Timeslot:
  - display (text): "10:00 AM"
  - minutes (number): 600

Purpose: Dropdown options for start/end time selection
```

### 2. Display Fields Populated Upfront
In the create AvailabilityRule step, you set:
- `start_time_display` = dropdown's display
- `end_time_display` = dropdown's display
- `days_of_week_display` = weekday dropdown joined

**This means:** Display fields are **NOT calculated**, they're **stored** from UI input

### 3. Days of Week Conversion
```
UI: Weekday dropdown returns list of objects with Display and Value
    [{Display: "Monday", Value: 1}, {Display: "Wednesday", Value: 3}]

Storage in AvailabilityRule:
  - days_of_week_numbers = [1, 3, 5] (List of numbers)
  - days_of_week_display = "Mon, Wed, Fri" (text)

Passed to generate_all:
  - days_of_week = "1,3,5" (comma-separated text)
```

### 4. Start/End Minutes Purpose Revealed
```
start_minutes = starttimedrop's value's minutes = 600
end_minutes = endtimedrop's value's minutes = 1080

In generate_day:
  slot_count = (end_minutes - start_minutes) / 10
             = (1080 - 600) / 10
             = 48 slots

So start_minutes and end_minutes are the SAME as start_time and end_time!
They're redundant but used consistently in calculations.
```

### 5. Service Drives Multi-Week Booking
```
Service data type has:
  - duration_weeks (number): How many weeks in the package
  
Parent selects service → automatically books that many weeks
No option to customize week count during booking
```

---

## CORRECTED FIELD PURPOSES

### AvailabilityRule

**Time Fields (ALL IN MINUTES FROM MIDNIGHT):**
- `start_time` (number): 600 = Availability starts at 10:00 AM
- `end_time` (number): 1080 = Availability ends at 6:00 PM
- `start_minutes` (number): 600 = Same as start_time (redundant but used)
- `end_minutes` (number): 1080 = Same as end_time (redundant but used)

**Display Fields (PRE-CALCULATED FROM UI):**
- `start_time_display` (text): "10:00 AM" (from Timeslot dropdown)
- `end_time_display` (text): "6:00 PM" (from Timeslot dropdown)
- `days_of_week_display` (text): "Mon, Wed, Fri" (from Weekday dropdown)

**Days of Week (BOTH STORED):**
- `days_of_week_numbers` (List of numbers): [1, 3, 5] (for logic)
- Converted to text "1,3,5" when passed to generate_all

**Relationship Fields:**
- `instructor` (InstructorProfile)
- `pool` (Pool)
- `service` (Services)

**Status:**
- `is_active` (yes/no)

---

## WHAT TO DOCUMENT

### Database Schema - Correct Field Types:
```
AvailabilityRule:
  ✓ start_time (number) - minutes from midnight
  ✓ end_time (number) - minutes from midnight
  ✓ start_minutes (number) - same as start_time
  ✓ end_minutes (number) - same as end_time
  ✓ start_time_display (text) - "10:00 AM"
  ✓ end_time_display (text) - "6:00 PM"
  ✓ days_of_week_numbers (List of numbers) - [1,3,5]
  ✓ days_of_week_display (text) - "Mon, Wed, Fri"
  ✓ instructor (InstructorProfile)
  ✓ pool (Pool)
  ✓ service (Services)
  ✓ start_date (date)
  ✓ end_date (date)
  ✓ is_active (yes/no)
  ? weeks (number) - not used in shown workflows
  ? status (option set) - not used in shown workflows

AppointmentSlot:
  ✓ start_time (date) - Jan 1, 10:00 AM
  ✓ end_time (date) - Jan 1, 10:10 AM
  ✓ slot_date (date) - Jan 1 (no time)
  ✓ is_available (yes/no)
  ✓ booking_count (number)
  ✓ pool (Pool)
  ✓ services (Services)
  ✓ instructor (InstructorProfile)
  ✓ Availability Rule (AvailabilityRule)

Booking:
  ✓ child (Child)
  ✓ service_type (Services)
  ✓ instructor (InstructorProfile)
  ✓ pool (Pool)
  ✓ start_date (date)
  ✓ start_time (date)
  ✓ end_time (date)
  ✓ end_date (date)
  ✓ weeks (number)
  ✓ booking_group_id (text)
  ✓ total_cost (number)
  ✓ appointment_slot (AppointmentSlot)
  ✓ parent (User)
  ✓ status (text)

Timeslot:
  ✓ display (text) - "10:00 AM"
  ✓ minutes (number) - 600
```

### Workflows to Document:

**Front-End:**
1. Generate Slots Button (instructor)
2. Book Lessons Button (parent)

**Backend:**
1. generate_all (with days_of_week as TEXT parameter)
2. generate_day (with Make Numeric List SSA)
3. create_slot (creates AppointmentSlot with date-type times)
4. book_lessons (entry point for booking)
5. create_booking_for_week (recursive multi-week booking)

---

## NEXT: COMPREHENSIVE DOCUMENTATION UPDATE

Now I'll update ALL documentation files to match this actual implementation!
