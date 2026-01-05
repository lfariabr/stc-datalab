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
```sql
-- Capacity Planning Drill (matches sp_EnrollmentSummaryByYear logic)
SELECT
    c.class_id,
    c.class_name,
    CONCAT(st.first_name, ' ', st.last_name) AS teacher_name,
    c.max_students,

    COUNT(DISTINCT CASE WHEN e.status = 'Active' THEN e.student_id END) AS active_students,

    c.max_students
      - COUNT(DISTINCT CASE WHEN e.status = 'Active' THEN e.student_id END) AS available_seats,

    CAST(
      ROUND(
        100.0 * COUNT(DISTINCT CASE WHEN e.status = 'Active' THEN e.student_id END)
        / NULLIF(c.max_students, 0),
      2) AS DECIMAL(5,2)
    ) AS capacity_utilization_percent,

    CASE
      WHEN COUNT(DISTINCT CASE WHEN e.status = 'Active' THEN e.student_id END) >= c.max_students
        THEN 'Full'
      WHEN COUNT(DISTINCT CASE WHEN e.status = 'Active' THEN e.student_id END) >= c.max_students * 0.9
        THEN 'Near Capacity'
      WHEN COUNT(DISTINCT CASE WHEN e.status = 'Active' THEN e.student_id END) <  c.max_students * 0.5
        THEN 'Under-enrolled'
      ELSE 'Available'
    END AS status_label

FROM Classes c
INNER JOIN Staff st
  ON c.staff_id = st.staff_id
LEFT JOIN Enrollments e
  ON e.class_id = c.class_id
-- If enrollments are semester-scoped, filter here to preserve empty classes:
-- AND e.semester = c.semester

WHERE c.year_level = 10
--   AND c.semester = @Semester          -- or whatever your semester filter is
  AND c.is_active = 1

GROUP BY
    c.class_id, c.class_name, st.first_name, st.last_name, c.max_students

ORDER BY c.class_name;
```

3. **Effort vs. Outcomes Insight**
Using `vw_AcademicPerformance`, find students whose effort rating is `Outstanding` but whose academic standing is `At Risk` or `Failing`. Return student name, class, attendance rate %, effort rating, academic standing.
Goal: Practice filtering a view that already encapsulates grade mapping and attendance-derived effort metrics.