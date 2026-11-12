# course-catalog-api

Backend of the course catalog: FastAPI, SQLAlchemy 2.1, PostgreSQL, and Alembic
migrations (Labs 4–7 of "Advanced Web Technologies").

## How to run

You need Python 3.11 or newer and PostgreSQL 16 or newer.

Once, as the `postgres` superuser, create the role and the database:

```sql
CREATE ROLE catalog_app LOGIN PASSWORD 'change-me';
CREATE DATABASE course_catalog OWNER catalog_app;
```

Then, from the project folder:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
alembic upgrade head
psql -h localhost -U catalog_app -d course_catalog -v ON_ERROR_STOP=1 -f seed.sql
alembic check
fastapi dev main.py
```

API documentation: http://localhost:8000/docs. The password `change-me` is for
teaching; in a real project, passwords are never put into the repository.

## What changed in the API

- **Persistent storage in PostgreSQL**: Courses, students, and enrollments now live in the `course_catalog` PostgreSQL database instead of in-memory Python lists (`data.py` and `ENROLLMENTS`), so all likes, new students, and enrollments survive server restarts.
- **Updated `POST /enrollments` contract**: Instead of `studentName`, the request body now expects `{"courseId": "...", "studentId": ...}` referencing an existing student's primary key (`enrollments.student_id` foreign key), and returns `id`, `courseId`, `studentId`, and server-generated `enrolledAt`.
- **New routes and joined enrollment details**: Added `POST /students` (`201 Created`) to register students, `GET /stats/courses` (`LEFT OUTER JOIN` + `count(enrollments.id)`) to return enrollment counts per course including `0`, and enriched `GET /enrollments` with a two-table `JOIN` returning `courseTitle` and `studentName`.
- **Course `level` field and filter**: Every course now includes a required `level` field (`beginner`, `intermediate`, or `advanced`), and `GET /courses` supports optional filtering via `?level=beginner` alongside `?is_elective=`, `sort`, `page`, and `page_size`.
- **Database constraint errors mapped to HTTP responses**: `IntegrityError` exceptions caught on `db.commit()` are inspected via `exc.orig.diag.constraint_name` and translated into `409 Conflict` (duplicate student email or duplicate course enrollment) and `404 Not Found` (unknown `course_id` or `student_id`), while unknown constraint violations re-raise as `500`.

## Migrations

| Command | What it does |
|---------|--------------|
| `alembic upgrade head` | applies all the missing migrations |
| `alembic downgrade -1` | rolls back the last migration |
| `alembic current` | shows the version of the database |
| `alembic history` | shows the chain of migrations |
| `alembic revision --autogenerate -m "text"` | writes a draft from models vs. database |
| `alembic check` | reports whether the models and the database have diverged |
| `alembic upgrade 0001:0002 --sql` | prints the SQL of a migration without running it |
| `alembic stamp head` | marks the database as up to date without running migrations |

## Experiment log

| # | Experiment | What I saw | Cause | How I fixed it |
|---|------------|------------|-------|----------------|
| 1 | the `uq` template in `naming_convention` changed | `check_models.py` printed `names only in models: ['enrollments_course_id_unique', 'students_email_unique']` and `names only in database: ['enrollments_course_id_student_id_key', 'students_email_key']` | Changing the `uq` naming template to `%(table_name)s_%(column_0_name)s_unique` made `Base.metadata` generate constraint names that differed from PostgreSQL's default `..._key` names | Restored `"uq": "%(table_name)s_%(column_0_N_name)s_key"` in `db.py` and re-ran `check_models.py` (`none` / `none`) |
| 2 | `min_length` removed from `StudentIn`, name `N` | `POST /students` returned `500 Internal Server Error`; server log showed `IntegrityError: (psycopg.errors.CheckViolation) ... violates check constraint "students_name_check"` | Without Pydantic's `Field(min_length=2)`, the invalid 1-character name reached PostgreSQL, triggered `students_name_check`, and was re-raised as `500` because `students_name_check` is not in `CONSTRAINT_ERRORS` | Restored `name: str = Field(min_length=2)` in `StudentIn` (`schemas.py`) so Pydantic returns `422` before hitting the database |
| 3 | a typo in a constraint name in the dictionary | `POST /students` with a duplicate email returned `500 Internal Server Error` and logged `UniqueViolation: duplicate key value violates unique constraint "students_email_key"` | `raise_http_error` looked up `"students_email_key"` in `CONSTRAINT_ERRORS`, missed the mistyped key `"students_email_uniq"`, and re-raised the `IntegrityError` | Restored the exact key `"students_email_key"` in `CONSTRAINT_ERRORS` in `main.py` |
| 4 | `title` without `NOT NULL` in the database, `alembic check` | `alembic check` failed with `FAILED: New upgrade operations detected: [[('modify_nullable', None, 'courses', 'title', ...)]]` (and `modify_type` when changed to `varchar(100)`) | `alembic check` compares `Base.metadata` (`Mapped[str]` -> `TEXT NOT NULL`) against the live database catalog and detects manual schema drift | Ran `ALTER TABLE courses ALTER COLUMN title SET NOT NULL;` and `ALTER COLUMN title TYPE text;` in `psql` |
| 5 | automatic migration `0002` on existing data | `alembic upgrade head` failed with `IntegrityError: (psycopg.errors.NotNullViolation) column "level" of relation "courses" contains null values` | Autogenerate emitted `ALTER TABLE courses ADD COLUMN level TEXT NOT NULL` in one step, which PostgreSQL rejects on a table that already contains 6 rows because existing rows have `NULL` | Rewrote `0002_add_course_level.py` into 3 steps: add `level` with `nullable=True`, backfill rows via `UPDATE`, then `alter_column(..., nullable=False)` |
| 6 | `import models` commented out in `env.py` | `alembic revision --autogenerate` generated `op.drop_index` and `op.drop_table` for `enrollments`, `courses`, and `students` | Without importing `models`, model classes never registered their tables in `Base.metadata`, so Alembic saw an empty target schema and concluded all DB tables should be dropped | Deleted `0003_oops.py` and uncommented `import models  # noqa: F401` in `alembic/env.py` |
| 7 | a `phone` field in the model without a migration | `alembic check` failed with `FAILED: New upgrade operations detected: [('add_column', None, 'students', Column('phone', Text(), table=<students>))]` | Adding `phone: Mapped[str \| None]` to `Student` updated `Base.metadata` while the database remained at revision `0002` without the column | Removed `phone: Mapped[str \| None]` from `Student` in `models.py` (or in real work, generate and apply a migration) |
| 8 | two migrations from one revision | `alembic heads` listed both `0003 (head)` and `0004 (head)`; `alembic upgrade head` failed with `FAILED: Multiple head revisions are present for given argument 'head'` | Creating `0003` and `0004` both branching from `0002` created a split migration graph with two leaf heads, so Alembic could not determine a single linear target | Ran `alembic merge heads -m "merge phone and language" --rev-id 0005` to test merging, then removed experimental revisions `0003`, `0004`, and `0005` |

## Decisions

- **Why the API response is chosen by the constraint name rather than by SQLSTATE, and why the error is caught after `commit` rather than checked in advance with a `SELECT`:**
  - **Constraint name vs. SQLSTATE**: A single SQLSTATE code covers multiple distinct domain errors. For example, `23505` (`unique_violation`) fires for both duplicate student email (`students_email_key`) and duplicate enrollment (`enrollments_course_id_student_id_key`), while `23503` (`foreign_key_violation`) fires for both missing course (`enrollments_course_id_fkey`) and missing student (`enrollments_student_id_fkey`). Inspecting `exc.orig.diag.constraint_name` identifies the exact violated rule regardless of PostgreSQL locale or version.
  - **Catching `IntegrityError` on `commit()` vs. pre-checking with `SELECT`**: Checking with a `SELECT` before `INSERT` introduces a **race condition (TOCTOU — Time-of-Check to Time-of-Use)** where two concurrent requests both see that an email is free and both attempt to insert, and it also adds an extra database round-trip on every successful request. Relying on PostgreSQL's atomic constraint check during `commit()` ("easier to ask forgiveness than permission") guarantees correctness under concurrency in a single query.
- **Why a like is incremented with `UPDATE courses SET likes = likes + 1` rather than by reading the number into the program:**
  - Reading `course.likes` into Python (`11`), adding `1` in memory (`12`), and writing `SET likes = 12` causes the **lost update** anomaly when two concurrent requests read `11` at the same time — both write `12`, losing one like. Executing `UPDATE courses SET likes = likes + 1 WHERE id = ... RETURNING ...` performs an atomic server-side increment: PostgreSQL acquires a row-level lock on the target row so concurrent updates serialize cleanly (`11 -> 12 -> 13`).
- **Why migration `0002` is split into three steps:**
  - Adding a `NOT NULL` column without a `DEFAULT` (`ALTER TABLE courses ADD COLUMN level TEXT NOT NULL`) to a populated table fails immediately with `NotNullViolation` because PostgreSQL creates the column with `NULL` for existing rows before any `UPDATE` can run. Splitting the migration into (1) `add_column(..., nullable=True)`, (2) `op.execute("UPDATE courses SET level = ...")` to populate all existing rows according to business logic, and (3) `alter_column(..., nullable=False)` inside a single transactional DDL block safely transitions live data without downtime or data loss.

## Migration 0002 self-check

| What to check | Result |
|---------------|--------|
| Goal and steps | **Yes** — Focused solely on adding the `courses.level` attribute in three clear steps (`add_column` nullable, `UPDATE` backfill, `alter_column` `NOT NULL`). |
| Data | **Yes** — Existing 6 courses are preserved and backfilled deterministically (`beginner` for electives, `intermediate` for core courses) before enforcing `NOT NULL`. |
| Renames and drops | **Yes** — `upgrade()` performs no column drops or renames, so zero data is lost when upgrading. |
| Rollback | **Yes** — `downgrade()` cleanly executes `op.drop_column('courses', 'level')`; on production, rolling back would drop the `level` column and any custom level values. |
| Verification | **Yes** — Verified static SQL via `alembic upgrade 0001:0002 --sql`, tested the full `upgrade -> downgrade -1 -> upgrade head` cycle, and confirmed `alembic check` returns `No new upgrade operations detected.` |
| History | **Yes** — `alembic heads` shows a single head (`0002 (head)`), and migration `0002` is committed together with the `models.py`, `schemas.py`, `main.py`, and `seed.sql` updates. |
| Application | **Yes** — `CourseOut` returns `level`, `GET /courses` supports `?level=`, and `seed.sql` populates `level` for all 6 courses with `TRUNCATE ... RESTART IDENTITY`. |

## Bonus

### Task 1. A `CHECK` that `autogenerate` does not see (`op.f`)

- **Why Alembic does not detect `CheckConstraint` changes**: Alembic's autogenerate does not parse or compare arbitrary SQL text expressions inside `CHECK` constraints because PostgreSQL normalizes and stores expressions in internal form (`pg_get_constraintdef`), making reliable AST comparison impossible. Therefore, `CheckConstraint` additions/removals must be written manually using `op.create_check_constraint` and `op.drop_constraint`.
- **Why `op.f(...)` is required**: When `MetaData(naming_convention=...)` is configured on `Base`, Alembic applies the `ck` template (`%(table_name)s_%(constraint_name)s_check`) to any plain string name passed to `op.create_check_constraint` or `op.drop_constraint`. Passing `"courses_level_check"` without `op.f(...)` in `downgrade()` expanded it twice into `"courses_courses_level_check_check"`, causing `UndefinedObject`. Wrapping the name in `op.f("courses_level_check")` marks the identifier as already formatted.

### Task 2. Renaming a column without losing data (`alter_column`)

- **Difference between `op.alter_column(..., new_column_name=...)` and `add_column` + `drop_column`**: Alembic's `--autogenerate` cannot know whether a column was renamed or replaced, so renaming `description` to `summary` in `Course` produces `op.add_column('courses', sa.Column('summary', ...))` followed by `op.drop_column('courses', 'description')`. That destroys all existing descriptions and fails with `NotNullViolation` on a non-empty table. Using `op.alter_column("courses", "description", new_column_name="summary")` emits `ALTER TABLE courses RENAME COLUMN description TO summary`, which updates only the catalog metadata in-place while preserving all row data.

### Task 3. The N+1 problem (`try_n_plus_1.py`)

- **Result**:
  - `lazy          enrollments: 6, queries: 7` (1 query for the 6 courses + 6 separate `SELECT ... FROM enrollments WHERE course_id = ...` queries, one per course).
  - `selectinload  enrollments: 6, queries: 2` (1 query for courses + 1 batched `SELECT ... FROM enrollments WHERE enrollments.course_id IN (...)` query).
- **How to spot N+1 and `selectinload` vs `joinedload`**: With `SQL_ECHO=true`, N+1 appears as a burst of identical `SELECT` statements differing only in the foreign-key parameter inside a loop. `selectinload` emits a second query with `WHERE fk IN (...)` (best for `one-to-many` collections because parent columns are not duplicated across rows), whereas `joinedload` fetches parent and child rows in a single `LEFT OUTER JOIN` query (best for `many-to-one` relationships like `Enrollment.course`).

### Task 4. A lost update in the ORM (`try_lost_update.py`)

- **Why `a.likes += 1` loses updates**: Both concurrent sessions (`alice` and `bob`) read `likes = 11` into Python objects. `a.likes += 1` and `b.likes += 1` compute `12` in Python memory and send `UPDATE courses SET likes = 12 WHERE id = 'api-design'`. The second commit overwrites the first with `12`, losing one increment.
- **Why `a.likes = Course.likes + 1` prevents lost updates**: Assigning the SQL expression `Course.likes + 1` tells SQLAlchemy to emit `UPDATE courses SET likes = (courses.likes + 1) WHERE courses.id = 'api-design'`. When `alice` executes the `UPDATE`, PostgreSQL acquires a row-level `FOR NO KEY UPDATE` lock on that row; `bob`'s `UPDATE` waits until `alice` commits (`12`), then re-evaluates `courses.likes + 1` on the committed row and writes `13`.

### Task 5. `passive_deletes=True` and `ON DELETE CASCADE` (`try_delete.py`)

- **Why `passive_deletes=True` is needed alongside `ON DELETE CASCADE`**: By default (`passive_deletes=False`), when `session.delete(student)` is called, SQLAlchemy's unit-of-work loads the student's child `enrollments` and attempts to dissociate them by executing `UPDATE enrollments SET student_id = NULL WHERE id = ...` before deleting the parent row. Because `enrollments.student_id` is `NOT NULL`, PostgreSQL rejects the `UPDATE` with `NotNullViolation`. Setting `passive_deletes=True` on `Student.enrollments` tells SQLAlchemy not to load or nullify child rows and to emit only `DELETE FROM students WHERE id = 3`, letting PostgreSQL's `ON DELETE CASCADE` foreign-key rule automatically delete the student's enrollments (`students: 3 -> 2`, `enrollments: 6 -> 5`).
