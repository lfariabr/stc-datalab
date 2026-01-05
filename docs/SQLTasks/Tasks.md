Below you'll find three practice prompts tied directly to the project’s reporting goals and interview prep themes:

1. **Student Support Follow-up**
List students who still have medical alerts (`medical_info IS NOT NULL`) but haven’t been marked present in any class during the last 7 days. Include their homeroom teacher and phone.
Tables to combine: `Students`, `Enrollments`, `Classes`, `Staff`, `Attendance`. Focus on LEFT joins so students without recent attendance still show up.

```sql
SELECT
  s.student_id,
  s.first_name,
  s.medical_info,
  t.first_name AS teacher_first_name,
  t.phone      AS teacher_phone
FROM Students s
LEFT JOIN Enrollments e ON e.student_id = s.student_id
LEFT JOIN Classes c     ON c.class_id = e.class_id
LEFT JOIN Staff t       ON t.staff_id = c.staff_id
WHERE s.medical_info IS NOT NULL
  AND NOT EXISTS (
    SELECT 1
    FROM Attendance a
    WHERE a.student_id = s.student_id
      AND a.attendance_date >= DATEADD(DAY, -7, GETDATE())
      AND a.status = 'Present'
  );
```

2. **Capacity Planning Drill**
For Year 10 classes this semester, show: class name, teacher, max capacity, active enrollment count, available seats, and a status label (`Full`, `Near Capacity`, `Under-enrolled`, `Available`) using the same thresholds as sp_EnrollmentSummaryByYear.
Hint: Recreate the CASE logic from the stored procedure to prove you can rebuild operational reporting outside stored procs if needed.

3. **Effort vs. Outcomes Insight**
Using `vw_AcademicPerformance`, find students whose effort rating is `Outstanding` but whose academic standing is `At Risk` or `Failing`. Return student name, class, attendance rate %, effort rating, academic standing.
Goal: Practice filtering a view that already encapsulates grade mapping and attendance-derived effort metrics.