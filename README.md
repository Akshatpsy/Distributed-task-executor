# Distributed Task Executor

Runs jobs made of tasks that depend on each other (a DAG), spreads the work across multiple workers, and recovers when a worker crashes. Built in Python with PostgreSQL, an AMQP message queue and FastAPI. Solo project.

## What it does

- Takes a set of tasks with dependencies and works out a valid run order using Kahn's algorithm (topological sort).
- A scheduler sends tasks that are ready to run to an AMQP queue. Workers pick them up and run them.
- All job and task state is stored in PostgreSQL.
- Handles failures: retries, a dead-letter queue, heartbeat and lease based crash recovery, and idempotent execution.
- A REST API (FastAPI) with WebSocket live updates, and a dashboard in `static/`.

## Architecture

```
Client / Dashboard
        |
   FastAPI (REST + WebSocket)
        |
   PostgreSQL  <---->  Scheduler  --->  AMQP queue  --->  Workers
   (state)                                                   |
        ^---------------------------------------------------|
                  workers write results and heartbeats
```

## How failures are handled

- **Worker crash:** each running task holds a lease that the worker keeps renewing with heartbeats. If the lease expires after [N seconds], the scheduler treats the worker as dead and puts the task back in the queue.
- **Retries:** a failed task is retried up to [N] times. After that it goes to the dead-letter queue.
- **Idempotency:** a task that runs twice after a crash does not cause duplicate results. Each task run has a unique key, and the worker checks PostgreSQL before running. A task already marked completed is skipped, and the final status update only succeeds once..

## Run it locally

```bash
cp .env.example .env            # set DATABASE_URL and RABBITMQ_URL
docker compose up -d            # [confirm what this starts: broker, database, or both]
pip install -r requirements.txt
psql "$DATABASE_URL" -f schema.sql
uvicorn src.api:app --port 8000
python -m src.[scheduler-module]   # in a second terminal
python -m src.[worker-module]      # in a third terminal
```

Then submit one of the jobs in `examples/` and watch it run on the dashboard at `http://localhost:8000`.

## Tests

```bash
python -m pytest
```
## Live demo

The hosted demo runs only the API and dashboard. The scheduler and worker are not running there, so tasks you submit stay in `pending` or `queued` by design. To see tasks actually execute, run the project locally or watch.

Hosted on: API on Railway, PostgreSQL on Neon, queue on a hosted AMQP broker.

## Limitations

- The hosted demo runs only the API and dashboard. Run the project locally to see tasks execute.
- Free-tier hosting: the Neon database may expire after inactivity and need its schema rebuilt.
- Runs with a single scheduler instance, so the scheduler is a single point of failure.
- Tasks can run more than once after a crash, so tasks must be safe to repeat.
- Not load tested with large numbers of tasks or workers.

## Project layout

```
src/        API, scheduler, worker
static/     dashboard
examples/   sample jobs
tests/      automated tests
schema.sql  database schema
```
