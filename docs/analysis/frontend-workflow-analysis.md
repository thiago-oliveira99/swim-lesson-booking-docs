# Front-End Workflow Documentation - ACTUAL IMPLEMENTATION

## Overview

This document details all front-end (UI) workflows in the swim lesson booking application **as actually implemented**. These are workflows triggered by user interactions like button clicks, dropdown selections, and page loads.

**Last Updated:** October 28, 2025  
**Status:** ✅ Matches production Bubble.io implementation

---

## Table of Contents

- [Workflow Types](#workflow-types)
- [Instructor Workflows](#instructor-workflows)
  - [Generate Slots Button](#generate-slots-button)
  - [Define Availability Form](#define-availability-form)
- [Parent Workflows](#parent-workflows)
  - [Book Lessons Button](#book-lessons-button)
  - [Browse Slots](#browse-slots)
  - [Cancel Booking](#cancel-booking)
- [Common UI Patterns](#common-ui-patterns)
- [State Management](#state-management)

---

## Workflow Types

### Regular Workflows vs API Workflows

**Regular Workflows (This Document):**
- Run in user's browser
- Triggered by button clicks, page loads, input changes
- Update UI immediately
- 10-second timeout limit
- Handle user interactions

**API Workflows (See [05-api-workflows.md](./05-api-workflows.md)):**
- Run on Bubble's servers
- Triggered by schedules or other workflows
- Background processing
- 60-second timeout limit
- Handle heavy processing

---

## Instructor Workflows

### Generate Slots Button

**Trigger:** Button "generate-slots-button" is clicked

**Purpose:** Creates an AvailabilityRule and schedules backend slot generation

**Page:** instructor_define_availability (or similar)

---

#### Workflow Steps

```
Step 1: Set states is_loading... of POOLS
  Purpose: Show loading spinner while processing
  
  Element: POOLS (RepeatingGroup or Group)
  State: is_loading
  Value: yes
  
  UI Effect: Shows loading indicator to user

Step 2: Create a new AvailabilityRule
  Type: AvailabilityRule
  
  Fields:
    pool = Dropdown pool-select-dropdown's value
    start_minutes = starttimedrop's value's minutes
    end_minutes = endtimedrop's value's minutes
    start_date = DateInput start-date-input's value
    end_date = DateInput end-date-input's value
    end_time = endtimedrop's value's minutes
    is_active = "yes"
    days_of_week_display = Weekday dropdown's value:each item's Display :join with ,
    start_time_display = starttimedrop's value's display
    end_time_display = endtimedrop's value's display
    days_of_week_numbers set list = Weekday dropdown's value:each item's Value
    instructor = Current User's instructor_profile
    start_time = starttimedrop's value's minutes

Step 3: Schedule API Workflow generate_all
  API Workflow: generate_all
  
  Parameters:
    rule = Result of step 2 (Create a new AvailabilityRule...)
    days_of_week = Result of step 2 (Create a new AvailabilityRule...)'s days_of_week_numbers :join with ","
    current_date = Result of step 2 (Create a new AvailabilityRule...)'s start_date
  
  Scheduled date: Current date/time
  
  Important: days_of_week is converted from list [1,3,5] to text "1,3,5"

Step 4: Set states is_loading... of POOLS
  Purpose: Hide loading spinner
  
  Element: POOLS
  State: is_loading
  Value: no
  
  UI Effect: Removes loading indicator

Step 5: Reset relevant inputs
  Purpose: Clear form after submission
  
  Actions:
    - Reset inputs in group/popup
    - Clear all form fields
    - Prepare for next availability rule

Step 6: Display list in RepeatingGroup availability-group
  Purpose: Show newly created availability rule in list
  
  Element: availability-group (RepeatingGroup)
  Data source: Search for AvailabilityRules
    - Constraints: instructor = Current User's instructor_profile
    - Sort by: Created Date (descending)
  
  UI Effect: New rule appears at top of list
```

---

#### Field Mapping Explained

**Dropdowns Used:**

1. **pool-select-dropdown**
   - Type: Dropdown
   - Choices: Pool
   - Data source: Search for Pools where is_active = yes
   - This dropdown's value → AvailabilityRule.pool

2. **starttimedrop / endtimedrop**
   - Type: Dropdown
   - Choices: Timeslot
   - Data source: Search for Timeslots (6 AM - 10 PM)
   - Option caption: Timeslot's display ("10:00 AM")
   - This dropdown's value → Timeslot object with:
     - .minutes = 600
     - .display = "10:00 AM"

3. **Weekday dropdown**
   - Type: Dropdown (multi-select enabled)
   - Choices: Custom option set "Weekday"
   - Options: 
     - Monday (Display: "Mon", Value: 1)
     - Tuesday (Display: "Tue", Value: 2)
     - Wednesday (Display: "Wed", Value: 3)
     - Thursday (Display: "Thu", Value: 4)
     - Friday (Display: "Fri", Value: 5)
     - Saturday (Display: "Sat", Value: 6)
     - Sunday (Display: "Sun", Value: 7)
   - This dropdown's value → List of selected weekdays

**Why Both minutes AND display?**

```
start_minutes = 600         ← For calculations (slot generation)
start_time_display = "10:00 AM"  ← For UI display (no conversion needed)

This stores both representations at creation time:
✓ Fast display (no formatting needed)
✓ Accurate calculations (numeric time)
```

**Why Convert days_of_week_numbers to text?**

```
Stored in AvailabilityRule:
  days_of_week_numbers = [1, 3, 5]  ← List of numbers

Passed to generate_all:
  days_of_week = "1,3,5"  ← Comma-separated text

Why?
  - API workflow parameter constraints
  - Easier to check containment: "1,3,5" contains "1"
  - vs looping through list to check if contains 1
```

---

#### Complete Example

**User Actions:**
1. Instructor selects "City Pool"
2. Selects start time: 10:00 AM
3. Selects end time: 6:00 PM
4. Selects days: Mon, Wed, Fri
5. Selects dates: Jan 1 - Jan 31
6. Clicks "Generate Slots"

**What Happens:**

```
Step 1: Show loading spinner

Step 2: Create AvailabilityRule
  pool = City Pool (object)
  start_time = 600
  end_time = 1080
  start_minutes = 600
  end_minutes = 1080
  start_time_display = "10:00 AM"
  end_time_display = "6:00 PM"
  days_of_week_numbers = [1, 3, 5]
  days_of_week_display = "Mon, Wed, Fri"
  start_date = Jan 1, 2025
  end_date = Jan 31, 2025
  instructor = Current User's instructor_profile
  is_active = yes

Step 3: Schedule generate_all
  rule = (the AvailabilityRule just created)
  days_of_week = "1,3,5"  ← Converted from [1,3,5]
  current_date = Jan 1, 2025

Step 4: Hide loading spinner

Step 5: Clear form
  - Pool dropdown reset
  - Time dropdowns reset
  - Date inputs reset
  - Weekday checkboxes unchecked

Step 6: Display in list
  - RepeatingGroup refreshes
  - New rule appears: "City Pool: Mon, Wed, Fri 10:00 AM - 6:00 PM"
  - User sees confirmation

Background (not visible):
  - generate_all API workflow begins
  - Processes dates one by one
  - Creates ~624 slots over 10-15 minutes
  - User can navigate away - process continues
```

---

### Define Availability Form

**Trigger:** Page load (instructor_define_availability)

**Purpose:** Load user's existing availability rules and populate dropdowns

---

#### Page Load Workflow

```
When: Page is loaded

Step 1: Load Timeslot options
  Purpose: Populate time dropdowns with available times
  
  Dropdown: starttimedrop
  Data source: Search for Timeslots
    Sort by: minutes (ascending)
  
  Result: Dropdown shows 6:00 AM, 6:30 AM, 7:00 AM, ... 10:00 PM

Step 2: Load Pool options
  Purpose: Show pools where this instructor teaches
  
  Dropdown: pool-select-dropdown
  Data source: Current User's instructor_profile's pools
  
  Result: Dropdown shows only instructor's assigned pools

Step 3: Load existing AvailabilityRules
  Purpose: Display instructor's current schedules
  
  RepeatingGroup: availability-group
  Data source: Search for AvailabilityRules
    Constraints:
      - instructor = Current User's instructor_profile
      - is_active = yes (optional: show only active)
    Sort by: start_date (descending)
  
  Result: List shows all current availability rules

Step 4: Set default values (optional)
  Purpose: Pre-fill form with sensible defaults
  
  Actions:
    - Set starttimedrop to 9:00 AM
    - Set endtimedrop to 5:00 PM
    - Set start_date to Current date
    - Set end_date to Current date +days: 28 (4 weeks)
```

---

#### RepeatingGroup Cell Design

**Each cell displays:**
```
Cell content:
  - Pool name: "City Pool"
  - Days: "Mon, Wed, Fri"
  - Time: "10:00 AM - 6:00 PM"
  - Dates: "Jan 1 - Jan 31, 2025"
  - Status badge: "Active" (green) or "Inactive" (gray)
  - Edit button (optional)
  - Delete button (optional)

Dynamic expressions:
  - Text: This AvailabilityRule's pool's name
  - Text: This AvailabilityRule's days_of_week_display
  - Text: This AvailabilityRule's start_time_display & " - " & This AvailabilityRule's end_time_display
  - Text: This AvailabilityRule's start_date :formatted as MMM D & " - " & This AvailabilityRule's end_date :formatted as MMM D, YYYY
```

---

## Parent Workflows

### Book Lessons Button

**Trigger:** Button "appointments-details-cancel-button" is clicked

**Purpose:** Books multi-week lesson package for child

**Page:** Parent dashboard with appointment details popup

---

#### Workflow Steps

```
Step 1: Schedule API Workflow book_lessons
  API Workflow: book_lessons
  
  Parameters:
    parent = Current User
    child = Popup appointments-details-popup's child
    service = Popup appointments-details-popup's service
    base_slot = Parent group's AppointmentSlot
    weeks = Popup appointments-details-popup's service's duration_weeks
  
  Scheduled date: Current date/time
  
  Important: weeks comes from service, not user input!
  - If service is "4-Week Beginner Package"
  - service.duration_weeks = 4
  - Automatically books 4 weeks

Step 2: Go to page parent_isr_appointments
  Purpose: Navigate to bookings list page
  
  Page: parent_isr_appointments (or "My Bookings")
  
  UI Effect:
    - User sees their new bookings
    - Confirmation message (if added)
    - List of upcoming lessons
```

---

#### Popup Context

**appointments-details-popup contains:**

```
Elements:
  - Text: Selected slot datetime
  - Dropdown: Select child
  - Dropdown: Select service package
  - Text: Price display (service's price_per_week × duration_weeks)
  - Text: Dates display (shows all 4 weeks)
  - Button: "Confirm Booking" (triggers this workflow)
  - Button: "Cancel" (closes popup)

Data passed to popup:
  - AppointmentSlot (parent group data)
  - Available services (dropdown data source)
  - User's children (dropdown data source)

Validation:
  - child is not empty
  - service is not empty
  - base_slot is available (is_available = yes)
```

---

#### Complete Example

**User Actions:**
1. Parent browses available slots (RepeatingGroup)
2. Clicks on "Monday, Jan 6, 10:00 AM" slot
3. Popup opens with booking details
4. Selects child: "Emma"
5. Selects service: "4-Week Beginner Package ($140)"
6. Clicks "Confirm Booking"

**What Happens:**

```
Step 1: Schedule book_lessons
  Parameters sent to API workflow:
    parent = John Smith (Current User)
    child = Emma (selected from dropdown)
    service = 4-Week Beginner Package
    base_slot = AppointmentSlot for Jan 6, 10 AM
    weeks = 4 (from service.duration_weeks)
  
  Scheduled for: Immediate execution

Step 2: Navigate to My Bookings page
  URL: /parent_isr_appointments
  
  Page loads:
    - Shows RepeatingGroup of bookings
    - Bookings appear shortly (API workflow creates them)
    - Emma's 4 lessons listed:
      * Jan 6, 10:00 AM
      * Jan 13, 10:00 AM
      * Jan 20, 10:00 AM
      * Jan 27, 10:00 AM

Background (API workflows):
  - book_lessons validates and starts process
  - create_booking_for_week (Week 0) creates Booking #1
  - create_booking_for_week (Week 1) creates Booking #2
  - create_booking_for_week (Week 2) creates Booking #3
  - create_booking_for_week (Week 3) creates Booking #4
  - All complete in ~30 seconds
```

---

### Browse Slots

**Page:** parent_booking (or similar)

**Purpose:** Display available appointment slots for parents to browse

---

#### Page Load Workflow

```
When: Page is loaded

Step 1: Load available slots
  RepeatingGroup: available-slots-group
  
  Data source: Search for AppointmentSlots
    Constraints:
      - is_available = yes
      - slot_date ≥ Current date
      - pool = (optional) Selected pool filter
      - instructor = (optional) Selected instructor filter
      - services = (optional) Selected service filter
    Sort by: slot_date (ascending), then start_time (ascending)
  
  Result: List of available slots, earliest first

Step 2: Load filter options
  Purpose: Populate dropdown filters
  
  Pool filter dropdown:
    Data source: Search for Pools where is_active = yes
  
  Instructor filter dropdown:
    Data source: Search for InstructorProfiles
  
  Service filter dropdown:
    Data source: Search for Services where is_active = yes
```

---

#### RepeatingGroup Cell Design

**Each cell displays:**
```
Cell content:
  - Date badge: "Mon, Jan 6"
  - Time: "10:00 AM"
  - Duration: "30 min"
  - Instructor: "John Smith"
  - Pool: "City Pool"
  - Service: "Beginner Package"
  - Price: "$35/week"
  - "Book" button → Opens booking popup

Click event:
  When: Cell is clicked (or "Book" button clicked)
  
  Action 1: Display data in popup
    Element: appointments-details-popup
    
    Data passed:
      - This AppointmentSlot (entire object)
      - Defaults for dropdowns
  
  Action 2: Show popup
    Element: appointments-details-popup
    Effect: Popup becomes visible with slot details

Dynamic expressions:
  - Text: This AppointmentSlot's slot_date :formatted as ddd, MMM D
  - Text: This AppointmentSlot's start_time :formatted as h:mm a
  - Text: This AppointmentSlot's instructor's user's name
  - Text: This AppointmentSlot's pool's name
```

---

#### Filter Workflow

**Trigger:** Dropdown value changed

```
When: Pool dropdown value is changed

Action: Filter RepeatingGroup
  Element: available-slots-group
  
  Constraints (add or modify):
    - pool = Dropdown pool-filter's value
    - (keep existing constraints)
  
  Result: RepeatingGroup refreshes, shows only slots at selected pool

When: Date input value is changed

Action: Filter RepeatingGroup
  Element: available-slots-group
  
  Constraints (add or modify):
    - slot_date = DateInput date-filter's value
    - (keep existing constraints)
  
  Result: Shows only slots on selected date

When: "Clear Filters" button is clicked

Action: Reset RepeatingGroup
  Element: available-slots-group
  
  Constraints: Reset to defaults (only is_available = yes, slot_date ≥ Current date)
  
  Result: Shows all available slots again
```

---

### Cancel Booking

**Trigger:** Button "cancel-booking-button" is clicked

**Purpose:** Cancel an existing booking and free up the slot

**Page:** parent_isr_appointments or booking details page

---

#### Workflow Steps

```
Step 1: Make changes to Booking
  Thing to change: Parent group's Booking
  
  Fields:
    status = "cancelled_by_parent"
  
  Important: Don't delete! Preserve history

Step 2: Make changes to AppointmentSlot
  Thing to change: Parent group's Booking's appointment_slot
  
  Fields:
    is_available = "yes"
    booking_count = This AppointmentSlot's booking_count - 1
  
  Important: Frees up slot for other parents

Step 3 (Optional): Send cancellation email
  Action: Send email
  
  To: Current User's email
  Subject: "Booking Cancelled"
  Body: "Your booking for [date] at [time] has been cancelled."

Step 4: Refresh page or close popup
  Action: Go to page parent_isr_appointments
  Or: Hide popup
  
  Result: User sees updated booking list
```

---

#### Cancellation Rules

**Business Logic (from [07-business-logic.md](./07-business-logic.md)):**

```
Cancellation allowed:
  ✓ More than 24 hours before lesson
  ✓ Status is "confirmed" (not already cancelled)
  ✓ Lesson hasn't started yet

Cancellation policy:
  - >24 hours notice: Full refund
  - <24 hours notice: No refund (forfeit payment)
  - Admin can override
```

**Conditional on cancel button:**
```
Only when:
  - This Booking's start_time > Current date/time +hours: 24
  - This Booking's status = "confirmed"

Button states:
  - If disabled: Show tooltip "Cancellation not available within 24 hours"
  - If enabled: Allow cancellation
```

---

## Common UI Patterns

### Loading States

**Pattern:** Show spinner while processing

```
When: Button is clicked (e.g., Generate Slots)

Step 1: Set state is_loading = yes
  Element: Group or icon
  State: is_loading (yes/no)
  Value: yes

Step 2-N: Do processing...

Step N+1: Set state is_loading = no
  Element: Same group or icon
  State: is_loading
  Value: no

Conditional visibility:
  Loading icon:
    - Only when: Parent group's is_loading is yes
  
  Form content:
    - Only when: Parent group's is_loading is no
```

---

### Error Handling

**Pattern:** Display error messages to users

```
When: API call fails or validation fails

Action: Display data in text element
  Element: error-message-text
  Text: Error message
  
Action: Show element
  Element: error-message-group
  Effect: Element becomes visible

Auto-hide after delay (optional):
  Action: Hide element (with delay)
  Element: error-message-group
  Delay: 5000ms (5 seconds)

Common errors:
  - "Please select all required fields"
  - "This slot is no longer available"
  - "Booking failed. Please try again."
  - "Unable to generate slots. Check your availability settings."
```

---

### Confirmation Dialogs

**Pattern:** Confirm destructive actions

```
When: Delete or Cancel button is clicked

Action: Show popup
  Element: confirm-dialog-popup
  
Popup contains:
  - Text: "Are you sure you want to cancel this booking?"
  - Button: "Yes, Cancel" → Proceeds with cancellation
  - Button: "No, Go Back" → Hides popup

Yes button workflow:
  Step 1: Make changes (delete/cancel)
  Step 2: Hide popup
  Step 3: Show success message

No button workflow:
  Step 1: Hide popup
  (No changes made)
```

---

### Form Validation

**Pattern:** Validate before submission

```
When: Submit button is clicked

Conditional on button:
  Only when:
    - Input field-1 is not empty
    - Input field-2 is not empty
    - Dropdown field-3's value is not empty
    - (All required fields filled)

Alternative: Show error on click

When: Submit button is clicked

Step 1: Check if valid
  Only when: Input is empty
  
  Action: Show error message
  Element: error-text
  Text: "Please fill in all required fields"

Step 2: Proceed if valid
  Only when: All inputs are not empty
  
  Action: Create record / Schedule workflow
```

---

## State Management

### Custom States Used

**Common custom states in the app:**

1. **is_loading (yes/no)**
   - Element: Groups, buttons, forms
   - Purpose: Show/hide loading spinners
   - Set to: yes when processing starts, no when done

2. **selected_slot (AppointmentSlot)**
   - Element: Parent group on booking page
   - Purpose: Track which slot user clicked
   - Set to: This cell's AppointmentSlot when clicked

3. **filter_date (date)**
   - Element: RepeatingGroup or page
   - Purpose: Store user's date filter selection
   - Set to: DateInput's value when changed

4. **booking_step (number)**
   - Element: Multi-step booking form
   - Purpose: Track which step user is on
   - Values: 1 (select slot), 2 (select child), 3 (confirm)

---

### Passing Data Between Elements

**Pattern 1: Via custom states**

```
When: Cell is clicked in RepeatingGroup

Action: Set state
  Element: Parent group (page-level)
  State: selected_slot
  Value: This cell's AppointmentSlot

Other elements reference:
  - Text element: Parent group's selected_slot's start_time
  - Button: Only when Parent group's selected_slot is not empty
```

**Pattern 2: Via URL parameters**

```
When: Navigate to booking details page

Action: Go to page
  Page: booking_details
  
  Send parameter:
    booking_id = This Booking's unique id

On destination page:
  Data source: Get data from URL
  Type: Booking
  Unique id: Get data from page URL (Booking)
  
  Page elements display: This Booking's child's name, etc.
```

**Pattern 3: Via popup data**

```
When: Show popup

Action: Display data in popup
  Element: booking-details-popup
  Data: This cell's AppointmentSlot

Inside popup:
  - Text: Parent group's AppointmentSlot's start_time
  - Dropdown child: (no parent group needed, uses Current User's children)
```

---

## Best Practices

### DO:
✅ Show loading states for long operations  
✅ Validate forms before submission  
✅ Confirm destructive actions  
✅ Provide clear error messages  
✅ Reset forms after successful submission  
✅ Use custom states for temporary UI data  
✅ Keep workflows simple (use API workflows for heavy processing)  

### DON'T:
❌ Process large datasets in regular workflows  
❌ Make users wait for background tasks  
❌ Hard-delete records (use status flags)  
❌ Forget to update related records  
❌ Show technical error messages to users  
❌ Allow multiple simultaneous submissions  
❌ Forget to hide popups/dialogs after actions  

---

## Related Documentation

- [05-api-workflows.md](./05-api-workflows.md) - Backend workflows
- [02-data-structure.md](./02-data-structure.md) - Database schema
- [07-business-logic.md](./07-business-logic.md) - Business rules
- [09-troubleshooting.md](./09-troubleshooting.md) - Common issues

---

**Questions?** Review the related documentation or check your Bubble app's workflow editor for the exact implementation details.
