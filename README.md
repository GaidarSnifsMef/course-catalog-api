# Course Catalog API

A standalone FastAPI backend for a course catalog with filtering, sorting, and pagination.

## How to run

1. Create and activate a virtual environment:
```bash
python3 -m venv .venv
source .venv/bin/activate
```

2. Install dependencies:
```bash
pip install -r requirements.txt
```

3. Start the development server:
```bash
fastapi dev main.py
```

API will be available at `http://localhost:8000` and interactive documentation at `http://localhost:8000/docs`.

## What changed in Lab 5

The API permits browser requests only from `http://localhost:3000`. This CORS rule lets the Next.js client read API responses while other origins receive no browser permission. JSON response and request-body field names use camelCase through `CamelModel`; Python code still uses snake_case.

- `POST /courses/{course_id}/like` increments the in-memory course counter and returns the updated course.
- `POST /enrollments` accepts `{ "courseId": "web-security", "studentName": "Aigerim" }` and returns `201` with an enrollment id.
- `GET /enrollments` returns all enrollments received since the FastAPI server last started.

Likes and enrollments are deliberately in memory for this lab, so restarting the backend resets them.

## What was verified

- `GET /` — returns status message: `{"message": "Course Catalog API is running"}`
- `GET /courses` — returns all 6 courses sorted by popularity (likes) descending (`ai-integration` first)
- `GET /courses/web-security` — returns course data for `web-security` (200 OK)
- `GET /courses/nope` — returns 404 Not Found: `{"detail": "Course not found"}`
- `GET /courses?is_elective=true` — returns 2 elective courses
- `GET /courses?is_elective=false` — returns 4 non-elective courses
- `GET /courses?sort=title` — returns courses sorted alphabetically by title
- `GET /courses?page=2&page_size=2` — returns 2 courses for page 2 (`web-security`, `backend-fastapi`)
- `GET /docs` — interactive Swagger UI with all 3 endpoints and schema models
