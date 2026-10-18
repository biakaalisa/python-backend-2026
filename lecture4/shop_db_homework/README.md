# Shop API + PostgreSQL + transaction isolation

## What this includes

- FastAPI implementation of Homework 2: items, soft deletion, carts, pagination and filters.
- SQLAlchemy 2 models, PostgreSQL database in Docker Compose with a persistent named volume.
- Independent SQLAlchemy sessions per request and committed mutations.
- Repeatable, assert-based experiments for PostgreSQL transaction isolation.

## Start (PowerShell)

From **this directory**:

```powershell
docker compose up --build -d
docker compose ps
```

API documentation: http://localhost:8000/docs

Database port for local development: `localhost:5432`, credentials: `shop` / `shop_dev_password`, database: `shop`. These are **local demo credentials**; set `POSTGRES_PASSWORD` in `.env` for anything else. Do not commit `.env`.

An existing port 8000 or 5432 (e.g. from your earlier monitoring homework) will cause a conflict. Stop that Compose project first.

The default container database survives `docker compose down` (but NOT `docker compose down -v`).

## Isolation demonstrations (PostgreSQL)

```powershell
docker compose run --rm api python transactions/isolation_demo.py
```

The script runs two simultaneous transactions (connections A and B), uses assertions, and reports the observed results. The demo table `isolation_demo` is independent of the shop tables, but each run truncates **that demo table**. Do not run concurrent copies of the experiment.

| Phenomenon / requested level | What PostgreSQL actually shows |
| --- | --- |
| Dirty read at READ UNCOMMITTED | **Impossible**: PostgreSQL treats it as READ COMMITTED; B sees committed value 100, not uncommitted 999 |
| Dirty read at READ COMMITTED | Impossible; B sees 100 |
| Non-repeatable read at READ COMMITTED | Reproduced: 100 -> 200 |
| Non-repeatable read at REPEATABLE READ | Prevented: 100 -> 100 |
| Phantom at REPEATABLE READ | **Impossible**: PostgreSQL's snapshot isolation prevents it; count stays 1 -> 1 |
| Phantom at SERIALIZABLE | Prevented: 1 -> 1 |

We also demonstrate a **write skew** anomaly prevented at SERIALIZABLE using two worker transactions. One fails with PostgreSQL SQLSTATE `40001` (serialization failure); an actual application should retry that transaction. The script does not fake dirty/phantom reads that PostgreSQL cannot exhibit.

## Tests

To run the supplied original HW2 test suite *without Docker*, you may use a SQLite file for testing only:

```powershell
python -m pip install -r requirements.txt
$env:DATABASE_URL="sqlite:///./test_shop.db"
$env:PYTHONPATH="."
pytest test_homework2.py -q
```

To run the original tests against the PostgreSQL container instead, first start Compose, then set `DATABASE_URL` to `postgresql+psycopg://shop:shop_dev_password@localhost:5432/shop` and run pytest. These tests create rows and are not isolated from the normal application database, so use a dedicated test DB for regular automated testing.

## Notes

Tables are created on first application import/start for simple homework setup. A production project would use Alembic migrations and separate test databases. The homework does not implement production user authentication.
