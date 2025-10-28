# Swim Lesson Booking App - Complete Documentation

## 📋 Documentation Status

**Last Updated:** October 28, 2025  
**Status:** ✅ Aligned with production Bubble.io implementation  
**Coverage:** Complete system documentation including database, workflows, and business logic

---

## 🎯 What's Documented

This repository contains **comprehensive documentation** of a multi-week swim lesson booking application built in Bubble.io.

### ✅ Completed Documentation

1. **[02-data-structure-CORRECTED.md](./02-data-structure-CORRECTED.md)**
   - All 12 data types with complete field definitions
   - Actual field types as implemented (dates for times, not numbers)
   - Relationships and business rules
   - Multi-week booking support
   - Billing and payment integration (Stripe)

2. **[04-workflows.md](./04-workflows.md)** ⚠️ *Needs Update*
   - Front-end button workflows
   - Page load workflows
   - User interactions
   - *Note: Written for original design, needs update for actual implementation*

3. **[05-api-workflows.md](./05-api-workflows.md)** ⚠️ *Needs Update*
   - Backend recursive workflows
   - Slot generation system
   - Multi-week booking creation
   - *Note: Written for original design, needs update for actual implementation*

4. **[07-business-logic.md](./07-business-logic.md)** ⚠️ *Needs Update*
   - Business rules and constraints
   - Decision rationale
   - Edge case handling
   - *Note: Written for original design, needs update for actual implementation*

5. **[09-troubleshooting.md](./09-troubleshooting.md)** ⚠️ *Needs Update*
   - Common issues and solutions
   - Debugging guides
   - Known bugs and workarounds
   - *Note: Written for original design, needs update for actual implementation*

### 📊 Analysis Documents (Reference)

- **[actual-workflow-analysis.md](./actual-workflow-analysis.md)** - Backend workflow breakdown
- **[frontend-workflow-analysis.md](./frontend-workflow-analysis.md)** - UI workflow breakdown
- **[database-comparison.md](./database-comparison.md)** - Original vs. actual schema comparison

---

## 🏗️ System Architecture

### Key Features

✅ **Multi-Week Booking Packages**
- Parents book 4, 6, or 8-week packages in one transaction
- All bookings linked via `booking_group_id`
- Automatic slot reservation for recurring weekly lessons

✅ **Recursive Slot Generation**
- Instructors define recurring availability (e.g., Mon/Wed/Fri, 10 AM - 6 PM)
- System automatically generates 10-minute bookable slots
- Efficient backend workflows prevent timeouts

✅ **Stripe Payment Integration**
- Full payment processing
- Invoice management
- Deposit and payment plan support

✅ **Flexible Scheduling**
- Multiple pools and locations
- Variable lesson durations (20, 30, 40, 60 minutes)
- Instructor-specific availability

---

## 📐 Data Model Overview

### Core Entities

```
User (Parent/Instructor)
  ↓
InstructorProfile → AvailabilityRule → AppointmentSlot
  ↓                                           ↓
Children ─────────────────────────→ Booking (Multi-Week)
  ↓                                           ↓
Services ──────────────────────────────→ Invoice/Billing
```

### Critical Design Decisions

#### 1. **Time Storage: Dates, Not Numbers**

**AppointmentSlot & Booking:**
```
start_time: date = "Jan 15, 2025 10:00:00 AM"  ✓ Actual implementation
NOT: start_time: number = 600  ✗ Original plan
```

**Why?** Easier to work with in Bubble, direct formatting, no conversion needed

**AvailabilityRule:**
```
start_time: number = 600 (minutes from midnight)  ✓ For calculations
start_time_display: text = "10:00 AM"  ✓ For UI display
```

**Why?** Defines abstract recurring pattern, needs numeric calculations

#### 2. **Days of Week: Dual Storage**

```
days_of_week_numbers: [1, 3, 5]  ← List of numbers for logic
days_of_week_display: "Mon, Wed, Fri"  ← Text for UI
Passed to workflows as: "1,3,5"  ← Comma-separated text
```

**Why?** Different representations for different purposes (UI, logic, parameters)

#### 3. **Display Fields: Pre-Calculated**

```
start_time: 600
start_time_display: "10:00 AM"  ← Stored from dropdown at creation
```

**Why?** Performance - avoid repeated formatting calculations on page loads

#### 4. **Denormalization: Data Copying**

AppointmentSlot copies from AvailabilityRule:
- instructor
- pool
- services

**Why?** Preserves data if AvailabilityRule deleted, improves query performance

---

## 🔄 Workflow System

### Slot Generation Flow

```
1. Instructor fills form → Clicks "Generate Slots"
   ↓
2. Create AvailabilityRule (with all fields)
   ↓
3. Schedule generate_all API workflow
   ↓
4. generate_all checks each date (Jan 1, Jan 2, Jan 3...)
   ├─ Does Jan 1 match days_of_week? YES → Schedule generate_day
   ├─ Does Jan 2 match days_of_week? NO → Skip
   ├─ Does Jan 3 match days_of_week? YES → Schedule generate_day
   ↓
5. generate_day creates numeric list [0, 10, 20, 30... 470]
   ↓
6. Schedule create_slot for each number (48 times per day)
   ↓
7. create_slot creates AppointmentSlot with date-type times
```

**Result:** 13 matching days × 48 slots = ~624 bookable slots

### Multi-Week Booking Flow

```
1. Parent selects slot → Chooses 4-week service → Clicks "Book"
   ↓
2. book_lessons API workflow called (weeks = 4)
   ↓
3. create_booking_for_week (Week 0)
   ├─ Search for slot on Jan 6
   ├─ Create Booking #1
   ├─ Update slot: is_available = no
   └─ Schedule create_booking_for_week (Week 1)
   ↓
4. create_booking_for_week (Week 1)
   ├─ Search for slot on Jan 13
   ├─ Create Booking #2
   └─ Schedule create_booking_for_week (Week 2)
   ↓
5. Continues for weeks 2, 3...
   ↓
6. Stop when current_week >= total_weeks
```

**Result:** 4 Bookings created, all with same booking_group_id

---

## 💡 Key Insights

### What Makes This Implementation Unique

1. **Recursive API Workflows** - Generates thousands of slots without timeouts
2. **Multi-Week Packages** - Parents book entire sessions, not single lessons
3. **Hybrid Time Storage** - Numbers for calculations, dates for instances
4. **Booking Groups** - Links related bookings across weeks
5. **Denormalized Data** - Performance optimized, data preserved

### Design Trade-offs

| Decision | Pro | Con |
|----------|-----|-----|
| Times as dates | Easy to use, no conversion | Less flexible than numbers |
| Display fields | Fast page loads | Redundant data, manual updates |
| Denormalization | Data preserved, fast queries | Updates don't propagate |
| Multi-week only | Simple UX, packages sell better | Less flexibility for single lessons |
| 10-min slots | Fine-grained control | Many database records |

---

## 🚀 Getting Started

### For Developers

1. **Read data structure first:** [02-data-structure-CORRECTED.md](./02-data-structure-CORRECTED.md)
2. **Understand workflows:** [actual-workflow-analysis.md](./actual-workflow-analysis.md)
3. **Review business logic:** [07-business-logic.md](./07-business-logic.md)
4. **Check troubleshooting:** [09-troubleshooting.md](./09-troubleshooting.md)

### For Product/Business

1. **Multi-week packages:** See Services data type in [02-data-structure-CORRECTED.md](./02-data-structure-CORRECTED.md)
2. **Pricing models:** See Services pricing section
3. **Cancellation policy:** See [07-business-logic.md](./07-business-logic.md)
4. **Payment integration:** See Billing/Invoice data types

---

## ⚠️ Known Issues

### Critical
1. ✅ **Booking workflow creates slots for non-existent dates** - FIXED
   - Was creating slots when it should search
   - Fixed by using `:items until #` to limit results

### Minor
1. **Pool.instructor field should be List** - Currently single User, should be List of InstructorProfiles
2. **Time display in 24-hour on some devices** - Browser-dependent
3. **Redundant fields** - Some fields duplicate data for performance

---

## 📝 Documentation TODO

### High Priority
- [ ] Update 04-workflows.md to match actual implementation
- [ ] Update 05-api-workflows.md with correct time types
- [ ] Update 07-business-logic.md with multi-week logic
- [ ] Update 09-troubleshooting.md with correct field types

### Medium Priority
- [ ] Add Billing/Invoice workflow documentation
- [ ] Add Stripe integration documentation
- [ ] Add user guide for instructors
- [ ] Add user guide for parents

### Low Priority
- [ ] Add deployment guide
- [ ] Add testing checklist
- [ ] Add performance optimization guide
- [ ] Add security best practices

---

## 🤝 Contributing

This documentation represents the **actual production implementation** as of October 28, 2025.

When making changes:
1. Update Bubble.io application first
2. Update documentation to match
3. Note any breaking changes
4. Update this README with status

---

## 📄 License

[Your License Here]

---

## 📞 Contact

[Your Contact Information]

---

## 🔗 Related Resources

- [Bubble.io Documentation](https://manual.bubble.io/)
- [Stripe API Documentation](https://stripe.com/docs/api)
- [Live Application](https://your-app.bubbleapps.io/)

---

**Note:** Files marked with ⚠️ were written based on original design discussions and need to be updated to match the actual implementation documented in `02-data-structure-CORRECTED.md` and the analysis files.
