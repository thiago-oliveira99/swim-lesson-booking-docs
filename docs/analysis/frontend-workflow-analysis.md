# Workflow Documentation

## Overview

This document details all interactive workflows in the swim lesson booking application, organized by user role and trigger type. Workflows are the actions that occur when users interact with buttons, forms, or page elements.

## Table of Contents

- [Workflow Types](#workflow-types)
- [Instructor Workflows](#instructor-workflows)
- [Parent Workflows](#parent-workflows)
- [Shared Workflows](#shared-workflows)
- [Workflow Execution Order](#workflow-execution-order)
- [Error Handling](#error-handling)

---

## Workflow Types

### Button Workflows
Triggered when a user clicks a button element. Execute immediately upon click.

### Page Load Workflows
Triggered automatically when a page loads. Used for initializing data or setting default states.

### Custom State Changes
Triggered when a custom state value changes. Used for reactive UI updates.

### Form Submission Workflows
Triggered when a user submits a form (Input elements with "Enter" key or Submit button).

---

## Instructor Workflows

### 1. Create Availability Rule Workflow

**Trigger:** Button click on "Create Availability Rule" or "Save Rule"

**Purpose:** Creates a new AvailabilityRule and triggers the recursive slot generation process

**Steps:**

1. **Create AvailabilityRule Record**
   ```
   Action: Create a new thing
   Type: AvailabilityRule
   Fields to set:
     - instructor = Current User's InstructorProfile
     - start_date = DatePicker's value
     - end_date = DatePicker's value
     - days_of_week = MultiSelectDropdown's value (comma-separated: "1,3,5")
     - start_time = TimePicker's value (in minutes from midnight)
     - end_time = TimePicker's value (in minutes from midnight)
     - location = PoolDropdown's value
     - service = ServiceDropdown's value
     - duration = Input's value (default: 30)
     - buffer_time = Input's value (default: 0)
     - is_active = yes
     - created_at = Current date/time
   ```

2. **Trigger Recursive Slot Generation**
   ```
   Action: Schedule API Workflow
   API Workflow: generate_all_slots
   Parameters:
     - availability_rule_id = Result of step 1's unique id
     - current_date = Result of step 1's start_date
   Schedule: Immediate (or 10 seconds delay to avoid timeout)
   ```

3. **Show Success Message**
   ```
   Action: Show alert
   Message: "Availability rule created! Generating appointment slots..."
   ```

4. **Navigate to Availability Page**
   ```
   Action: Navigate to page
   Destination: availability_management
   Send more parameters to page:
     - instructor_id = Current User's InstructorProfile's unique id (optional)
   ```

**Business Logic:**
- Days of week must be selected (validation: MultiSelect is not empty)
- End date must be >= Start date (validation: EndDate >= StartDate)
- End time must be > Start time (validation: EndTime > StartTime)
- Duration must be > 0 and <= (EndTime - StartTime)
- Buffer time must be >= 0

**Error Handling:**
- Show validation errors inline before allowing submission
- If API workflow scheduling fails, show error alert and don't navigate away

---

### 2. Edit Availability Rule Workflow

**Trigger:** Button click on "Edit" next to an existing rule

**Purpose:** Updates an existing AvailabilityRule and optionally regenerates slots

**Steps:**

1. **Update AvailabilityRule Record**
   ```
   Action: Make changes to thing
   Thing to change: Current cell's AvailabilityRule
   Fields to change:
     - start_date = DatePicker's value
     - end_date = DatePicker's value
     - days_of_week = MultiSelectDropdown's value
     - start_time = TimePicker's value
     - end_time = TimePicker's value
     - location = PoolDropdown's value
     - service = ServiceDropdown's value
     - duration = Input's value
     - buffer_time = Input's value
     - updated_at = Current date/time
   ```

2. **Conditional: Check if Dates Changed**
   ```
   Condition: DatePicker's value ≠ Current cell's AvailabilityRule's start_date
              OR DatePicker's value ≠ Current cell's AvailabilityRule's end_date
   
   If true:
     - Delete existing AppointmentSlots (only those with is_booked = no)
     - Trigger generate_all_slots API workflow with updated rule
   ```

3. **Show Success Message**
   ```
   Action: Show alert
   Message: "Availability rule updated!"
   ```

**Business Logic:**
- Cannot edit rules that have booked slots within the date range (validation check)
- If editing causes date range to shrink, slots outside range are soft-deleted (marked inactive)
- If editing causes date range to expand, new slots are generated only for new dates

**Warning Prompts:**
- "This rule has X booked appointments. Some changes may not apply to existing bookings. Continue?"

---

### 3. Delete/Deactivate Availability Rule Workflow

**Trigger:** Button click on "Delete" or "Deactivate" next to a rule

**Purpose:** Removes or deactivates an availability rule based on booking status

**Steps:**

1. **Check for Existing Bookings**
   ```
   Search: Search for Bookings
   Constraints:
     - appointment_slot's availability_rule = Current cell's AvailabilityRule
     - status = "confirmed" or "pending"
   ```

2. **Conditional Action**
   ```
   If Search's count > 0:
     Action: Make changes to thing
     Thing: Current cell's AvailabilityRule
     Fields:
       - is_active = no
       - deactivated_at = Current date/time
     
     Alert: "Rule deactivated (has bookings). Slots remain but won't generate new ones."
   
   Else:
     Action: Delete thing
     Thing: Current cell's AvailabilityRule
     
     Action: Delete a list of things
     Things: Search for AppointmentSlots where availability_rule = Current cell's AvailabilityRule
     
     Alert: "Rule and all associated empty slots deleted."
   ```

**Business Logic:**
- **Soft Delete:** If bookings exist, mark rule as inactive (is_active = no)
- **Hard Delete:** If no bookings exist, completely remove rule and all generated slots
- Deactivated rules don't generate new slots but preserve historical data

---

### 4. View Calendar Workflow

**Trigger:** Page load on "Instructor Calendar" page

**Purpose:** Display instructor's schedule in a calendar view with all booked and available slots

**Steps:**

1. **Load Instructor Profile**
   ```
   Data source: Current User's InstructorProfile
   ```

2. **Load Appointment Slots**
   ```
   Search: Search for AppointmentSlots
   Constraints:
     - availability_rule's instructor = Current User's InstructorProfile
     - date >= Current date/time rounded down to day
     - date <= Current date/time +days 30 (or custom range)
   Sort by: date (ascending), then start_time (ascending)
   ```

3. **Display Repeating Group**
   ```
   Type of content: AppointmentSlot
   Data source: Result of step 2
   
   Each cell shows:
     - Date: Current cell's AppointmentSlot's date (formatted)
     - Time: Current cell's AppointmentSlot's start_time converted to HH:MM
     - Duration: Current cell's AppointmentSlot's duration
     - Location: Current cell's AppointmentSlot's location's name
     - Status badge: Conditional formatting based on booking status
     - Child name: Current cell's AppointmentSlot's booking's child's name (if booked)
   ```

4. **Status Badge Conditional**
   ```
   When: Search for Bookings:count > 0
         where appointment_slot = Current cell's AppointmentSlot
   Background color: Green
   Text: "Booked"
   
   Otherwise:
   Background color: Blue
   Text: "Available"
   ```

**Interactive Elements:**
- Click on slot → Show booking details popup
- Filter controls: Date range, location, status (available/booked)
- Week/Month view toggle

---

### 5. View Booking Details Workflow

**Trigger:** Button click on a booked slot in calendar or bookings list

**Purpose:** Display detailed information about a specific booking

**Steps:**

1. **Set Custom State**
   ```
   Action: Set state
   State: selected_slot
   Value: Current cell's AppointmentSlot
   ```

2. **Show Popup**
   ```
   Action: Show element
   Element: BookingDetailsPopup
   ```

3. **Load Booking Data in Popup**
   ```
   Data source: Search for Bookings:first item
   Constraints:
     - appointment_slot = Parent group's selected_slot
   
   Display:
     - Child: This Booking's child's name
     - Parent: This Booking's parent's name
     - Parent contact: This Booking's parent's phone
     - Service: This Booking's service's name
     - Date: This Booking's appointment_slot's date
     - Time: This Booking's appointment_slot's start_time
     - Location: This Booking's appointment_slot's location
     - Status: This Booking's status
     - Booking date: This Booking's created_at
     - Special notes: This Booking's notes
   ```

**Interactive Elements:**
- "Cancel Booking" button → Triggers cancel booking workflow
- "Contact Parent" button → Opens email client with parent's email
- "Close" button → Hides popup

---

### 6. Cancel Booking Workflow (Instructor-Initiated)

**Trigger:** Button click on "Cancel Booking" in booking details popup

**Purpose:** Allows instructor to cancel a booking and free up the slot

**Steps:**

1. **Confirmation Dialog**
   ```
   Action: Show alert
   Message: "Cancel this booking for [Child's name]? This will notify the parent."
   Style: Yes/No
   ```

2. **Update Booking Status**
   ```
   Action: Make changes to thing
   Thing: Current cell's Booking
   Fields:
     - status = "cancelled_by_instructor"
     - cancelled_at = Current date/time
     - cancellation_reason = Input's value (optional text area)
   ```

3. **Update Appointment Slot**
   ```
   Action: Make changes to thing
   Thing: Current cell's Booking's appointment_slot
   Fields:
     - booking = empty (clear the relationship)
     - is_booked = no
   ```

4. **Send Notification Email**
   ```
   Action: Schedule API Workflow (optional)
   Workflow: send_cancellation_email
   Parameters:
     - booking_id = Current cell's Booking's unique id
     - recipient = Current cell's Booking's parent's email
   ```

5. **Hide Popup and Refresh**
   ```
   Action: Hide element (BookingDetailsPopup)
   Action: Reset relevant elements
   ```

**Business Logic:**
- Cancellations within 24 hours of appointment time should prompt additional confirmation
- System tracks who cancelled (instructor vs parent) via status field
- Cancelled slots become immediately available for rebooking

---

## Parent Workflows

### 1. Search Available Slots Workflow

**Trigger:** Button click on "Search" or form submission on search page

**Purpose:** Find available appointment slots based on parent's search criteria

**Steps:**

1. **Validate Search Parameters**
   ```
   Validations:
     - Date range selected (start and end date)
     - At least one day of week selected (if filtering by day)
     - End date >= Start date
   ```

2. **Execute Search Query**
   ```
   Search: Search for AppointmentSlots
   Constraints:
     - date >= DatePicker start's value
     - date <= DatePicker end's value
     - is_booked = no
     - availability_rule's is_active = yes
     - (Optional) location = PoolDropdown's value (if selected)
     - (Optional) availability_rule's service = ServiceDropdown's value (if selected)
     - (Optional) availability_rule's instructor = InstructorDropdown's value (if selected)
   Sort by: date (ascending), start_time (ascending)
   ```

3. **Apply Day of Week Filter (if applicable)**
   ```
   Additional constraint (if DayFilter is not "All"):
     - Extract day of week from date field
     - Match against selected days (1=Mon, 2=Tue, etc.)
   
   Note: Bubble doesn't have built-in day-of-week extraction, so this may require:
     - Advanced filter in repeating group
     - Custom state with day calculation
     - Or: Pre-filter on server side via API workflow
   ```

4. **Display Results in Repeating Group**
   ```
   Type of content: AppointmentSlot
   Data source: Result of step 2
   
   Each cell shows:
     - Date (formatted): Current cell's AppointmentSlot's date
     - Day of week: Current cell's AppointmentSlot's date:formatted as 'dddd'
     - Time: Current cell's AppointmentSlot's start_time (converted to HH:MM)
     - Duration: Current cell's AppointmentSlot's duration
     - Instructor: Current cell's AppointmentSlot's availability_rule's instructor's name
     - Location: Current cell's AppointmentSlot's location's name
     - Service: Current cell's AppointmentSlot's availability_rule's service's name
     - "Book Now" button
   ```

5. **Handle No Results**
   ```
   Conditional element:
   When: RepeatingGroup's list of AppointmentSlots:count = 0
   Show: "No available slots found. Try adjusting your search criteria."
   ```

**Business Logic:**
- Search results show only future slots (date >= current date)
- Maximum search range: 6-8 weeks (configurable)
- Results limited to first 100 slots (performance consideration)
- Cache search results in custom state to avoid repeated searches

**Performance Optimization:**
- Use pagination for large result sets
- Index database on: date, is_booked, location
- Consider server-side search API workflow for complex filters

---

### 2. Book Appointment Workflow

**Trigger:** Button click on "Book Now" next to an available slot

**Purpose:** Creates a booking for a selected child and slot

**Steps:**

1. **Open Booking Confirmation Popup**
   ```
   Action: Set state
   State: selected_slot
   Value: Current cell's AppointmentSlot
   
   Action: Show element
   Element: BookingConfirmationPopup
   ```

2. **Display Booking Form in Popup**
   ```
   Data to display:
     - Slot date: Parent group's selected_slot's date
     - Slot time: Parent group's selected_slot's start_time
     - Duration: Parent group's selected_slot's duration
     - Instructor: Parent group's selected_slot's availability_rule's instructor's name
     - Location: Parent group's selected_slot's location's name
     - Service: Parent group's selected_slot's availability_rule's service's name
   
   Input fields:
     - Dropdown: Select child (options = Current User's children)
     - Text area: Special notes (optional)
   ```

3. **Validate Before Booking**
   ```
   Validations:
     - Child selected (Dropdown is not empty)
     - Slot still available (re-check is_booked = no)
     - No conflicting bookings for same child at same time
   
   Conflict check:
   Search for Bookings:count = 0
   Constraints:
     - child = Dropdown's value
     - appointment_slot's date = Parent group's selected_slot's date
     - appointment_slot's start_time <= Parent group's selected_slot's start_time + duration
     - appointment_slot's end_time >= Parent group's selected_slot's start_time
   ```

4. **Create Booking Record**
   ```
   Action: Create a new thing
   Type: Booking
   Fields:
     - appointment_slot = Parent group's selected_slot
     - parent = Current User
     - child = ChildDropdown's value
     - service = Parent group's selected_slot's availability_rule's service
     - status = "confirmed"
     - created_at = Current date/time
     - notes = TextArea's value
   ```

5. **Update Appointment Slot**
   ```
   Action: Make changes to thing
   Thing: Parent group's selected_slot
   Fields:
     - booking = Result of step 4 (the new Booking)
     - is_booked = yes
   ```

6. **Update Child's Service (Optional)**
   ```
   Action: Make changes to thing
   Thing: ChildDropdown's value
   Fields:
     - current_service = Parent group's selected_slot's availability_rule's service
   
   Note: This tracks what service the child is currently enrolled in
   ```

7. **Send Confirmation Email**
   ```
   Action: Schedule API Workflow (optional)
   Workflow: send_booking_confirmation
   Parameters:
     - booking_id = Result of step 4's unique id
     - recipient = Current User's email
   ```

8. **Show Success Message and Refresh**
   ```
   Action: Hide element (BookingConfirmationPopup)
   Action: Show alert
   Message: "Booking confirmed! You'll receive a confirmation email shortly."
   Action: Navigate to page
   Destination: my_bookings
   ```

**Critical Business Logic:**

**Race Condition Prevention:**
- Before creating booking, re-check that slot's is_booked = no
- If slot was just booked by another user, show error: "Sorry, this slot was just booked. Please select another time."
- Use Bubble's built-in optimistic locking (create only when condition is met)

**Constraint on Create Booking:**
```
Only when: Parent group's selected_slot's is_booked = no
```

**Conflict Prevention:**
- Parent cannot book overlapping slots for same child
- System checks for time conflicts before allowing booking
- Show clear error message if conflict detected

**Error Handling:**
- Handle payment failures (if payment integration exists)
- Roll back booking if any step fails
- Clear selected_slot state to prevent duplicate bookings

---

### 3. View My Bookings Workflow

**Trigger:** Page load on "My Bookings" page or navigation from menu

**Purpose:** Display all bookings for the current parent user

**Steps:**

1. **Load Parent's Bookings**
   ```
   Search: Search for Bookings
   Constraints:
     - parent = Current User
     - status is not "cancelled_by_parent" (optional: show all including cancelled)
   Sort by: appointment_slot's date (descending - most recent first)
   ```

2. **Display Bookings in Repeating Group**
   ```
   Type of content: Booking
   Data source: Result of step 1
   
   Each cell shows:
     - Child name: Current cell's Booking's child's name
     - Date: Current cell's Booking's appointment_slot's date (formatted)
     - Time: Current cell's Booking's appointment_slot's start_time (formatted)
     - Instructor: Current cell's Booking's appointment_slot's availability_rule's instructor's name
     - Location: Current cell's Booking's appointment_slot's location's name
     - Service: Current cell's Booking's service's name
     - Status badge: Current cell's Booking's status
     - "View Details" button
     - "Cancel" button (conditional: only if date is in future)
   ```

3. **Status Badge Conditionals**
   ```
   Upcoming (date >= current date):
     Background: Blue
     Text: "Upcoming"
   
   Completed (date < current date):
     Background: Gray
     Text: "Completed"
   
   Cancelled:
     Background: Red
     Text: "Cancelled"
   ```

4. **Filter Controls**
   ```
   Dropdown: Filter by child (All children, or specific child)
   Dropdown: Filter by status (All, Upcoming, Completed, Cancelled)
   Date range: Show bookings within date range
   ```

5. **Apply Filters (Advanced Filter on Repeating Group)**
   ```
   Advanced filter:
   - If ChildFilter is not "All": This Booking's child = ChildFilter's value
   - If StatusFilter is not "All": This Booking's status = StatusFilter's value
   - If DateRange is set: This Booking's appointment_slot's date is in range
   ```

**Interactive Elements:**
- Click "View Details" → Show booking details popup (similar to instructor view)
- Click "Cancel" → Trigger cancel booking workflow
- Export to calendar button → Generate .ics file for download

---

### 4. Cancel Booking Workflow (Parent-Initiated)

**Trigger:** Button click on "Cancel" next to a future booking

**Purpose:** Allows parent to cancel their child's booking

**Steps:**

1. **Cancellation Policy Check**
   ```
   Calculate hours until appointment:
   Hours = (Current cell's Booking's appointment_slot's date + start_time) - Current date/time
   
   If Hours < 24:
     Show warning: "Cancellations within 24 hours may incur a fee. Continue?"
   ```

2. **Confirmation Dialog**
   ```
   Action: Show alert
   Message: "Cancel booking for [Child's name] on [Date] at [Time]?"
   Style: Yes/No
   ```

3. **Update Booking Status**
   ```
   Action: Make changes to thing
   Thing: Current cell's Booking
   Fields:
     - status = "cancelled_by_parent"
     - cancelled_at = Current date/time
   ```

4. **Free Up Appointment Slot**
   ```
   Action: Make changes to thing
   Thing: Current cell's Booking's appointment_slot
   Fields:
     - booking = empty (clear relationship)
     - is_booked = no
   ```

5. **Notify Instructor**
   ```
   Action: Schedule API Workflow
   Workflow: send_cancellation_notification_to_instructor
   Parameters:
     - booking_id = Current cell's Booking's unique id
     - instructor_id = Current cell's Booking's appointment_slot's availability_rule's instructor's unique id
   ```

6. **Show Confirmation**
   ```
   Action: Show alert
   Message: "Booking cancelled successfully. The instructor has been notified."
   Action: Reset relevant elements (refresh repeating group)
   ```

**Business Logic:**
- Cancellations more than 24 hours in advance: Full refund (if applicable)
- Cancellations within 24 hours: Partial refund or fee
- Slot becomes immediately available for other parents to book
- Track cancellation reason (optional survey popup)

---

### 5. Manage Children Workflow

**Trigger:** Button click on "Add Child" or "Edit Child" on profile/settings page

**Purpose:** Allows parents to add or edit their children's profiles

**Steps for Adding a Child:**

1. **Show Add Child Popup**
   ```
   Action: Show element
   Element: AddChildPopup
   ```

2. **Validate Child Information**
   ```
   Validations:
     - Name is not empty
     - Date of birth is not empty
     - Date of birth is in the past
     - Age calculated from DOB is > 0 and < 18 (configurable)
   ```

3. **Create Child Record**
   ```
   Action: Create a new thing
   Type: Children
   Fields:
     - parent = Current User
     - name = NameInput's value
     - date_of_birth = DOBPicker's value
     - notes = NotesTextArea's value (optional)
     - current_service = ServiceDropdown's value (optional - set if enrolling immediately)
     - created_at = Current date/time
   ```

4. **Show Success Message**
   ```
   Action: Hide element (AddChildPopup)
   Action: Show alert
   Message: "[Child's name] added successfully!"
   Action: Reset inputs
   ```

**Steps for Editing a Child:**

1. **Load Child Data into Form**
   ```
   Inputs pre-filled with:
     - Name: Current cell's Children's name
     - DOB: Current cell's Children's date_of_birth
     - Notes: Current cell's Children's notes
     - Service: Current cell's Children's current_service
   ```

2. **Update Child Record**
   ```
   Action: Make changes to thing
   Thing: Current cell's Children
   Fields:
     - name = NameInput's value
     - date_of_birth = DOBPicker's value
     - notes = NotesTextArea's value
     - current_service = ServiceDropdown's value
   ```

3. **Show Success Message**
   ```
   Action: Hide element (EditChildPopup)
   Action: Show alert
   Message: "Child information updated!"
   ```

**Business Logic:**
- Cannot delete child if they have any bookings (past or future)
- If child has bookings, can only "archive" (mark as inactive)
- Parent can add multiple children under one account
- Each child can be enrolled in different services

---

## Shared Workflows

### 1. User Registration Workflow

**Trigger:** Form submission on sign-up page

**Purpose:** Creates new user account and determines user role

**Steps:**

1. **Create User Account**
   ```
   Action: Sign the user up
   Email: EmailInput's value
   Password: PasswordInput's value
   ```

2. **Determine User Role**
   ```
   Radio button selection or toggle:
   - "I'm a parent looking to book lessons"
   - "I'm an instructor offering lessons"
   ```

3. **Conditional: Create InstructorProfile (if instructor)**
   ```
   Only when: RoleSelector's value = "instructor"
   
   Action: Create a new thing
   Type: InstructorProfile
   Fields:
     - user = Result of step 1 (new User)
     - name = NameInput's value
     - bio = empty (filled in later)
     - phone = PhoneInput's value (optional)
     - is_active = yes
     - created_at = Current date/time
   
   Action: Make changes to thing
   Thing: Result of step 1 (new User)
   Fields:
     - instructor_profile = Result of InstructorProfile creation
   ```

4. **Welcome Email**
   ```
   Action: Schedule API Workflow
   Workflow: send_welcome_email
   Parameters:
     - user_id = Result of step 1's unique id
     - role = RoleSelector's value
   ```

5. **Navigate to Appropriate Dashboard**
   ```
   If instructor:
     Destination: instructor_dashboard
   Else:
     Destination: parent_dashboard (or search page)
   ```

**Business Logic:**
- Email must be unique (Bubble handles this automatically)
- Password must meet security requirements (min 8 characters)
- Phone number validation (optional but recommended)
- Can create instructor profile later if user signs up as parent first

---

### 2. User Login Workflow

**Trigger:** Form submission on login page

**Purpose:** Authenticates user and redirects to appropriate dashboard

**Steps:**

1. **Authenticate User**
   ```
   Action: Log the user in
   Email: EmailInput's value
   Password: PasswordInput's value
   ```

2. **Check User Role and Redirect**
   ```
   If Current User's instructor_profile is not empty:
     Destination: instructor_dashboard
   Else:
     Destination: parent_dashboard
   ```

**Error Handling:**
- Invalid credentials: Show error "Invalid email or password"
- Account locked: Show error and provide password reset link
- Email not verified: Prompt to resend verification email

---

### 3. Profile Edit Workflow

**Trigger:** Button click on "Save Profile" on settings/profile page

**Purpose:** Updates user's basic information

**Steps:**

1. **Update User Record**
   ```
   Action: Make changes to thing
   Thing: Current User
   Fields:
     - name = NameInput's value
     - email = EmailInput's value (requires re-authentication)
     - phone = PhoneInput's value
   ```

2. **Conditional: Update InstructorProfile (if instructor)**
   ```
   Only when: Current User's instructor_profile is not empty
   
   Action: Make changes to thing
   Thing: Current User's instructor_profile
   Fields:
     - name = NameInput's value
     - bio = BioTextArea's value
     - phone = PhoneInput's value
   ```

3. **Show Success Message**
   ```
   Action: Show alert
   Message: "Profile updated successfully!"
   ```

**Business Logic:**
- Email changes require email verification
- Cannot change email if it's already in use by another account
- Phone number formatting validation

---

## Workflow Execution Order

### Understanding Bubble's Workflow Processing

Workflows in Bubble execute sequentially, step by step, within the same workflow. However:

1. **Synchronous Steps:** Most actions execute immediately
2. **Asynchronous Steps:** API workflow schedules execute separately
3. **Custom Events:** Can be triggered but execute in their own context

### Example: Create Availability Rule Workflow Execution

```
1. User clicks "Create Rule" button (t=0ms)
2. Validation checks run (t=5ms)
3. AvailabilityRule record created (t=50ms)
4. API workflow scheduled (t=60ms)
   └─> API workflow runs separately (t=70ms to t=several minutes)
5. Success message shown (t=65ms)
6. Navigation occurs (t=70ms)
7. User sees new page while slots generate in background
```

### Race Condition Awareness

**Critical Scenario: Booking Workflow**

Two parents try to book the same slot simultaneously:

```
Parent A clicks "Book Now" (t=0ms)
Parent B clicks "Book Now" (t=5ms)

Parent A: Check is_booked (t=10ms) → FALSE ✓
Parent B: Check is_booked (t=15ms) → FALSE ✓

Parent A: Create booking (t=20ms)
Parent B: Create booking (t=25ms) → CONFLICT!

Parent A: Update slot is_booked = yes (t=30ms)
Parent B: Update slot is_booked = yes (t=35ms) → OVERWRITES!
```

**Solution:** Use conditional creation

```
Create Booking
Only when: Search for AppointmentSlots:count > 0
          where unique id = selected_slot's unique id
          and is_booked = no

If condition fails, show error: "Slot just booked by another user"
```

---

## Error Handling

### Common Error Scenarios and Solutions

#### 1. Network Timeout During Booking

**Scenario:** User clicks "Book Now" but network is slow

**Solution:**
```
Add loading state:
- Disable "Book Now" button immediately on click
- Show spinner: "Processing your booking..."
- Timeout after 30 seconds with error message
- Allow retry with same data
```

#### 2. API Workflow Fails to Schedule

**Scenario:** generate_all_slots fails to schedule due to server load

**Solution:**
```
Workflow step:
1. Schedule API workflow
2. Check result of step 1 (Bubble provides success/failure)
3. If failed:
   - Log error to database (Error Log data type)
   - Show user: "Slots are being generated. This may take a few minutes."
   - Retry scheduling after 30 seconds (use recursive check)
```

#### 3. Validation Failures

**Scenario:** User submits form with invalid data

**Solution:**
```
Client-side validation:
- Real-time input validation (as user types)
- Disable submit button until all fields valid
- Show inline error messages in red below inputs

Server-side validation:
- Re-validate in workflow before creating records
- Show alert with specific error message
- Don't navigate away on validation failure
```

#### 4. Booking Confirmation Race Condition

**Scenario:** Slot gets booked between search and booking click

**Solution:**
```
Booking workflow step 1:
Search for AppointmentSlots:count > 0
Constraints:
  - unique id = selected_slot's unique id
  - is_booked = no

If count = 0:
  Show alert: "Sorry, this slot was just booked. Refreshing available slots..."
  Reset custom state
  Refresh repeating group
  Don't proceed with booking
```

#### 5. Recursive Workflow Infinite Loop

**Scenario:** generate_day workflow doesn't increment date properly

**Solution:**
```
In generate_day workflow:
Parameter: current_date (type: date)

Step 1: Check termination condition
Only when: current_date < availability_rule's end_date

Step 2: Create slots for current_date

Step 3: Schedule next iteration
Schedule API workflow: generate_day
Parameters:
  - current_date = This workflow's current_date + days: 1
             (NOT +days: 0 !)
  - availability_rule_id = This workflow's availability_rule_id
```

---

## Workflow Performance Optimization

### 1. Minimize Database Queries

**Bad Practice:**
```
Repeating Group displays:
- Current cell's appointment_slot's availability_rule's instructor's name
- Current cell's appointment_slot's availability_rule's location's name
- Current cell's appointment_slot's availability_rule's service's name

Each cell makes 3 separate database queries!
```

**Good Practice:**
```
Set data source with :merged with to pre-load relationships:

Search for AppointmentSlots
:merged with Search for AppointmentSlots's availability_rule's instructor
:merged with Search for AppointmentSlots's availability_rule's location

This loads all data in one query.
```

### 2. Use Custom States for Frequently Accessed Data

**Bad Practice:**
```
Every button references "Search for AvailabilityRules where instructor = Current User's instructor_profile"
```

**Good Practice:**
```
On page load:
Set state "current_instructor_rules" = Search for AvailabilityRules where instructor = Current User's instructor_profile

In workflows:
Reference: Parent group's current_instructor_rules
```

### 3. Paginate Large Lists

**Bad Practice:**
```
Repeating Group shows:
All AppointmentSlots where date >= current date
(Could be thousands of slots!)
```

**Good Practice:**
```
Repeating Group shows:
Search for AppointmentSlots:items until #100
where date >= current date

Add "Load More" button at bottom
```

### 4. Debounce Search Inputs

**Bad Practice:**
```
Input element triggers search on every keystroke
User types "swimming" → 8 searches executed!
```

**Good Practice:**
```
Input element triggers search:
Only when: Input's value:length > 2
and after Input hasn't changed for 500ms

Or use a "Search" button instead of auto-search
```

---

## Workflow Testing Checklist

### Before Deploying Workflows to Production

- [ ] Test all happy path scenarios (normal user flow)
- [ ] Test validation errors (invalid inputs)
- [ ] Test race conditions (multiple users booking same slot)
- [ ] Test with slow network (throttle connection)
- [ ] Test on mobile devices (touch interactions)
- [ ] Test with empty data (new users, no bookings yet)
- [ ] Test with maximum data (100+ slots, 50+ bookings)
- [ ] Test API workflow recursion (verify it terminates)
- [ ] Test cancellation flows (both parent and instructor)
- [ ] Verify email notifications are sent
- [ ] Test across different time zones
- [ ] Test accessibility (keyboard navigation, screen readers)

---

## Business Rules Summary

### Booking Rules
1. Parent cannot book overlapping slots for same child
2. Slots can only be booked if is_booked = no and is_active = yes
3. Parent can book multiple children in same time slot
4. Bookings cannot be made for past dates/times

### Cancellation Rules
1. Cancellations >24 hours: Full refund (if applicable)
2. Cancellations <24 hours: Partial refund or fee
3. Instructor can cancel with required notification
4. Cancelled slots immediately become available

### Availability Rules
1. Rules with bookings cannot be deleted (soft delete only)
2. End date must be >= start date
3. End time must be > start time
4. Days of week must include at least one day
5. Duration must be > 0 and fit within time range

### Slot Generation Rules
1. Slots generated in 10-minute increments (configurable)
2. Only generated for selected days of week
3. Respect buffer time between slots
4. Stop generation at end_date (inclusive)
5. Skip dates where rule is not active

---

## Conclusion

This workflow documentation captures all interactive behaviors in the swim lesson booking application. Each workflow is designed with:

- Clear trigger conditions
- Step-by-step execution logic
- Business rule enforcement
- Error handling and validation
- Performance considerations
- Race condition prevention

When implementing new workflows, follow these patterns to maintain consistency and reliability across the application.
