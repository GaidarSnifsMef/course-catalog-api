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

API will be available at `http://127.0.0.1:8000` and interactive documentation at `http://127.0.0.1:8000/docs`.

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
