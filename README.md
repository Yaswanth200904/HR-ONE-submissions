# Employee Attendance & Analytics API

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export MONGO_URI="mongodb://localhost:27017" MONGO_DB="attendance_db"   # or put them in a local .env
uvicorn app.main:app --port 8000
```
Needs MongoDB 6.0+ (developed against 7.0; uses `$setWindowFields`/`$rank`). Indexes are created at startup.

Local database (Windows with Docker Desktop):

```powershell
Copy-Item .env.example .env
docker compose up -d mongo
docker compose ps
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
uvicorn app.main:app --port 8000
```

The MongoDB container initializes `attendance_db` with demo employees and attendance on its first start. Its named volume keeps data between restarts; `docker compose down` stops the service without deleting data. The database port is bound to localhost and has no authentication, so this compose setup is for local development only. Open `http://localhost:8000/` for the operations UI or `http://localhost:8000/docs` for the API.

Notes
- The operations UI is served at `http://localhost:8000/`; interactive API documentation is at `http://localhost:8000/docs`.
- API and analytics logic is in `app/main.py`; the browser UI is in `app/static/`. Analytics are single MongoDB aggregation pipelines; only the final half-up rounding is done in Python (Mongo's `$round` is banker's rounding).
- `/analytics/.../trend` builds the calendar inside the pipeline with `$range` instead of `$densify` (same result: one row per day).
- `/admin/explain/{endpoint}` builds the query with the same builder functions as the real endpoint.
