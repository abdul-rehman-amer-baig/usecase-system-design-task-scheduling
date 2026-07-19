# System Design: News Scheduling Architecture

This blog explains how I landed on the right service to accomplish this task. I will walk you through my thought process as you read.

## Task Description

We have to manage a news feature that supports multiple actions:

1. Publish News
2. UnPublish News
3. Delete News
4. Schedule News

In this article we are going to discuss the **Schedule News** feature where users can set a time at which the news will be published. I came up with multiple approaches and let us dive deeper into each one and understand their pros/cons.

---

## Stage #01: Celery Beat + Redis Broker

![Celery Beat Architecture](images/redis-celery-beat.png)

- We have integrated **Celery Beat** which is responsible for scheduling news at its due time, and **Celery Worker** which is responsible for executing the task
- Redis as a broker using two logical databases:
  - `/0` as the task queue — Beat `LPUSH`es tasks into the `/0` when their scheduled time arrives
  - `/1` as the result backend — stores return values of executed tasks temporarily (optional if you want to display the values in terminal/logs)
- Celery Beat polls PostgreSQL every 10 minutes, checks if any scheduled task is due, and if so `LPUSH`es it to Redis. Each worker child holds an open `BRPOP` connection to Redis — when the task arrives, Redis wakes the child that has been waiting the longest and hands it the task for execution
- Flower subscribes (event listener on) to `/0` redis logical DB and it captures the real time event emitted/published by celery as it processes the task. We can also configure different redis logical database `/1` for result to display custom values for our own readability. But real time events always go to the `/0`
  - The following kind of events are published by celery to `/0` database as it process the task:

    ```
    → "task-received"
    → "task-started"
    → "task-succeeded" / "task-failed"
    ```

  - For custom results, Flower does not subscribe to `/1` as it does with `/0`. Instead it performs an on-demand lookup to `/1` using the **task ID** as the key whenever you view a specific task in the dashboard.

### Notes

- The main pro of BRPOP connection is to keep the celery worker child in sleeping mode and pings it once a new task is arrived. It solves the problem of continuous polling
- In our use case, we allow admins to schedule news in 30 minute intervals. Regardless of timezone differences, only the hour offset differs across regions — minutes and seconds remain perfectly in sync everywhere.
  - We fire the `publish_scheduled_news` task every 10 minutes instead of every minute to avoid overloading PostgreSQL with connections. Each worker child opens its own database connection when executing the task, and if multiple children are running simultaneously each holds a separate connection. Since PostgreSQL has a default `max_connections` limit of 100 — shared across your FastAPI app, worker children, and other services — keeping the task frequency reasonable helps stay within that limit.

---

## Stage #02: Redis-based Task Queue with ETA Check

![Redis Celery Architecture](images/redis-celery.png)

- The main problem with **Stage #01** is the consumption of PostgreSQL connections. Since PostgreSQL has a default max connection limit of `100`, there might be cases where multiple celery workers are already holding the PostgreSQL connection. When we receive a new task, another Celery worker has to open a new connection to process it. This can become a bottleneck for FastAPI application that requires frequent PostgreSQL connections.
- We replaced this with Redis-based approach, where we set the ETA of the task and Celery workers access these tasks through `BRPOP` commands. They check the ETA, and if scheduled time has not arrived yet, the task is pushed back to Redis as a pending task.
- Since Redis offers persistence to some extent we can hold at most xyz MB of data on Redis, depending on the plan. However, Redis is built as a cache-first service so persisting tasks on redis would not make our solution reliable.
  - To decipher this unreliability, we need to first look at how Redis persistence works. It basically takes the snapshot periodically and these snapshots help Redis recover tasks in case of a failure or service restart.
  - There might be a case, that our task is `LPUSH`(ed) at 11:45 AM but snapshot happens at 12:00 and if we restart the service between 11:45 AM ↔ 12:00 PM window that task will be lost

---

## Stage #03: AWS EventBridge (Final Solution)

![AWS EventBridge Architecture](images/eventbridge.png)

- Based on the drawbacks we discussed above, to build a more reliable solution, AWS provides a service that fits our use case: **Amazon EventBridge**.
- We can schedule the task, and EventBridge, without depending on any other service, will fire it purely based on the clock when the scheduled time arrives.
- Amazon EventBridge is a serverless event-routing service. It offers several features through which we can accomplish our tasks:
  - **Scheduler**: lets us store a task on EventBridge with timezone awareness. We keep a reference to the stored task in our local Postgres DB, so we can update the time if the admin edits the task, or delete it using its `task_id`.
  - **API destination**: when the scheduled task is due, Scheduler invokes the configured target, the API destination, which calls our configured HTTP API (in our case, to publish news to our local DB).
  - **Connection**: our endpoint requires authentication, backed by an API key, so the API destination is paired with a Connection that holds the API key and credentials. The Connection stores the credential in AWS Secrets Manager under the hood.
---