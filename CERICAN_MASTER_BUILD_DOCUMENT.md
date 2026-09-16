# CERICAN — MASTER BUILD DOCUMENT
## School Management System · Ghana Education Service Framework
### Single Source of Truth for Google AI Studio / Gemini Implementation

> **This document supersedes all prior versions.** It combines the V2 Implementation Specification (authoritative source of truth), the Complete Build Blueprint (technical foundation), and the UX Analysis Report (usability requirements) into one unified specification. Treat every rule here as mandatory unless explicitly marked as configurable.

---

# PART 0 — MASTER PROMPT FOR GEMINI

Paste this at the start of every Google AI Studio session:

```
You are implementing Cerican, a production school management system for Ghanaian
basic and JHS schools (Ghana Education Service framework).

THIS DOCUMENT IS THE AUTHORITATIVE SOURCE OF TRUTH. Do not reintroduce functionality
from any older version when it conflicts with this document.

CORE RULES:
1. Stack: Next.js 14 (App Router), Supabase/PostgreSQL, Supabase Auth, Tailwind CSS, Zod.
2. Build mobile-first at ~390px before optimizing desktop.
3. Enforce all authorization on the server and via Supabase RLS. Never rely on hidden UI.
4. NEVER use SELECT COUNT(*) + 1 for student or staff IDs. Use the id_sequences table
   with transactional row-locking.
5. NEVER generate fallback IDs in the frontend. If ID generation fails, return a server error.
6. Store historical class placement in student_enrollments, not just students.class_id.
7. Normalize parents/guardians: one guardian record can link to multiple children via
   student_guardians.
8. Implement real aggregate calculation using aggregate_rules tables.
9. NEVER silently treat a missing score as zero. Null is intentional and incomplete.
10. Implement grades server-side with validated grading bands.
11. Recompute subject positions transactionally at scoresheet submission, not on every keystroke.
12. Scoresheet status is an explicit state machine: DRAFT → IN_PROGRESS → SUBMITTED →
    REOPENED → LOCKED.
13. Teachers MUST NOT delete submitted/locked scores.
14. Attendance is a real core module. Missing attendance ≠ low attendance.
15. Finance uses fee_assessments + payment transactions. Never overwrite payment history.
16. Reports have readiness checks, preview, publish/finalize, and version snapshots.
17. PDF generation DOES NOT equal academic finalization. These are separate events.
18. THE REMARKS ENGINE IS FULLY DETERMINISTIC. There are NO runtime Gemini or Claude API
    calls for remarks. No AI prompt builder, no AI audit call, no /api/ai/remarks route,
    no model_used field in production remarks tables.
19. classifyStudent() is deterministic conditional business logic only.
20. Primary performance tier is exactly one of: STRUGGLING, BELOW_AVERAGE, AVERAGE,
    ABOVE_AVERAGE, HIGH_ACHIEVER.
21. Pick at most one secondary pattern using the defined priority order.
22. Return MANUAL_REVIEW_REQUIRED for incomplete academic profiles. Never fabricate.
23. Store remark templates in remark_templates. Store selection history in
    remark_generation_log.
24. Initial template selection is deterministic (stable key from student ID + pattern + slot).
    Regenerate cycles through stored variants — it does NOT call any API.
25. Validate rendered remark text AFTER placeholder substitution.
26. Never leak unknown placeholders into final remarks.
27. Never assume non-Female means Male. Support neutral language/Other gender.
28. Never hardcode subject groups by name. Use configured group membership in the database.
29. Make pattern thresholds configurable with safe defaults.
30. Build the mobile full-screen remarks editor with character counter, status badges,
    variant cycling, manual edit, save, and approve.
31. Build a report readiness dashboard before final generation.
32. Pre-fill active academic year/term from school_current_context on every form.
33. Never expose another student's, parent's, teacher's, or school's data.
34. Every form: client/server validation, loading, success, error, and empty states.
35. Every destructive action: confirmation dialog + server-side authorization.
36. Standardize all CTA labels. No vague GO / PROCEED / MORE / Submit.
37. All tables: card layout on mobile, controlled horizontal scroll with sticky first
    column for score-heavy tables.
38. Keep audit history for critical actions. NEVER log plaintext passwords or secrets.
39. Make bulk operations idempotent. Provide partial failure and retry handling.
40. Write NO TODOs or placeholder logic in production paths.
41. Every core business rule must have an automated test.
42. Output: production-ready TypeScript + SQL, proper migration ordering, RLS policies
    for every sensitive table, unit/integration tests for scoring/aggregates/positions/
    remarks/security/ID allocation, seed data for grading schemas and remark templates.
```

---

# PART 1 — NON-NEGOTIABLE PRINCIPLES

1. Cerican is a production system. Correctness, data integrity, security, auditability,
   mobile usability, and predictable workflows are non-negotiable.
2. The production remarks system is fully deterministic. No LLM call at runtime.
3. Every academic result must come from deterministic business rules and validated records.
4. Never invent missing academic data. Incomplete records produce an explicit review state.
5. The server/database is authoritative for scores, grades, positions, aggregates,
   permissions, financial balances, report status, and final remarks.
6. Never create fallback student IDs in frontend code.
7. No sensitive operation is trusted because a button is hidden in the UI.
8. No destructive/irreversible academic, financial, or security action happens from a
   single accidental tap.
9. The app is mobile-first at ~390px and must remain usable on desktop and tablet.
10. All critical workflows have explicit loading, success, error, empty, and
    confirmation states.
11. Preserve historical truth. Old reports, payments, class placements, and generated
    remarks remain auditable even after later edits.
12. Prefer configuration for school-specific rules; keep safe defaults in code.
13. Use explicit status/state machines instead of inferring state from whether a random
    field contains data.
14. Do not add features merely because they sound modern. Core workflows: student/staff
    management, academics, reports, remarks, attendance, finance, portals, notifications,
    audit, and promotion.

---

# PART 2 — SYSTEM ARCHITECTURE

```
                         CERICAN
                            |
        +-------------------+-------------------+
        |                   |                   |
      ADMIN              TEACHER           STUDENT/PARENT
        |                   |                   |
        +-------------------+-------------------+
                            |
                     DOMAIN SERVICES
                            |
       +-----------+--------+--------+------------+
       |           |                 |            |
    Scoring     Remarks           Reports      Finance
       |           |                 |            |
    Grades     Deterministic      PDF/Preview   Payments
    Positions   Templates         Versioning    Receipts
    Aggregates  Patterns           Publish       Arrears
       |           |                 |            |
       +-----------+--------+--------+------------+
                            |
                     SUPABASE / POSTGRES
                            |
                    RLS + Audit + Auth
```

### Architecture Principles

- **Mobile-first.** Every screen designed for 390px first.
- **Role-scoped routing.** Each panel is a separate layout with server-side auth guard.
- **Single source of truth.** One database, one ID system, multiple views per role.
- **No runtime AI for remarks.** Gemini/Claude are development assistants only.
- **Progressive enhancement.** Core functionality works; enhanced features layer on top.

---

# PART 3 — TECHNOLOGY STACK

| Layer | Choice | Reason |
|-------|--------|--------|
| Frontend | Next.js 14 (App Router) | SSR for fast mobile loads; file-based routing per role |
| Styling | Tailwind CSS | Mobile-first utility classes; fast to iterate |
| Database | Supabase (PostgreSQL) | Built-in auth; RLS; generous free tier |
| Auth | Supabase Auth | Role-based; custom claims for each role |
| PDF Generation | @react-pdf/renderer | Server-side report card PDF generation |
| Email/SMS | Resend (email) + Twilio (SMS) | Optional notification channels |
| File Storage | Supabase Storage | Student/staff photos, generated PDFs |
| Hosting | Vercel | Next.js native deployment |
| Charts | Recharts | Score trend charts; dashboards |
| Validation | Zod | Client and server validation |
| State | Zustand | Global school/term context |

**Do NOT introduce runtime LLM packages for remarks.**

### Key Dependencies (package.json)

```json
{
  "dependencies": {
    "next": "14.x",
    "react": "18.x",
    "tailwindcss": "3.x",
    "@supabase/supabase-js": "2.x",
    "@supabase/auth-helpers-nextjs": "0.x",
    "@react-pdf/renderer": "3.x",
    "recharts": "2.x",
    "resend": "2.x",
    "twilio": "4.x",
    "zod": "3.x",
    "date-fns": "3.x",
    "lucide-react": "0.x",
    "zustand": "4.x"
  }
}
```

---

# PART 4 — DATABASE SCHEMA

> **Migration order matters.** Create tables in dependency-safe order. Staff before Classes FK, etc.

## 4.1 Schools (Tenancy Root)

```sql
CREATE TABLE schools (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT NOT NULL,
  code        TEXT UNIQUE NOT NULL,      -- e.g. "MOGASCO01"
  address     TEXT,
  region      TEXT,
  district    TEXT,
  phone       TEXT,
  logo_url    TEXT,
  ges_circuit TEXT,
  created_at  TIMESTAMPTZ DEFAULT now()
);
```

## 4.2 Current Academic Context

**Do not use multiple `is_current` booleans.** Use a single authoritative context record per school.

```sql
CREATE TABLE school_current_context (
  school_id        UUID PRIMARY KEY REFERENCES schools(id) ON DELETE CASCADE,
  academic_year_id UUID NOT NULL REFERENCES academic_years(id),
  term_id          UUID NOT NULL REFERENCES terms(id),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## 4.3 Academic Years and Terms

```sql
CREATE TABLE academic_years (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id  UUID REFERENCES schools(id) ON DELETE CASCADE,
  label      TEXT NOT NULL,             -- e.g. "2025/2026"
  start_date DATE NOT NULL,
  end_date   DATE NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE terms (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  academic_year_id UUID REFERENCES academic_years(id) ON DELETE CASCADE,
  term_number      INTEGER CHECK (term_number IN (1, 2, 3)),
  start_date       DATE,
  end_date         DATE,
  vacation_date    DATE,
  reopening_date   DATE,
  total_school_days INTEGER
);
```

Constraints: Term dates must not contradict the academic year. Dates use `DATE` type, not timestamps.

## 4.4 ID Sequences (Replaces COUNT()+1 Trigger)

**Never use `SELECT COUNT(*) + 1` for IDs.** Use atomic transactional allocation.

```sql
CREATE TABLE id_sequences (
  school_id     UUID NOT NULL REFERENCES schools(id) ON DELETE CASCADE,
  seq_type      TEXT NOT NULL CHECK (seq_type IN ('student', 'staff')),
  current_value INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (school_id, seq_type)
);
```

Allocation procedure (must use `FOR UPDATE` row lock):

```sql
CREATE OR REPLACE FUNCTION allocate_next_id(
  p_school_id UUID,
  p_seq_type  TEXT
) RETURNS INTEGER AS $$
DECLARE
  next_val INTEGER;
BEGIN
  SELECT current_value + 1 INTO next_val
  FROM id_sequences
  WHERE school_id = p_school_id AND seq_type = p_seq_type
  FOR UPDATE;

  UPDATE id_sequences
  SET current_value = next_val
  WHERE school_id = p_school_id AND seq_type = p_seq_type;

  RETURN next_val;
END;
$$ LANGUAGE plpgsql;
```

## 4.5 Staff

Email is optional (not required for login). Never create fake emails.

```sql
CREATE TABLE staff (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id    UUID REFERENCES schools(id) ON DELETE CASCADE,
  staff_id_code TEXT,
  UNIQUE (school_id, staff_id_code),
  first_name   TEXT NOT NULL,
  surname      TEXT NOT NULL,
  other_names  TEXT,
  gender       TEXT CHECK (gender IN ('Male', 'Female', 'Other')),
  staff_type   TEXT DEFAULT 'Teacher',
  contact_one  TEXT,
  contact_two  TEXT,
  email        TEXT,
  photo_url    TEXT,
  date_joined  DATE,
  is_active    BOOLEAN DEFAULT true,
  must_change_password BOOLEAN DEFAULT true,
  last_login_at TIMESTAMPTZ,
  user_id      UUID,
  created_at   TIMESTAMPTZ DEFAULT now(),
  created_by   UUID,
  updated_at   TIMESTAMPTZ DEFAULT now(),
  updated_by   UUID
);
```

**On account creation:** Generate password securely, mark `must_change_password = true`, show one-time credential to admin, optionally email if configured. Never log plaintext passwords in audit logs.

## 4.6 Classes

```sql
CREATE TABLE class_divisions (
  id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID REFERENCES schools(id),
  name      TEXT NOT NULL,             -- "Preschool", "Primary", "JHS"
  levels    TEXT[]
);

-- Create classes WITHOUT class_teacher_id FK initially
CREATE TABLE classes (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id        UUID REFERENCES schools(id) ON DELETE CASCADE,
  name             TEXT NOT NULL,
  level_order      INTEGER,
  division_id      UUID REFERENCES class_divisions(id),
  max_students     INTEGER DEFAULT 40,
  created_at       TIMESTAMPTZ DEFAULT now()
);

-- Add FK after staff table exists
ALTER TABLE classes ADD COLUMN class_teacher_id UUID REFERENCES staff(id);
```

## 4.7 Students

```sql
CREATE TABLE students (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id       UUID REFERENCES schools(id) ON DELETE CASCADE,
  student_id_code TEXT,
  UNIQUE (school_id, student_id_code),
  surname         TEXT NOT NULL,
  first_name      TEXT NOT NULL,
  other_names     TEXT,
  gender          TEXT CHECK (gender IN ('Male', 'Female', 'Other')),
  date_of_birth   DATE,
  photo_url       TEXT,
  class_id        UUID REFERENCES classes(id),  -- current class only
  is_active       BOOLEAN DEFAULT true,
  user_id         UUID,
  enrollment_date DATE DEFAULT CURRENT_DATE,
  created_at      TIMESTAMPTZ DEFAULT now(),
  created_by      UUID,
  updated_at      TIMESTAMPTZ DEFAULT now(),
  updated_by      UUID
);
```

**ID generation:** Call `allocate_next_id()` in a server action, never in frontend code. Format: `{school_code}/{class_level_padded}/{year_suffix}/{sequence_padded}`.

## 4.8 Student Enrollment History

**Required for historical reporting.** Reports use enrollment history, not today's class.

```sql
CREATE TABLE student_enrollments (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  student_id       UUID NOT NULL REFERENCES students(id) ON DELETE CASCADE,
  academic_year_id UUID NOT NULL REFERENCES academic_years(id),
  class_id         UUID NOT NULL REFERENCES classes(id),
  enrollment_status TEXT NOT NULL DEFAULT 'active',
  start_date       DATE,
  end_date         DATE,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (student_id, academic_year_id)
);
```

## 4.9 Guardians / Parents (Normalized)

One guardian record links to multiple children.

```sql
CREATE TABLE guardians (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  full_name  TEXT NOT NULL,
  phone_one  TEXT,
  phone_two  TEXT,
  email      TEXT,
  user_id    UUID,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE student_guardians (
  student_id    UUID NOT NULL REFERENCES students(id) ON DELETE CASCADE,
  guardian_id   UUID NOT NULL REFERENCES guardians(id) ON DELETE CASCADE,
  relationship  TEXT NOT NULL,          -- "Mother", "Father", "Guardian"
  is_primary    BOOLEAN DEFAULT false,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (student_id, guardian_id)
);
```

Every parent/guardian query must authorize through `student_guardians`.

## 4.10 Subjects

```sql
CREATE TABLE subjects (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id     UUID REFERENCES schools(id),
  name          TEXT NOT NULL,
  code          TEXT,
  class_id      UUID REFERENCES classes(id),
  display_order INTEGER DEFAULT 0,
  created_at    TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE subject_teacher_assignments (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subject_id       UUID REFERENCES subjects(id) ON DELETE CASCADE,
  teacher_id       UUID REFERENCES staff(id) ON DELETE CASCADE,
  academic_year_id UUID REFERENCES academic_years(id),
  term_id          UUID REFERENCES terms(id),
  UNIQUE (subject_id, teacher_id, term_id)
);
```

## 4.11 Subject Group Configuration

**Never hardcode subject arrays in business logic.** Use configured group membership.

```sql
CREATE TABLE subject_groups (
  id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID NOT NULL REFERENCES schools(id),
  group_key TEXT NOT NULL,   -- "VERBAL", "NUMERICAL", "PRACTICAL", "THEORY"
  name      TEXT NOT NULL
);

CREATE TABLE subject_group_members (
  subject_group_id UUID NOT NULL REFERENCES subject_groups(id),
  subject_id       UUID NOT NULL REFERENCES subjects(id),
  PRIMARY KEY (subject_group_id, subject_id)
);
```

## 4.12 Aggregate Rules

**Never use `subjects.is_core` alone for aggregate logic.**

```sql
CREATE TABLE aggregate_rules (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id        UUID NOT NULL REFERENCES schools(id) ON DELETE CASCADE,
  name             TEXT NOT NULL,          -- "Four Core + Two Elective"
  division         TEXT,
  selection_method TEXT NOT NULL,          -- "four_cores_two_electives", "best_six"
  is_active        BOOLEAN DEFAULT true,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE aggregate_rule_subjects (
  aggregate_rule_id UUID NOT NULL REFERENCES aggregate_rules(id) ON DELETE CASCADE,
  subject_id        UUID NOT NULL REFERENCES subjects(id) ON DELETE CASCADE,
  category          TEXT,                  -- "core", "elective"
  rank              INTEGER,
  PRIMARY KEY (aggregate_rule_id, subject_id)
);
```

Expose `computeAggregate(studentId, termId, ruleId)` as the deterministic aggregate function.

## 4.13 Score Sheet Configuration

```sql
CREATE TABLE scoresheet_config (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id        UUID REFERENCES schools(id),
  class_id         UUID REFERENCES classes(id),  -- NULL = all classes
  term_id          UUID REFERENCES terms(id),
  column_count     INTEGER DEFAULT 2,
  columns          JSONB,
  -- e.g. [{"name":"Class Score","key":"class_score","max":40},
  --        {"name":"Exam Score","key":"exam_score","max":60}]
  class_score_max  NUMERIC DEFAULT 40,
  exam_score_max   NUMERIC DEFAULT 60,
  updated_at       TIMESTAMPTZ DEFAULT now()
);
```

## 4.14 Scores

```sql
CREATE TABLE scores (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  student_id     UUID REFERENCES students(id) ON DELETE CASCADE,
  subject_id     UUID REFERENCES subjects(id) ON DELETE CASCADE,
  term_id        UUID REFERENCES terms(id) ON DELETE CASCADE,
  entered_by     UUID REFERENCES staff(id),
  class_score    NUMERIC(5,2),
  exam_score     NUMERIC(5,2),
  total_score    NUMERIC(5,2),  -- computed by server when both components are present
  grade          TEXT,
  position       INTEGER,
  subject_remark TEXT,
  is_locked      BOOLEAN DEFAULT false,
  created_at     TIMESTAMPTZ DEFAULT now(),
  created_by     UUID,
  updated_at     TIMESTAMPTZ DEFAULT now(),
  updated_by     UUID,
  UNIQUE (student_id, subject_id, term_id)
);
```

**Score completeness rule:**
- Both components present → calculate total
- One or both missing → `total_score` remains NULL (incomplete)
- Never silently treat missing exam score as zero unless the school explicitly configures that policy

## 4.15 Scoresheet Status

```sql
CREATE TABLE scoresheet_submissions (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subject_id   UUID REFERENCES subjects(id),
  class_id     UUID REFERENCES classes(id),
  term_id      UUID REFERENCES terms(id),
  teacher_id   UUID REFERENCES staff(id),
  status       TEXT NOT NULL DEFAULT 'DRAFT'
               CHECK (status IN ('DRAFT','IN_PROGRESS','SUBMITTED','REOPENED','LOCKED')),
  submitted_at TIMESTAMPTZ,
  locked_at    TIMESTAMPTZ,
  locked_by    UUID,
  created_at   TIMESTAMPTZ DEFAULT now(),
  UNIQUE (subject_id, class_id, term_id)
);
```

## 4.16 Grading Schema

```sql
CREATE TABLE grading_schemas (
  id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id UUID REFERENCES schools(id),
  name      TEXT DEFAULT 'GES Standard',
  is_active BOOLEAN DEFAULT true,
  bands     JSONB
  -- e.g. [{"min":80,"max":100,"grade":"A1","remark":"Excellent"},
  --        {"min":70,"max":79,"grade":"B2","remark":"Very Good"}, ...]
);
```

Validate active grading schemas before activation: full 0–100 coverage, no gaps, no overlaps, min ≤ max. If a score matches no band, return a structured validation error — never silently return null.

## 4.17 Student Reports and Versioning

```sql
CREATE TABLE student_reports (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  student_id            UUID REFERENCES students(id) ON DELETE CASCADE,
  term_id               UUID REFERENCES terms(id) ON DELETE CASCADE,
  class_id              UUID REFERENCES classes(id),
  total_score           NUMERIC(6,2),
  aggregate             NUMERIC(5,2),
  overall_position      INTEGER,
  out_of                INTEGER,
  headteacher_remark_id UUID,   -- FK to remark_generation_log
  class_teacher_remark_id UUID,
  next_term_begins      DATE,
  attendance_days_present INTEGER,
  total_school_days     INTEGER,
  status                TEXT NOT NULL DEFAULT 'DRAFT'
                        CHECK (status IN ('DRAFT','READY_FOR_REVIEW','PUBLISHED',
                                          'REOPENED','SUPERSEDED')),
  effective_aggregate_rule_id UUID,   -- snapshot of rule used
  generated_at          TIMESTAMPTZ,
  published_at          TIMESTAMPTZ,
  UNIQUE (student_id, term_id)
);

CREATE TABLE report_versions (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  report_id         UUID NOT NULL REFERENCES student_reports(id) ON DELETE CASCADE,
  version_number    INTEGER NOT NULL,
  score_snapshot    JSONB NOT NULL,
  remark_snapshot   JSONB NOT NULL,
  settings_snapshot JSONB NOT NULL,
  pdf_url           TEXT,
  generated_by      UUID,
  generated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (report_id, version_number)
);
```

The snapshot captures what was actually printed/published. Later database edits cannot silently rewrite history.

## 4.18 Assignments

```sql
CREATE TABLE assignments (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subject_id      UUID REFERENCES subjects(id),
  teacher_id      UUID REFERENCES staff(id),
  term_id         UUID REFERENCES terms(id),
  title           TEXT NOT NULL,           -- was "Enter Assignment Number" — FIXED
  assignment_type TEXT,                    -- "Class Work", "Homework", "Project", "Quiz", "Test"
  sequence_number INTEGER,
  max_score       NUMERIC(5,2),
  assigned_date   DATE DEFAULT CURRENT_DATE,
  due_date        DATE,
  description     TEXT,
  created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE assignment_scores (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  assignment_id UUID REFERENCES assignments(id) ON DELETE CASCADE,
  student_id    UUID REFERENCES students(id) ON DELETE CASCADE,
  score         NUMERIC(5,2),
  submitted_at  TIMESTAMPTZ,
  UNIQUE (assignment_id, student_id)
);
```

**Assignment scores do NOT automatically modify the official class/exam score** unless a specific school grading configuration explicitly says so.

## 4.19 Attendance

```sql
CREATE TABLE attendance (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  student_id  UUID REFERENCES students(id) ON DELETE CASCADE,
  class_id    UUID REFERENCES classes(id),
  term_id     UUID REFERENCES terms(id),
  date        DATE NOT NULL,
  status      TEXT CHECK (status IN ('Present', 'Absent', 'Late', 'Excused')),
  recorded_by UUID REFERENCES staff(id),
  created_at  TIMESTAMPTZ DEFAULT now(),
  UNIQUE (student_id, date)
);
```

**Missing attendance MUST NOT be classified as low attendance.** Never generate attendance-based remark patterns when attendance data for the period is insufficiently complete.

## 4.20 Finance (Assessments + Payments)

**Never use a simple `amount_paid` field.** Use separate assessment and payment transaction records.

```sql
CREATE TABLE fee_types (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id    UUID REFERENCES schools(id),
  name         TEXT NOT NULL,              -- "School Fees", "PTA Levy"
  term_id      UUID REFERENCES terms(id),
  class_id     UUID REFERENCES classes(id),  -- NULL = all classes
  is_mandatory BOOLEAN DEFAULT true
);

CREATE TABLE fee_assessments (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  student_id      UUID NOT NULL REFERENCES students(id),
  fee_type_id     UUID NOT NULL REFERENCES fee_types(id),
  amount_assessed NUMERIC(10,2) NOT NULL,
  due_date        DATE,
  created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE payments (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  fee_assessment_id  UUID NOT NULL REFERENCES fee_assessments(id),
  amount             NUMERIC(10,2) NOT NULL,
  payment_date       DATE NOT NULL DEFAULT CURRENT_DATE,
  payment_method     TEXT NOT NULL,         -- "Cash", "Mobile Money", "Bank Transfer"
  receipt_number     TEXT NOT NULL,
  UNIQUE (receipt_number, /* school_id via assessment chain */),
  recorded_by        UUID REFERENCES staff(id),
  status             TEXT NOT NULL DEFAULT 'posted'
                     CHECK (status IN ('posted', 'void', 'refund', 'correction')),
  created_at         TIMESTAMPTZ DEFAULT now()
);
```

Balance = assessment minus posted payments. Do not silently overwrite old payments. Finalized financial records require VOID/REFUND/CORRECTION states for corrections.

## 4.21 Report Settings

```sql
CREATE TABLE report_settings (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id             UUID REFERENCES schools(id),
  class_id              UUID REFERENCES classes(id),   -- NULL = school-wide default
  term_id               UUID REFERENCES terms(id),     -- NULL = all terms
  use_new_ges_format    BOOLEAN DEFAULT false,
  show_aggregate        BOOLEAN DEFAULT true,
  show_raw_score        BOOLEAN DEFAULT true,
  show_overall_position BOOLEAN DEFAULT true,
  show_subject_code     BOOLEAN DEFAULT false,
  show_raw_class_score  BOOLEAN DEFAULT true,
  show_raw_exam_score   BOOLEAN DEFAULT true,
  show_grades_column    BOOLEAN DEFAULT true,
  show_subject_remarks  BOOLEAN DEFAULT true,
  show_subject_position BOOLEAN DEFAULT false,
  show_signature        BOOLEAN DEFAULT true,
  show_grading_schema   BOOLEAN DEFAULT true,
  aggregate_rule_id     UUID REFERENCES aggregate_rules(id),
  auto_headteacher_remarks BOOLEAN DEFAULT true,
  auto_class_teacher_remarks BOOLEAN DEFAULT true,
  remarks_tone          TEXT DEFAULT 'professional_warm',
  max_remark_chars      INTEGER DEFAULT 200,
  updated_at            TIMESTAMPTZ DEFAULT now()
);
```

**Settings precedence (use a central resolver — never let each screen implement its own):**
```
system defaults → school defaults → class override → term-specific override
```

```ts
resolveReportSettings({ schoolId, classId, termId })
```

The UI must show whether a value is inherited or overridden, e.g.:
```
Aggregate method: Four Core + Two Electives
Source: School default  [Override for this class]
```

## 4.22 Remark Templates

```sql
CREATE TABLE remark_templates (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pattern_key      TEXT NOT NULL,
  clause_slot      TEXT NOT NULL CHECK (clause_slot IN ('primary', 'secondary')),
  remark_type      TEXT NOT NULL CHECK (remark_type IN ('headteacher', 'class_teacher')),
  gender           TEXT NOT NULL DEFAULT 'Any'
                   CHECK (gender IN ('Male', 'Female', 'Other', 'Any')),
  tone             TEXT NOT NULL DEFAULT 'professional_warm',
  division         TEXT,
  variant_index    INTEGER NOT NULL,
  variant_text     TEXT NOT NULL,
  max_chars        INTEGER DEFAULT 200,
  is_active        BOOLEAN DEFAULT true,
  template_version INTEGER NOT NULL DEFAULT 1,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (
    pattern_key, clause_slot, remark_type,
    gender, tone, division, variant_index, template_version
  )
);
```

Never silently mutate historical template language. When wording changes, create a new version — old version remains for historical reports.

## 4.23 Remark Generation Log

```sql
CREATE TABLE remark_generation_log (
  id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  student_report_id       UUID NOT NULL REFERENCES student_reports(id),
  remark_type             TEXT NOT NULL,
  primary_pattern_key     TEXT NOT NULL,
  secondary_pattern_key   TEXT,
  primary_template_id     UUID REFERENCES remark_templates(id),
  secondary_template_id   UUID REFERENCES remark_templates(id),
  primary_variant_index   INTEGER,
  secondary_variant_index INTEGER,
  template_version_snapshot INTEGER,
  generated_text          TEXT NOT NULL,
  final_text              TEXT,
  status                  TEXT NOT NULL DEFAULT 'GENERATED'
                          CHECK (status IN ('NOT_STARTED','GENERATED','EDITED',
                                            'APPROVED','MANUAL_REVIEW_REQUIRED')),
  source                  TEXT NOT NULL DEFAULT 'SYSTEM_GENERATED'
                          CHECK (source IN ('SYSTEM_GENERATED','ADMIN_EDITED','MANUAL')),
  was_edited              BOOLEAN NOT NULL DEFAULT false,
  generated_by            UUID,
  edited_by               UUID,
  edited_at               TIMESTAMPTZ,
  created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Do NOT store:** `model_used`, `prompt_sent`, AI audit result — the production remarks engine no longer uses them.

## 4.24 Remark Pattern Thresholds (Configurable)

```sql
CREATE TABLE remark_thresholds (
  id                        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id                 UUID NOT NULL REFERENCES schools(id),
  verbal_numerical_diff     NUMERIC DEFAULT 15,
  practical_theory_diff     NUMERIC DEFAULT 15,
  classwork_exam_diff       NUMERIC DEFAULT 15,
  consistent_spread         NUMERIC DEFAULT 10,
  inconsistent_spread       NUMERIC DEFAULT 30,
  low_attendance_pct        NUMERIC DEFAULT 80,
  excellent_attendance_pct  NUMERIC DEFAULT 95,
  trend_position_change     INTEGER DEFAULT 3,
  max_remark_chars          INTEGER DEFAULT 200
);
```

## 4.25 Audit Log

```sql
CREATE TABLE audit_log (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id   UUID REFERENCES schools(id),
  user_id     UUID,
  user_role   TEXT,
  action      TEXT,
  -- STUDENT_CREATED, STUDENT_UPDATED, STUDENT_ARCHIVED,
  -- STAFF_CREATED, STAFF_UPDATED, PASSWORD_REVOKED, PASSWORD_RESET,
  -- SCORESHEET_SUBMITTED, SCORESHEET_REOPENED,
  -- REPORT_GENERATED, REPORT_PUBLISHED, REPORT_REOPENED,
  -- REMARK_GENERATED, REMARK_EDITED, REMARK_APPROVED,
  -- PAYMENT_RECORDED, PAYMENT_VOIDED, PROMOTION_EXECUTED, SETTINGS_CHANGED
  table_name  TEXT,
  record_id   UUID,
  old_values  JSONB,
  new_values  JSONB,
  ip_address  TEXT,
  created_at  TIMESTAMPTZ DEFAULT now()
);
```

Never write passwords, tokens, auth secrets, or unnecessary sensitive PII to this log.

## 4.26 Notifications

```sql
CREATE TABLE notifications (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  school_id      UUID REFERENCES schools(id),
  recipient_type TEXT,                    -- "admin", "teacher", "parent", "student"
  recipient_id   UUID,
  title          TEXT,
  body           TEXT,
  channel        TEXT,                    -- "in_app", "sms", "email"
  sent_at        TIMESTAMPTZ,
  is_read        BOOLEAN DEFAULT false,
  reference_type TEXT,
  reference_id   UUID
);
```

Notification failure must NEVER roll back a core academic transaction.

## 4.27 Required Indexes

```sql
CREATE INDEX ON scores (school_id) WHERE school_id IS NOT NULL;
CREATE INDEX ON scores (term_id);
CREATE INDEX ON scores (class_id);
CREATE INDEX ON scores (student_id);
CREATE INDEX ON scores (subject_id);
CREATE INDEX ON student_enrollments (academic_year_id);
CREATE INDEX ON student_enrollments (class_id);
CREATE INDEX ON payments (fee_assessment_id);
CREATE INDEX ON remark_generation_log (student_report_id);
CREATE INDEX ON audit_log (school_id, created_at DESC);
```

---

# PART 5 — USER ROLES

| Role | Permissions |
|------|-------------|
| SUPER_ADMIN | Everything + multi-school management |
| SCHOOL_ADMIN | Full school: registration, reports, finance, settings, staff |
| HEAD_TEACHER | Academic and report oversight + headteacher remarks; no unnecessary financial/system powers |
| TEACHER | Assigned subjects and classes only; own assignments; class attendance as authorized |
| ACCOUNTANT/BURSAR | Finance operations |
| STUDENT | Own academic records and permitted portal data only |
| PARENT | One or more linked children only |

**Keep role authorization separate from UI navigation.** Hidden buttons are not security.

---

# PART 6 — SUPABASE RLS POLICIES

```sql
-- Teachers can only access scores for their assigned subjects
CREATE POLICY "teachers_own_subject_scores_select" ON scores
FOR SELECT USING (
  subject_id IN (
    SELECT subject_id FROM subject_teacher_assignments
    WHERE teacher_id = (SELECT id FROM staff WHERE user_id = auth.uid())
  )
  AND NOT is_locked
);

CREATE POLICY "teachers_own_subject_scores_update" ON scores
FOR UPDATE USING (
  subject_id IN (
    SELECT subject_id FROM subject_teacher_assignments
    WHERE teacher_id = (SELECT id FROM staff WHERE user_id = auth.uid())
  )
  AND NOT is_locked
);

-- Teachers cannot delete scores
-- (No DELETE policy created for teachers)

-- Students see only their own scores
CREATE POLICY "students_own_scores" ON scores
FOR SELECT USING (
  student_id = (SELECT id FROM students WHERE user_id = auth.uid())
);

-- Parents see only linked children
CREATE POLICY "parents_linked_student_reports" ON student_reports
FOR SELECT USING (
  student_id IN (
    SELECT sg.student_id FROM student_guardians sg
    JOIN guardians g ON sg.guardian_id = g.id
    WHERE g.user_id = auth.uid()
  )
  AND status = 'PUBLISHED'
);

-- Finance data not exposed to teachers/students
CREATE POLICY "admin_only_payments" ON payments
FOR ALL USING (
  auth.uid() IN (
    SELECT user_id FROM staff
    WHERE school_id = (SELECT school_id FROM fee_assessments WHERE id = fee_assessment_id)
    AND staff_type IN ('Admin', 'Head Teacher', 'Accountant')
  )
);
```

---

# PART 7 — ADMIN PANEL

## 7.1 Navigation Structure

```
ADMIN PANEL (mobile: bottom tab bar; desktop: sidebar)

├── 🏠  Dashboard
├── 📋  Registration
│   ├── New Student
│   ├── New Staff
│   └── Registration Status
├── 🎓  Academic
│   ├── Score Sheets
│   ├── Master Score Sheet
│   ├── Reports
│   │   ├── Report Readiness
│   │   ├── Generate Reports
│   │   └── Individual Reports
│   ├── Remarks
│   │   ├── Headteacher Remarks
│   │   └── Class Teacher Remarks
│   └── Order of Merit
├── 🏫  Manage
│   ├── Classes
│   ├── Subjects
│   ├── Subject–Teacher Assignment
│   ├── Students
│   ├── Staff
│   ├── Contacts (Guardians)
│   └── Archives
├── 📅  Calendar
│   └── Term Dates (by Division)
├── 💰  Finance
│   ├── Fee Types
│   ├── Fee Assessments
│   ├── Record Payment
│   ├── Arrears Report
│   └── PTA Dues
├── ⚙️   Settings
│   ├── Report Settings
│   ├── Score Sheet Settings
│   ├── Grading Schema
│   ├── Remark Templates
│   ├── Subject Groups
│   └── School Profile
├── 🔐  Access
│   ├── Staff Passwords
│   └── Student Passwords
└── 👤  My Account
    ├── Profile
    ├── Change Password
    └── Sign Out
```

## 7.2 Admin Dashboard

The home screen is a real action dashboard — NOT just a search engine.

```
┌──────────────────────────────────────────────────────┐
│  Morning Glory Academy · Term 1, 2026/2027           │
│  Logged in as: [Admin Name]                          │
├──────────┬──────────┬──────────┬────────────────────┤
│ Students │  Staff   │ Classes  │ Scoresheets Ready  │
│   342    │   28     │   13     │    9 / 13 ✓        │
├──────────┴──────────┴──────────┴────────────────────┤
│ ⚠️ ACTION REQUIRED                                   │
│  · 4 classes have incomplete scoresheets            │
│  · 12 students have outstanding fee balance         │
│  · 3 remarks need review                           │
│  · Term ends in 14 days                            │
├──────────────────────────────────────────────────────┤
│ QUICK ACTIONS                                        │
│  [Register Student] [Enter Scores] [Generate Reports]│
│  [Record Payment]   [Review Remarks]                 │
├──────────────────────────────────────────────────────┤
│ RECENT ACTIVITY (last 10 actions with timestamps)    │
└──────────────────────────────────────────────────────┘
```

## 7.3 Student Registration

**All fields visible — nothing hidden behind a vague MORE button.**

**Section 1 — Identity**
- Surname (required)
- First Name (required)
- Other Names / Middle Name
- Date of Birth (native `<input type="date">` — never a dropdown)
- Gender (radio: Male / Female / Other)
- Photo (drag & drop or camera capture)

**Section 2 — Academic Placement**
- Class (required dropdown — must select, cannot save with "None")
- Enrollment Date (auto-filled today, editable)

**Section 3 — Guardians**
- Inline guardian search/link (reuses existing guardian records)
- Add new guardian: Full Name, Phone, Relationship

**Validation rules:**
- Surname + First Name + Class are required before save
- Duplicate detection: warn if Surname + First Name + DOB match an existing student
- Student ID is server-generated via `allocate_next_id()` — never generated client-side

**Success state:**
```
Student registered successfully
Student ID: MOGASCO01/03/26/0343
[View Record]  [Register Another]
```

## 7.4 Staff Registration

**Fields:** Surname, First Name, Other Names, Gender (radio), Staff Type, Email (optional), Contact One, Contact Two, Date Joined, Photo

**On save:** Generate secure temporary password → mark `must_change_password = true` → show one-time credential to admin in a modal → optionally send to staff email.

**Terminology:** Use "Save Staff Member" not "SAVE INFO."

## 7.5 Admin Scoresheet View

Admin sees all subjects for a class in a grid (master view):

- Color coding: red (<40%), amber (40–59%), green (60%+)
- Sticky student name column + horizontal scroll with visible indicator
- Click cell to edit (only when term is open and record is not locked)
- Export to Excel / Print to PDF actions visible
- Lock Term requires confirmation + audit log entry

## 7.6 Headteacher Remarks — Full-Screen Editor

**Old: narrow table cell — unusable on mobile. New: two-step flow.**

**Step 1 — List View:**
```
HEADTEACHER REMARKS · Class 3 · Term 1

✓  Abass Sheikh       Approved
⚡ Agordor Blessing   Generated (needs review)
⚠  Sarah Mensah       Manual Review Required
✗  John Doe           Not Started
```

**Step 2 — Per-Student Editor (full-screen on mobile):**
```
Agordor Blessing

Generated Remark (System)
─────────────────────────────────────────────
[remark text — up to 200 chars]

Primary pattern:   HIGH_ACHIEVER
Secondary pattern: IMPROVING
Variant: 3 of 7

Other available variants:
[Variant 1] [Variant 2] [Variant 4] [Variant 5]

Character count: 142 / 200

[Edit Manually]  [Cycle to Next Variant]  [Save]  [Approve]
```

Sticky bottom action bar on mobile so Save is always accessible. Character counter visible at all times.

## 7.7 Report Settings UI

**Grouped sections — no more flat list of 14 unlabeled checkboxes:**

**Score Display:** Show Raw Class Score / Show Raw Exam Score / Show Total Score / Show Grades Column

**Report Structure:** Show Subject Code / Show Subject Remarks / Show Subject Position / Show Overall Position / Show Aggregate Score

**Computation:** Aggregate method (dropdown from active aggregate rules) / Use New GES Report Format

**Automatic Remarks:** ☐ Headteacher remarks / ☐ Class teacher remarks / Tone selector / Max length

**Signature & Footer:** Show Headteacher Signature / Show Grading Schema

**Apply to:** dropdown (All Classes / Specific Class)

**Scope callout always visible:** "These settings apply globally unless overridden per class."

## 7.8 Report Readiness Gate

**Do not generate final reports while required prerequisites are unresolved.**

```
REPORT READINESS · Class 3 · Term 1

✓ Student records complete
✓ Subject assignments complete
✓ Required scores present
✓ Scoresheets submitted
✓ Grades calculated
✓ Positions calculated
✓ Aggregate rule configured and valid
✓ Attendance sufficiently complete
⚠ 3 remarks not yet approved

[Review Remarks]
[Generate Deterministic Remarks]
[Preview Reports]
[Generate Reports — Blocked until all issues resolved]
```

A report generation attempt returns a structured list of blocking issues, not a generic failure.

## 7.9 Report Workflow

```
Academic → Reports → Select Class / Term (pre-filled from context)
  → REPORT READINESS (review blockers)
  → REMARKS (generate / review / approve)
  → PREVIEW (preview report card)
  → GENERATE (generate PDFs + version snapshots)
  → PUBLISH (finalize → lock → optionally notify parents)
```

**Report lifecycle:**
```
DRAFT → READY_FOR_REVIEW → PUBLISHED → REOPENED → SUPERSEDED
```

Parents/students only ever see `PUBLISHED` reports.

## 7.10 Score Entry Workflow and Position Recomputation

```
Teacher edits scores
    ↓
Score saved (no expensive global rerank on every keystroke)
    ↓
Teacher submits scoresheet
    ↓
Transactional position recomputation using RANK() OVER (
  PARTITION BY class_id, subject_id, term_id
  ORDER BY total_score DESC
)
    ↓
Subject positions finalized
    ↓
Overall positions computed at report readiness / finalization
```

Ties must be handled consistently. Define whether positions use RANK, DENSE_RANK, or another scheme and document it.

---

# PART 8 — TEACHER PANEL

## 8.1 Navigation Structure

```
TEACHER PANEL (mobile: bottom tab bar; desktop: sidebar)

├── 🏠  Home (Task Dashboard)
├── 📊  Scoresheets (with completion status)
├── 📝  Assignments
│   ├── Add Assignment
│   ├── View Assignments
│   └── Record Assignment Scores
├── 👥  My Class
├── 📅  Timetable
└── 👤  My Account
    ├── Profile
    ├── Change Password
    └── Sign Out
```

## 8.2 Teacher Dashboard

```
┌──────────────────────────────────────────────────────┐
│  Good morning, [Teacher Name] 👋                     │
│  Term 1 · 2026/2027                                  │
├──────────────────────────────────────────────────────┤
│ YOUR PENDING TASKS                                   │
│  🔴 Numeracy — Not submitted                         │
│  🟡 Literacy — 12/28 entered                         │
│  ✅ Science — Submitted                             │
│  ✅ History — Submitted                             │
├──────────────────────────────────────────────────────┤
│ MY CLASS: Class 3 · 28 students                     │
│  Attendance today: Not marked                       │
│  [Mark Attendance Now]                              │
└──────────────────────────────────────────────────────┘
```

This screen provides real task context — not just identity information.

## 8.3 Teacher Scoresheet Entry

Vertical mobile-friendly student list (not a horizontal table):

```
Class 3 · NUMERACY · Term 1

Progress: 12 / 28 students entered            [Submit Scoresheet]

1. ABASS SHEIKH
   Class Score: [    ]  / 40
   Exam Score:  [    ]  / 60
   Total: —    Grade: —

2. AGORDOR BLESSING
   Class Score: [ 36 ]  / 40  ✓
   Exam Score:  [ 43 ]  / 60  ✓
   Total: 79    Grade: B2

[Load More ↓]
```

- Auto-save on blur (never pretend it succeeded if the request failed — show error)
- Validation: class score cannot exceed configured maximum; exam score cannot exceed configured maximum; no negatives
- Grade computed immediately client-side from grading schema; confirmed server-side on save
- "Submit Scoresheet" transitions status to SUBMITTED → notifies admin
- Explicit locked state shown when scoresheet is LOCKED

## 8.4 Assignment Form

**Fixed — "Enter Assignment Number" was the original broken label:**

- Class (required)
- Subject (required)
- Assignment Title (required) — replaces ambiguous "Enter Assignment Number"
- Assignment Type (Class Work / Homework / Project / Quiz / Test)
- Sequence Number (auto-incremented, shown as "Assignment #3 for this term")
- Maximum Score
- Due Date (`<input type="date">`)
- Description (optional textarea)

---

# PART 9 — STUDENT & PARENT PORTAL

## 9.1 Student Portal (`/student`)

Login with Student ID code + password (admin-set).

- View own published report cards (all terms/years)
- Download report card PDF
- View subject scores per term
- View assignment scores
- View attendance summary
- View outstanding fees (read-only)
- No draft report data visible

## 9.2 Parent Portal (`/parent`)

Login linked to one or more children.

- Child switcher if multiple children linked
- View linked child's published report cards
- Download PDF
- View fee balance + payment history
- Receive notifications when report is ready or fee reminder
- Only sees `student_guardians`-authorized children — never another student

---

# PART 10 — DETERMINISTIC REMARKS ENGINE

> **The production remarks system does not call any external AI.** It uses deterministic pattern classification + stored template banks + placeholder substitution.

## 10.1 Data Quality Short-Circuit

```ts
function validateStudentProfile(profile: StudentProfile):
  'COMPLETE' |
  'INCOMPLETE_SCORES' |
  'INVALID_SCORE_DATA' |
  'MISSING_REQUIRED_ATTENDANCE' |
  'MISSING_CURRENT_POSITION' |
  'NO_PREVIOUS_TERM'
```

Never generate a remark from an invalid/incomplete profile. Return `MANUAL_REVIEW_REQUIRED`.

## 10.2 Primary Performance Tiers (Exactly One)

```ts
function getPrimaryTier(avg: number): PrimaryTier {
  if (avg < 40) return 'STRUGGLING';
  if (avg < 50) return 'BELOW_AVERAGE';
  if (avg < 65) return 'AVERAGE';
  if (avg < 80) return 'ABOVE_AVERAGE';
  return 'HIGH_ACHIEVER';
}
```

A student has **exactly one** primary tier. Never two simultaneously.

## 10.3 Secondary Pattern Priority (At Most One)

Pick the highest-priority applicable secondary pattern. Never force one if none is appropriate.

```
Priority:
 1. DECLINING
 2. LOW_ATTENDANCE
 3. IMPROVING
 4. VERBAL_STRONG_NUMERICAL_WEAK
 5. NUMERICAL_STRONG_VERBAL_WEAK
 6. PRACTICAL_STRONG_THEORY_WEAK
 7. THEORY_STRONG_PRACTICAL_WEAK
 8. STRONG_CLASSWORK_WEAK_EXAMS
 9. WEAK_CLASSWORK_STRONG_EXAMS
10. INCONSISTENT
11. EXCELLENT_ATTENDANCE
12. CONSISTENT
```

**Incompatible pairs that must never coexist:**
- IMPROVING + DECLINING
- CONSISTENT + INCONSISTENT
- LOW_ATTENDANCE + EXCELLENT_ATTENDANCE

## 10.4 Pattern Detection Rules

All thresholds come from `remark_thresholds` table, never hardcoded.

```ts
// Consistency
spread <= thresholds.consistent_spread    → CONSISTENT
spread >= thresholds.inconsistent_spread  → INCONSISTENT

// Verbal vs Numerical (use normalized averages; require sufficient subjects in both groups)
verbalAvg - numericalAvg >= thresholds.verbal_numerical_diff → VERBAL_STRONG_NUMERICAL_WEAK
numericalAvg - verbalAvg >= thresholds.verbal_numerical_diff → NUMERICAL_STRONG_VERBAL_WEAK

// Classwork vs Exam (compare normalized percentages, not raw values)
classPct = classScore / classMax * 100
examPct  = examScore  / examMax  * 100
classPct - examPct >= thresholds.classwork_exam_diff → STRONG_CLASSWORK_WEAK_EXAMS
examPct - classPct >= thresholds.classwork_exam_diff → WEAK_CLASSWORK_STRONG_EXAMS

// Trend (requires both valid positions)
function computeTrend(currentPos: number, previousPos: number) {
  const delta = previousPos - currentPos;
  if (delta >= thresholds.trend_position_change) return 'IMPROVING';
  if (delta <= -thresholds.trend_position_change) return 'DECLINING';
  return 'STABLE';
}

// Attendance
attendance < thresholds.low_attendance_pct   → LOW_ATTENDANCE
attendance >= thresholds.excellent_attendance_pct → EXCELLENT_ATTENDANCE
// Missing attendance data → no attendance pattern
```

Subject group membership comes from `subject_group_members` table. Never from arrays of subject names hardcoded in TypeScript.

## 10.5 Template Selection

Selection is deterministic — same student + same pattern + same config = same variant every time.

```ts
// Stable key for initial selection
const selectionKey = hash(`${studentId}:${patternKey}:${slot}:${remarkType}`);
const variantIndex = selectionKey % totalVariants;
```

**Regenerate action** does NOT call any API. It advances through stored variants:
```
Variant 2 → [Regenerate] → Variant 3 → [Regenerate] → Variant 4 ...
```

Store `primary_variant_index` and `secondary_variant_index` in `remark_generation_log` so reprints return the saved choice.

**Template fallback order (log every fallback):**
```
exact pattern + type + gender + tone + division
      ↓
Any gender
      ↓
Any division
      ↓
default tone
      ↓
system-safe default template
      ↓ (if nothing found)
MANUAL_REVIEW_REQUIRED — never return blank silently
```

## 10.6 Placeholders

Standard placeholders:

```
{{name}}         — student's first name or full name
{{they}}         — pronoun (he/she/they)
{{them}}         — pronoun (him/her/them)
{{their}}        — pronoun (his/her/their)
{{prev_pos}}     — previous term position
{{cur_pos}}      — current position
{{strong_subject}} — highest-scoring subject name
{{focus_area}}   — lowest/weak subject/area name
```

Pronouns:

```ts
function getPronouns(gender: string) {
  if (gender === 'Male')   return { they: 'he',   them: 'him', their: 'his'   };
  if (gender === 'Female') return { they: 'she',  them: 'her', their: 'her'   };
  return                          { they: 'they', them: 'them', their: 'their' }; // Other/unknown
}
```

**Never assume non-Female means Male.**

Do not provide score/grade placeholders to the normal report-remark system.

## 10.7 Post-Render Validation

After substitution, validate:
- Rendered length ≤ `max_chars` (a long name can push a template over the limit)
- No unknown placeholders remain (e.g., `{{unknown}}`)
- No forbidden score/grade tokens
- Required placeholders resolved
- Valid Unicode text
- No duplicated whitespace
- Template is active
- Selected template matches pattern/type/slot

## 10.8 Remark Generation Function

```ts
function generateRemark(profile, remarkType, options) {
  const quality = validateStudentProfile(profile);
  if (!quality.ok) {
    return { status: 'MANUAL_REVIEW_REQUIRED', remark: null, reason: quality.reason };
  }

  const classification = classifyStudent(profile);
  const primaryKey    = classification.primaryTier;
  const secondaryKey  = classification.secondaryPatterns[0] ?? null;

  const primaryTemplate = pickVariant({
    patternKey: primaryKey, slot: 'primary', remarkType,
    gender: profile.gender, tone: profile.tone, division: profile.division,
    studentId: profile.id, variantIndex: options?.primaryVariantIndex
  });

  const secondaryTemplate = secondaryKey
    ? pickVariant({ patternKey: secondaryKey, slot: 'secondary', remarkType,
                    gender: profile.gender, tone: profile.tone, division: profile.division,
                    studentId: profile.id, variantIndex: options?.secondaryVariantIndex })
    : null;

  const rendered = renderTemplates(
    [primaryTemplate, secondaryTemplate].filter(Boolean), profile
  );

  validateRenderedRemark(rendered, profile);

  return {
    status: 'GENERATED',
    remark: rendered,
    primaryKey, secondaryKey,
    primaryTemplateId:   primaryTemplate?.id ?? null,
    secondaryTemplateId: secondaryTemplate?.id ?? null
  };
}
```

## 10.9 Batch Generation

```
load eligible students
→ validate profiles (mark MANUAL_REVIEW_REQUIRED for incomplete)
→ classify each student
→ select templates deterministically
→ render placeholders
→ validate rendered text
→ save to remark_generation_log
→ produce summary report:
  Generated: N
  Skipped (already approved): N
  Manual review required: N
  Failed validation: N

Do NOT overwrite an approved remark unless admin explicitly requests it.
```

No API concurrency limits apply since no network calls are made.

## 10.10 Sample Remark Template Bank

These are illustrative examples for the seed file:

**HIGH_ACHIEVER — primary / headteacher:**
```
"{{name}} has delivered an outstanding academic performance this term."
"{{name}} has shown a strong and sustained commitment to academic excellence."
"A very successful term for {{name}}, marked by impressive and consistent achievement."
```

**ABOVE_AVERAGE — primary:**
```
"{{name}} has performed very well this term and should continue building on this strong foundation."
"{{name}} has shown solid academic progress and good potential for even greater achievement."
```

**AVERAGE — primary:**
```
"{{name}} has made satisfactory progress this term and should continue working steadily."
"{{name}} has shown a sound level of progress, with room to build greater consistency next term."
```

**BELOW_AVERAGE — primary:**
```
"{{name}} has faced some academic challenges this term but can make steady progress with greater consistency and support."
```

**STRUGGLING — primary:**
```
"{{name}} has found some areas challenging this term. Continued support, confidence, and steady effort will help next term."
```

**IMPROVING — secondary:**
```
"The improvement in class standing shows determination and is worth building on next term."
"This upward progress reflects growing commitment; {{name}} should continue in the same direction."
```

**DECLINING — secondary:**
```
"A renewed focus next term should help {{name}} regain momentum and make the most of {{their}} ability."
```

**LOW_ATTENDANCE — secondary:**
```
"Improved attendance next term will help {{name}} benefit fully from classroom learning and support."
```

**Seed file must have 5–8 active variants per required combination.**

---

# PART 11 — API ROUTES

All routes enforce server-side session and school-scoped authorization. Never trust client-supplied IDs.

```
/api/auth/
  POST /login
  POST /logout

/api/students/
  GET    /                          list with filters
  POST   /                          create (calls allocate_next_id on server)
  GET    /[id]
  PATCH  /[id]
  DELETE /[id]                      soft delete → archive

/api/staff/
  GET / POST / GET [id] / PATCH [id]

/api/scores/
  GET  /class/[classId]/term/[termId]        full class scoresheet
  POST /                                      enter or update score
  POST /submit/[subjectId]/[classId]/[termId] submit + recompute positions
  GET  /student/[studentId]/term/[termId]     student's all scores

/api/remarks/                        (deterministic only — no AI calls)
  POST  /generate/[studentId]/[termId]
  POST  /generate-all/[classId]/[termId]
  PATCH /[reportId]
  POST  /[reportId]/approve

/api/reports/
  GET  /student/[studentId]/term/[termId]
  GET  /readiness/[classId]/[termId]
  POST /generate/class/[classId]/term/[termId]
  GET  /pdf/[reportId]
  GET  /order-of-merit/[classId]/[termId]

/api/attendance/
  GET  /class/[classId]/date/[date]
  POST /
  GET  /student/[studentId]/term/[termId]

/api/finance/
  GET  /fees/class/[classId]
  GET  /arrears/term/[termId]
  POST /payment

/api/notifications/
  GET  /
  PATCH /[id]/read

/api/jobs/
  GET  /[jobId]                     check bulk job status
```

---

# PART 12 — UI / UX DESIGN SYSTEM

## 12.1 Design Tokens

```css
:root {
  /* Brand */
  --color-primary:       #1B4F8A;
  --color-primary-light: #3A7BC8;
  --color-primary-dark:  #0F3560;

  /* Semantic */
  --color-success: #16A34A;
  --color-warning: #D97706;
  --color-danger:  #DC2626;
  --color-info:    #0284C7;

  /* Neutrals */
  --color-surface:    #FFFFFF;
  --color-background: #F8FAFC;
  --color-nav:        #1E293B;
  --color-border:     #E2E8F0;
  --color-text:       #1E293B;
  --color-text-muted: #64748B;
  --color-text-inv:   #FFFFFF;

  /* Score colours */
  --score-excellent: #16A34A;    /* 80+ */
  --score-good:      #0284C7;    /* 65–79 */
  --score-average:   #D97706;    /* 50–64 */
  --score-below:     #DC2626;    /* <50 */

  /* Typography */
  --font-sans:    'Inter', system-ui, sans-serif;
  --font-display: 'Lexend', 'Inter', sans-serif;

  /* Spacing (4px base) */
  --space-1: 4px;   --space-2: 8px;
  --space-3: 12px;  --space-4: 16px;
  --space-6: 24px;  --space-8: 32px;

  /* Radius */
  --radius-sm: 6px;  --radius-md: 10px;
  --radius-lg: 16px; --radius-pill: 9999px;

  /* Shadows */
  --shadow-card:  0 1px 3px rgba(0,0,0,0.08), 0 1px 2px rgba(0,0,0,0.04);
  --shadow-modal: 0 20px 60px rgba(0,0,0,0.15);
}
```

## 12.2 Button System (Unified — exactly four variants)

```tsx
// Primary: blue filled — main actions
<Button variant="primary">Generate Reports</Button>

// Secondary: outlined — secondary actions
<Button variant="secondary">Cancel</Button>

// Danger: red filled — destructive actions, ALWAYS with confirmation dialog
<Button variant="danger">Revoke Access</Button>

// Ghost: text only — tertiary actions
<Button variant="ghost">View Details →</Button>
```

All buttons: `min-height: 44px` (mobile touch target). No browser-default button styling anywhere in the app.

**REVOKE ACCESS uses `variant="danger"`. GENERATE PASSWORD uses `variant="primary"`.** Never make them visually identical.

## 12.3 Navigation (Mobile vs Desktop)

```tsx
// Mobile (≤768px): bottom tab bar
<nav className="fixed bottom-0 left-0 right-0 flex justify-around bg-nav py-2">
  <TabItem href="/admin"           icon={<Home />}    label="Home"     />
  <TabItem href="/admin/academic"  icon={<BookOpen />} label="Academic" />
  <TabItem href="/admin/manage"    icon={<Users />}   label="Manage"   />
  <TabItem href="/admin/finance"   icon={<Wallet />}  label="Finance"  />
  <TabItem href="/admin/more"      icon={<Menu />}    label="More"     />
</nav>

// Desktop (>768px): full sidebar with all items
```

Replace the hamburger-only pattern. The bottom tab bar is the standard for mobile.

## 12.4 Table Strategy

**Score-heavy tables (Master Score Sheet, Subject Management):**
```css
.scoresheet-table th:first-child,
.scoresheet-table td:first-child {
  position: sticky;
  left: 0;
  background: white;
  z-index: 1;
  border-right: 2px solid var(--color-border);
}
.scoresheet-wrapper {
  overflow-x: auto;
  -webkit-overflow-scrolling: touch;
  /* visible scroll affordance */
  background-image: linear-gradient(to right, white, white),
                    linear-gradient(to right, white, white),
                    linear-gradient(to right, rgba(0,0,0,0.08), transparent),
                    linear-gradient(to left, rgba(0,0,0,0.08), transparent);
  background-size: 30px 100%;
  background-attachment: local, local, scroll, scroll;
}
```

**Management tables (Students, Staff, Classes):**
```css
/* Mobile: each row becomes a card */
@media (max-width: 767px) {
  .data-table, .data-table tbody,
  .data-table tr, .data-table td { display: block; }
  .data-table thead { display: none; }
  .data-table td::before {
    content: attr(data-label);
    font-weight: 600;
    color: var(--color-text-muted);
    display: block;
    font-size: 11px;
    text-transform: uppercase;
    margin-bottom: 2px;
  }
  .data-table tr {
    border: 1px solid var(--color-border);
    border-radius: var(--radius-md);
    margin-bottom: var(--space-3);
    padding: var(--space-4);
    background: white;
  }
}
```

Minimum mobile body text: 14px. Minimum touch target: 44px.

## 12.5 Forms

- Label above field on every form, no exceptions
- Required field marked with * and validated
- Inline error below field in red
- All date inputs: `<input type="date">` — never `<select>` for dates
- All dropdowns: placeholder is "Select [field]…" not "none" or "None"
- On submission failure: keep entered values, explain exactly what must be fixed
- Zod validation on both client and server for every form

**CTA standardization (fix from the UX audit):**

| Screen | ❌ Old | ✅ New |
|--------|--------|--------|
| Student Report filter | PROCEED | Generate Report Cards |
| Headteacher Remarks | GO | Load Student List |
| Subject Scoresheet Admin | Submit | View Scoresheet |
| Subject Scoresheet Teacher | SUBMIT | Open Score Entry |
| Student Search | SEARCH | Find Student |
| Score Sheet Settings | Submit | Save Settings |
| Registration forms | MORE | Add More Details |
| Pupil Registration | SAVE | Save Student |
| Staff Registration | SAVE INFO. | Save Staff Member |
| School Calendar | Save | Save Calendar |

Use "Student" everywhere — never "Pupil". Use "Select Class…" not "none" / "None".

## 12.6 Confirmation Dialogs

Every destructive action requires a confirmation dialog:

```tsx
<ConfirmDialog
  title="Revoke Login Access"
  description="This will prevent [Name] from logging in immediately. You can restore access at any time from Staff Passwords."
  confirmLabel="Yes, Revoke Access"
  cancelLabel="Cancel"
  variant="danger"
  onConfirm={handleRevoke}
/>
```

## 12.7 Empty / Loading / Success / Error States

Every async workflow must handle all four states:

```
Loading scores…
Saving…
Saved successfully
Could not save score. Check your connection and try again. [Retry]

No students found in Class 3.  [Register a Student]
No payments recorded for this period.
No scoresheets submitted yet.
```

**Toast notifications:**
```ts
toast.success("Student registered · ID: MOGASCO01/03/26/0343");
toast.error("Cannot save: Class Score cannot exceed 40");
toast.info("Remarks generated for 28 students. Review before publishing.");
toast.warning("4 classes still have incomplete scoresheets.");
```

## 12.8 Accessibility Baseline

- Keyboard focus states on all interactive elements
- Descriptive `aria-label` on EDIT buttons (e.g., `aria-label="Edit Agordor Blessing"`)
- Adequate color contrast (≥ 4.5:1 for body text)
- Status communicated through text/icon as well as color:
  ```
  ✓ Submitted   ● In Progress   ⚠ Needs Review   🔒 Locked
  ```
- Touch targets ≥ 44px
- Form errors associated with their field (`aria-describedby`)
- No color-only status indicators

## 12.9 Design System Components

Build and reuse: Button, Input, Select, DateInput, Modal, ConfirmDialog, Toast, Badge, Skeleton, EmptyState, ErrorState, DataTable (desktop) / CardList (mobile), ProgressIndicator.

---

# PART 13 — FILE / FOLDER STRUCTURE

```
cerican/
├── app/
│   ├── (auth)/
│   │   ├── login/page.tsx
│   │   └── layout.tsx
│   ├── admin/
│   │   ├── layout.tsx
│   │   ├── page.tsx                        Dashboard
│   │   ├── registration/
│   │   │   ├── student/page.tsx
│   │   │   └── staff/page.tsx
│   │   ├── academic/
│   │   │   ├── scoresheets/page.tsx
│   │   │   ├── master-score/page.tsx
│   │   │   ├── reports/page.tsx
│   │   │   ├── remarks/page.tsx
│   │   │   └── order-of-merit/page.tsx
│   │   ├── manage/
│   │   │   ├── classes/page.tsx
│   │   │   ├── subjects/page.tsx
│   │   │   ├── students/page.tsx
│   │   │   ├── staff/page.tsx
│   │   │   └── contacts/page.tsx
│   │   ├── finance/
│   │   │   ├── fees/page.tsx
│   │   │   ├── payments/page.tsx
│   │   │   └── arrears/page.tsx
│   │   ├── attendance/page.tsx
│   │   └── settings/
│   │       ├── report/page.tsx
│   │       ├── scoresheet/page.tsx
│   │       ├── grading/page.tsx
│   │       ├── remark-templates/page.tsx
│   │       ├── subject-groups/page.tsx
│   │       └── school/page.tsx
│   ├── teacher/
│   │   ├── layout.tsx
│   │   ├── page.tsx                        Task Dashboard
│   │   ├── scoresheets/page.tsx
│   │   ├── assignments/
│   │   │   ├── add/page.tsx
│   │   │   ├── view/page.tsx
│   │   │   └── scores/page.tsx
│   │   ├── my-class/page.tsx
│   │   ├── attendance/page.tsx
│   │   ├── timetable/page.tsx
│   │   └── account/page.tsx
│   ├── student/
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── parent/
│   │   ├── layout.tsx
│   │   └── page.tsx
│   └── api/
│       ├── auth/route.ts
│       ├── students/route.ts
│       ├── staff/route.ts
│       ├── scores/route.ts
│       ├── remarks/route.ts               (deterministic only)
│       ├── reports/route.ts
│       ├── attendance/route.ts
│       ├── finance/route.ts
│       ├── notifications/route.ts
│       └── jobs/route.ts
│
├── components/
│   ├── ui/
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Select.tsx
│   │   ├── DateInput.tsx
│   │   ├── Modal.tsx
│   │   ├── ConfirmDialog.tsx
│   │   ├── Toast.tsx
│   │   ├── Badge.tsx
│   │   ├── Skeleton.tsx
│   │   ├── EmptyState.tsx
│   │   ├── ErrorState.tsx
│   │   ├── DataTable.tsx
│   │   ├── CardList.tsx
│   │   └── ProgressIndicator.tsx
│   ├── admin/
│   │   ├── Dashboard.tsx
│   │   ├── StudentForm.tsx
│   │   ├── StaffForm.tsx
│   │   ├── ScoreSheetGrid.tsx
│   │   ├── RemarksEditor.tsx
│   │   ├── ReportReadiness.tsx
│   │   ├── ReportSettings.tsx
│   │   └── FinanceTable.tsx
│   ├── teacher/
│   │   ├── TaskDashboard.tsx
│   │   ├── ScoreEntryList.tsx
│   │   └── AssignmentForm.tsx
│   ├── reports/
│   │   ├── ReportCard.tsx                 PDF template
│   │   └── OrderOfMerit.tsx
│   └── navigation/
│       ├── AdminNav.tsx
│       ├── TeacherNav.tsx
│       └── MobileBottomBar.tsx
│
├── lib/
│   ├── auth/
│   │   └── session.ts
│   ├── supabase/
│   │   ├── client.ts
│   │   ├── server.ts
│   │   └── middleware.ts
│   ├── scoring/
│   │   ├── grades.ts
│   │   ├── positions.ts
│   │   ├── aggregate.ts
│   │   ├── validation.ts
│   │   └── readiness.ts
│   ├── remarks/
│   │   ├── classify.ts                    Pattern classification (deterministic)
│   │   ├── generate.ts                    Main generation function
│   │   ├── templates.ts                   Template fetching + fallback
│   │   ├── variants.ts                    Variant selection + cycling
│   │   ├── placeholders.ts                Placeholder substitution + validation
│   │   ├── validation.ts                  Post-render validation
│   │   ├── batch.ts                       Batch generation
│   │   └── types.ts
│   ├── reports/
│   │   ├── settings.ts                    resolveReportSettings()
│   │   ├── generate.ts
│   │   ├── publish.ts
│   │   └── versioning.ts
│   ├── finance/
│   │   ├── assessments.ts
│   │   ├── payments.ts
│   │   ├── balances.ts
│   │   └── receipts.ts
│   ├── attendance/
│   │   └── summaries.ts
│   ├── notifications/
│   │   ├── sms.ts                         Twilio
│   │   └── email.ts                       Resend
│   ├── audit/
│   │   └── log.ts
│   ├── jobs/
│   │   └── runner.ts                      Idempotent bulk job runner
│   └── validations/
│       ├── student.schema.ts
│       ├── staff.schema.ts
│       ├── score.schema.ts
│       ├── assignment.schema.ts
│       ├── attendance.schema.ts
│       ├── payment.schema.ts
│       ├── report-settings.schema.ts
│       ├── aggregate-rule.schema.ts
│       ├── remark-template.schema.ts
│       └── remark-generation.schema.ts
│
├── hooks/
│   ├── useCurrentContext.ts               Active school/year/term from school_current_context
│   ├── useScores.ts
│   └── useNotifications.ts
│
├── store/
│   └── useSchoolStore.ts                  Zustand: school, user, active context
│
├── supabase/
│   ├── migrations/
│   │   ├── 001_initial.sql                Schools, years, terms, current_context
│   │   ├── 002_sequences.sql              id_sequences table + allocate_next_id()
│   │   ├── 003_staff_classes.sql          Staff, classes (without class_teacher_id FK)
│   │   ├── 004_staff_class_fk.sql         Add class_teacher_id FK
│   │   ├── 005_students.sql               Students + student_enrollments
│   │   ├── 006_guardians.sql              guardians + student_guardians
│   │   ├── 007_subjects.sql               Subjects, subject groups, aggregate rules
│   │   ├── 008_scores.sql                 Scores + scoresheet_submissions
│   │   ├── 009_grading.sql                grading_schemas
│   │   ├── 010_assignments.sql            Assignments + assignment_scores
│   │   ├── 011_attendance.sql             Attendance
│   │   ├── 012_finance.sql                fee_types, fee_assessments, payments
│   │   ├── 013_reports.sql                student_reports + report_versions + report_settings
│   │   ├── 014_remarks.sql                remark_templates + remark_generation_log + thresholds
│   │   ├── 015_audit.sql                  audit_log + notifications
│   │   ├── 016_rls_policies.sql           All RLS policies
│   │   └── 017_indexes.sql                All performance indexes
│   └── seed/
│       ├── grading.sql                    GES standard grading schema
│       ├── remark_templates.sql           5–8 variants per required combination
│       ├── subjects.sql                   Standard GES subjects
│       └── subject_groups.sql             VERBAL/NUMERICAL/PRACTICAL/THEORY mappings
│
├── public/
│   ├── logo.png
│   └── report-card-bg.png
│
└── .env.local                             (see Appendix C)
```

**Remove these directories — they are not needed:**
```
lib/ai/gemini.ts
lib/ai/claude.ts
lib/ai/remarks/prompts.ts
lib/ai/audit.ts
```

---

# PART 14 — BUILD ORDER

## Phase 1 — Foundation

- Next.js project + Tailwind setup
- Supabase project + migrations 001–002 (schools, context, sequences)
- Supabase Auth + role-based routing + protected layouts
- Design system components (Button, Input, Modal, Toast, etc.)
- Mobile navigation (bottom tab bar + sidebar)
- `useCurrentContext` hook + `school_current_context` integration
- Audit log foundation

**Test:** Login works for Admin and Teacher. Protected routes redirect correctly. Active context pre-fills on all forms.

## Phase 2 — Core School Data

- Migrations 003–007 (staff, classes, students, guardians, subjects)
- Student registration form (all fields, validation, server-side ID generation)
- Staff registration form (password generation, `must_change_password`)
- Class management, subject management, subject-teacher assignment
- Student list/search/edit with enrollment history
- Guardian/contact management

**Test:** Register a student → server generates ID → student appears in list with correct class. Two simultaneous student creations produce unique IDs.

## Phase 3 — Academic Engine

- Migrations 008–009 (scores, grading)
- Score entry interface — teacher vertical list with auto-save
- Admin scoresheet grid view (color-coded)
- Scoresheet status machine (DRAFT → SUBMITTED → LOCKED)
- Grade computation (client-side preview + server-side authority)
- Position recomputation at scoresheet submission boundary (RANK() OVER)
- Aggregate rule engine (`computeAggregate`)
- Report readiness checker

**Test:** Teacher enters scores → grades appear instantly → admin sees completion status → positions computed correctly with ties.

## Phase 4 — Remarks Engine

- Migration 014 (remark_templates, remark_generation_log, thresholds)
- `classify.ts` — deterministic pattern classification
- `templates.ts` — template fetching with fallback order
- `variants.ts` — deterministic initial selection + regenerate cycling
- `placeholders.ts` — substitution + validation
- `generate.ts` — main `generateRemark()` function
- `batch.ts` — bulk generation with partial failure handling
- Seed remark template bank (5–8 variants per combination)
- Full-screen remarks editor UI (list view + per-student editor)
- Remark template administration UI

**Test:** Generate remarks for students with 15+ different pattern combinations. Verify determinism, character limits, no placeholder leakage, correct pronouns.

## Phase 5 — Reports

- Migration 013 (student_reports, report_versions, report_settings)
- `resolveReportSettings()` with precedence chain
- Report settings UI (grouped sections, scope callout)
- Report readiness dashboard
- Report card PDF template (A4, GES format, from snapshot)
- Order of Merit (deterministic ranking + print/download)
- Publish/finalize flow + version snapshot creation

**Test:** Full end-to-end: enter scores → generate remarks → approve remarks → preview → generate PDF → publish → parent sees report.

## Phase 6 — Finance

- Migration 012 (fee_types, fee_assessments, payments)
- Fee type setup, assessment creation
- Payment recording with receipt numbers
- Balance calculation (assessment minus posted payments)
- Arrears report
- Void/correction flow

**Test:** Record partial payment → balance updates correctly → void payment → balance reverts.

## Phase 7 — Teacher Operations and Portals

- Assignments + assignment scores
- My Class view (roster + student profiles)
- Timetable view
- Teacher account settings + password change
- Student portal (own reports, scores, attendance)
- Parent portal (linked children + child switcher + published reports)
- SMS/email notification integration

**Test:** Parent receives notification when report published. Can switch between multiple children. Cannot access another student's data.

## Phase 8 — Operations / Polish

- Promotion workflow (preview → exceptions → confirm → execute → update enrollment history)
- Bulk operations with job records (idempotent, partial retry)
- Audit dashboard
- Data export (Excel export from any table)
- Report lock/unlock with confirmation and audit entry
- Performance review (indexes, compound queries)

---

# PART 15 — TESTING STRATEGY

## Unit Tests — Remarks Engine

Test at minimum:

```
HIGH_ACHIEVER only
HIGH_ACHIEVER + IMPROVING
ABOVE_AVERAGE + CONSISTENT
AVERAGE + STRONG_CLASSWORK_WEAK_EXAMS
BELOW_AVERAGE + DECLINING
STRUGGLING + LOW_ATTENDANCE
VERBAL_STRONG_NUMERICAL_WEAK
NUMERICAL_STRONG_VERBAL_WEAK
PRACTICAL_STRONG_THEORY_WEAK
THEORY_STRONG_PRACTICAL_WEAK
STRONG_CLASSWORK_WEAK_EXAMS
WEAK_CLASSWORK_STRONG_EXAMS
INCONSISTENT
CONSISTENT
No secondary pattern
Incomplete score record → MANUAL_REVIEW_REQUIRED
Missing previous term → no IMPROVING/DECLINING
Missing attendance → no attendance pattern
Other/unknown gender → neutral pronouns
Very long student name → character limit check
No active template → fallback chain
Fallback to system-safe template
Variant cycling sequence
Historical template version preserved on reprint
```

Each test verifies: correct pattern, correct template, correct variant, correct placeholder substitution, deterministic repeatability, character limit, no placeholder leakage, no incorrect pronouns, correct MANUAL_REVIEW_REQUIRED state when warranted.

## Concurrency Tests

- Two students created simultaneously → unique IDs both succeed
- Two staff created simultaneously → unique IDs
- Two teachers updating different scores → both succeed
- Two users attempting to modify a locked score → both rejected
- Two users attempting to publish the same report → exactly one succeeds
- Duplicate payment submission → rejected
- Duplicate report generation request → idempotent

## Security Tests

Attempt all of the following — all must fail appropriately:

```
Teacher → another teacher's scores
Teacher → another class
Teacher → locked scores (edit attempt)
Student → another student's records
Parent → unrelated student
Parent → unrelated fee record
Admin school A → school B records
Anonymous → protected endpoints
URL manipulation → another student's report PDF
```

## UX End-to-End Tests (on 390px mobile viewport)

**Admin:**
```
login → dashboard → register student → view generated ID
→ enter scores → readiness check → generate remarks
→ review/edit remarks → preview report → publish
```

**Teacher:**
```
login → dashboard (see pending tasks) → open pending scoresheet
→ enter scores → submit → see confirmation → mark attendance
```

**Parent:**
```
login → choose child → view published report → download PDF
→ view fee status
```

## UX QA Checklist (Every Core Screen)

```
[ ] No horizontal clipping of essential controls
[ ] 44px-ish touch targets on all interactive elements
[ ] Readable body text (minimum 14px)
[ ] Clear page title
[ ] Clear primary action
[ ] Loading state implemented
[ ] Success state implemented
[ ] Error state implemented
[ ] Empty state implemented
[ ] Confirmation for all destructive actions
[ ] Active year/term context visible (pre-filled from school_current_context)
[ ] No unnecessary repeated selectors
[ ] No browser-default buttons anywhere
```

---

# PART 16 — SECURITY CHECKLIST

Before launch, verify:

```
[ ] Every protected route checks authenticated session server-side
[ ] Every query is school-scoped (no cross-school data leakage)
[ ] Teachers can only access assigned subjects/classes
[ ] Teachers cannot edit locked scores
[ ] Teachers cannot delete academic records
[ ] Students can only see their own published records
[ ] Parents can only see linked children via student_guardians
[ ] Finance data is role-protected (not accessible to teachers/students)
[ ] Audit logs are restricted (not accessible to teachers/students)
[ ] Passwords are never stored or logged in plaintext
[ ] Destructive actions require server-side authorization (not just client-side UI)
[ ] Student IDs cannot be generated or guessed from frontend code
[ ] Report PDFs are path-secured (not guessable by URL manipulation)
[ ] REVOKE ACCESS has confirmation dialog
[ ] All client-supplied IDs (classId, studentId, termId, schoolId) are validated
    against the authenticated user's permissions on the server
```

---

# PART 17 — ORDER OF OPERATIONS (END OF TERM)

```
1.  Teachers enter scores
2.  Teachers submit scoresheets (status → SUBMITTED)
3.  System validates scores and computes grades (server-side)
4.  System computes subject positions (transactionally)
5.  System verifies attendance completeness
6.  System calculates aggregate/overall position (deterministic rule)
7.  System checks report readiness → surfaces blockers
8.  System generates deterministic remarks (batch)
9.  Admin reviews / edits / approves remarks
10. System previews reports (admin review)
11. Admin publishes / finalizes reports (status → PUBLISHED)
12. System creates report version snapshots + PDFs
13. System locks scores according to finalization policy
14. System queues parent/student notifications
15. Order of Merit becomes final and available to print
```

---

# APPENDIX A — GES GRADING SCHEMA

| Score Range | Grade | Remark |
|-------------|-------|--------|
| 80 – 100    | A1    | Excellent |
| 70 – 79     | B2    | Very Good |
| 60 – 69     | B3    | Good |
| 50 – 59     | C4    | Credit |
| 45 – 49     | C5    | Credit |
| 40 – 44     | C6    | Credit |
| 35 – 39     | D7    | Pass |
| 30 – 34     | E8    | Pass |
| 0  – 29     | F9    | Fail |

---

# APPENDIX B — REMARK PATTERN REFERENCE

| Pattern | Detected When | Remark Strategy |
|---------|---------------|-----------------|
| HIGH_ACHIEVER | Average ≥ 80 | Acknowledge excellence; challenge to maintain |
| ABOVE_AVERAGE | Average 65–79 | Encourage pushing for top |
| AVERAGE | Average 50–64 | Encourage without overstating |
| BELOW_AVERAGE | Average 40–49 | Kind; build from one strength |
| STRUGGLING | Average < 40 | Compassionate; focus on effort |
| VERBAL_STRONG_NUMERICAL_WEAK | Verbal avg 15+ above numerical (configured threshold) | Acknowledge verbal; encourage maths |
| NUMERICAL_STRONG_VERBAL_WEAK | Numerical avg 15+ above verbal | Acknowledge analytical; encourage language |
| PRACTICAL_STRONG_THEORY_WEAK | Practical avg 15+ above theory | Acknowledge creativity; connect to academics |
| THEORY_STRONG_PRACTICAL_WEAK | Theory avg 15+ above practical | Acknowledge academics; encourage hands-on |
| STRONG_CLASSWORK_WEAK_EXAMS | Class% 15+% above exam% (normalized) | Acknowledge effort; exam prep strategies |
| WEAK_CLASSWORK_STRONG_EXAMS | Exam% 15+% above class% | Acknowledge capability; encourage daily work |
| CONSISTENT | Score spread ≤ configured threshold | Acknowledge balance |
| INCONSISTENT | Score spread ≥ configured threshold | Acknowledge peaks; encourage evenness |
| IMPROVING | Position improved by ≥ configured threshold | Acknowledge improvement directly |
| DECLINING | Position declined by ≥ configured threshold | Gentle re-engagement |
| STABLE | Position similar to last term | Acknowledge consistency |
| LOW_ATTENDANCE | Attendance < configured threshold | Note importance gently |
| EXCELLENT_ATTENDANCE | Attendance ≥ configured threshold | Optionally acknowledge |

Primary tier: **exactly one**. Secondary: **at most one**, chosen by priority order.

---

# APPENDIX C — ENVIRONMENT VARIABLES

```env
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://[project-id].supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=[anon-key]
SUPABASE_SERVICE_ROLE_KEY=[service-role-key]

# Optional notification channels (not required for core academic functions)
TWILIO_ACCOUNT_SID=[sid]
TWILIO_AUTH_TOKEN=[token]
TWILIO_PHONE_NUMBER=+1[number]

RESEND_API_KEY=[resend-key]
RESEND_FROM_EMAIL=noreply@cerican.com

NEXT_PUBLIC_APP_URL=https://cerican.com
CRON_SECRET=[random-secret-for-cron-auth]

# NOTE: GEMINI_API_KEY and ANTHROPIC_API_KEY are NOT required for the
# production remarks system. Only include them if another separately
# defined production feature independently requires them.
```

---

# APPENDIX D — THINGS GEMINI MUST NOT DECIDE INDEPENDENTLY

The coding model must not make its own interpretation on these — implement the configuration point instead of guessing:

1. Student ID allocation method
2. Historical student class placement (use enrollment history)
3. Aggregate subject selection (use aggregate_rules)
4. Score completeness definition (null ≠ zero)
5. Grade band behavior when no band matches (structured error)
6. Tie/position behavior (declare and document)
7. Assignment contribution to official scores (explicit config, off by default)
8. Report readiness definition (based on configured required inputs)
9. Report finalization/locking policy (config, not PDF generation)
10. Parent-to-child authorization (student_guardians only)
11. Teacher score edit permissions (assigned subjects + unlocked)
12. Remark primary/secondary priority (use defined order)
13. Remark missing-data behavior (MANUAL_REVIEW_REQUIRED)
14. Remark variant determinism (stable hash key)
15. Remark template versioning (new version, old unchanged)
16. Financial payment history (never overwrite — use void/correction)
17. Promotion exceptions (requires explicit exception handling)
18. Audit log redaction (never log passwords/tokens)
19. Notification failure behavior (never roll back core transactions)
20. Multi-school data isolation (school_id on every query)

---

# APPENDIX E — WHAT THIS SYSTEM SHOULD FEEL LIKE

When built correctly, every user should be able to say:

```
I know what term I am working in.
I know what needs my attention right now.
I know when my data was saved.
I know who can change what.
I can complete core workflows on a phone.
I cannot accidentally destroy important records.
I can see exactly why a report is not ready.
I can review every generated remark before publishing.
The same student's report is reproducible from any past term.
Financial transactions remain historically accurate.
Parents only see their own children.
Teachers only see the records they are authorized to manage.
```

---

*END OF CERICAN MASTER BUILD DOCUMENT*
*Version: Combined (V2 Implementation Spec + Complete Blueprint + UX Analysis)*
*Date compiled: September 10, 2026*
*Give this entire document to Google AI Studio as the system/context for Gemini.*
