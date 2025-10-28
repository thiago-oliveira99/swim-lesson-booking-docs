# Quick Reference - Swim Lesson Booking App

## 🎯 Files Ready for GitHub

### ✅ UPLOAD NOW:
1. **README-GITHUB.md** → Rename to `README.md`
2. **02-data-structure-CORRECTED.md** → Rename to `docs/02-data-structure.md`
3. **actual-workflow-analysis.md** → Place in `docs/analysis/`
4. **frontend-workflow-analysis.md** → Place in `docs/analysis/`
5. **database-comparison.md** → Place in `docs/analysis/`

### ⏳ NEEDS UPDATE BEFORE UPLOAD:
- 04-workflows.md
- 05-api-workflows.md
- 07-business-logic.md
- 09-troubleshooting.md

---

## 📐 Key Architecture Facts

### Time Storage
**AvailabilityRule:** Numbers (600 = 10 AM)  
**AppointmentSlot:** Dates (Jan 15, 2025 10:00 AM)  
**Booking:** Dates (Jan 15, 2025 10:00 AM)

### Days of Week
**Stored:** List [1, 3, 5]  
**Displayed:** "Mon, Wed, Fri"  
**Passed to workflows:** "1,3,5" (text)

### Multi-Week Bookings
**Parent books:** 4-week package  
**System creates:** 4 separate Booking records  
**Linked by:** booking_group_id

---

## 🔄 Critical Workflows

### Slot Generation
```
generate_all → generate_day → create_slot (×48)
Result: ~624 slots for Mon/Wed/Fri 10AM-6PM over 4 weeks
```

### Multi-Week Booking
```
book_lessons → create_booking_for_week (×4)
Result: 4 Bookings, all same booking_group_id
```

---

## 💾 Database

**12 Data Types:**
1. User
2. InstructorProfile
3. Child
4. Pool
5. Services
6. AvailabilityRule
7. AppointmentSlot ← Generated automatically
8. Booking ← Created on booking
9. Timeslot ← Helper for dropdowns
10. Billing ← Payment tracking
11. Invoice ← Simplified billing
12. BookingSession ← Booking flow state

---

## 🔗 Key Links

- **All docs:** `/mnt/user-data/outputs/`
- **Upload guide:** `GITHUB-UPLOAD-GUIDE.md`
- **This reference:** `QUICK-REFERENCE.md`

---

## ⚡ Quick Commands

### View all files:
```bash
ls -lh /mnt/user-data/outputs/
```

### Download specific file:
Use computer:// links provided in chat

---

Made with ❤️ by Claude
