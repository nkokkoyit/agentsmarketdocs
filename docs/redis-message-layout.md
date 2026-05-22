# Redis Message Layout

## Scope

Tai codebase nay, Redis duoc dung cho `Celery broker` va `Celery result backend`, khong phai noi luu business record chinh cua task.

Can cu local code:

- Celery dung chung `settings.redis_url` cho ca `broker` va `backend`: `app/infrastructure/queue/celery_app.py`
- API enqueue job qua `run_crew.apply_async(args=[job_id, crew_key, payload], task_id=job_id)`: `app/application/services/job_service.py`
- Worker consume task `app.workers.worker.run_crew`: `app/workers/worker.py`
- Runtime dependency dang pin `celery==5.3.6`, `redis==5.0.4`: `requirements.txt`

## Redis se co nhung gi cho 1 job

Voi 1 request:

```json
{
  "cv_text": "Alice, 3 years backend engineering. Skills: Python, FastAPI, SQL.",
  "job_description": "Looking for backend engineer with Python, FastAPI and SQL."
}
```

va `crew_key = "recruitment_screening"`, app se tao mot `job_id`, sau do Redis co the chua 3 nhom data:

1. Broker queue message trong list queue mac dinh `celery`
2. Trang thai in-flight cua worker trong `unacked` va `unacked_index`
3. Ket qua task trong key `celery-task-meta-<job_id>`

## 1. Broker Message Trong Queue `celery`

### Vi sao la queue `celery`

Repo khong khai bao `task_queues`, `task_routes`, `task_default_queue`, nen `apply_async(...)` se di vao queue mac dinh cua Celery la `celery`.

### Key Redis lien quan

- `celery`
  - Kieu: `LIST`
  - Chua cac message task chua duoc worker lay ra
- `_kombu.binding.celery`
  - Kieu: `SET`
  - Binding metadata do Kombu tao cho queue `celery`

### Message Decoded View

Moi phan tu trong list `celery` la 1 JSON string. Khi parse ra, message co dang tong quat nhu sau. Vi du ben duoi da rut gon cac field quan trong nhat:

```json
{
  "body": "[[\"<job_id>\",\"recruitment_screening\",{\"cv_text\":\"Alice, 3 years backend engineering. Skills: Python, FastAPI, SQL.\",\"job_description\":\"Looking for backend engineer with Python, FastAPI and SQL.\"}],{}, {\"callbacks\": null, \"errbacks\": null, \"chain\": null, \"chord\": null}]",
  "content-encoding": "utf-8",
  "content-type": "application/json",
  "headers": {
    "lang": "py",
    "task": "app.workers.worker.run_crew",
    "id": "<job_id>"
  },
  "properties": {
    "correlation_id": "<job_id>",
    "delivery_info": {
      "exchange": "celery",
      "routing_key": "celery"
    },
    "priority": 0
  }
}
```

### Y nghia field trong `body`

Phan `body` la JSON string cua protocol task message version 2 cua Celery. Khi decode them 1 lop nua, no co dang:

```json
[
  [
    "<job_id>",
    "recruitment_screening",
    {
      "cv_text": "Alice, 3 years backend engineering. Skills: Python, FastAPI, SQL.",
      "job_description": "Looking for backend engineer with Python, FastAPI and SQL."
    }
  ],
  {},
  {
    "callbacks": null,
    "errbacks": null,
    "chain": null,
    "chord": null
  }
]
```

Map vao code:

- arg 1: `job_id`
- arg 2: `crew_key`
- arg 3: `payload`

Day la ket qua truc tiep cua:

```python
run_crew.apply_async(args=[job_id, crew_key, payload], task_id=job_id)
```

## 2. In-Flight Keys Khi Worker Da Lay Message

Khi worker da reserve message nhung chua ack xong, Redis transport cua Kombu co the dung them cac key sau:

- `unacked`
  - Theo doi message dang xu ly
- `unacked_index`
  - Index de restore message neu qua `visibility_timeout`
- `unacked_mutex`
  - Khoa phuc vu restore

Gia tri mac dinh cua `visibility_timeout` la `3600` giay.

Dieu nay quan trong vi message co the roi khoi list `celery` nhung van chua xong; luc do no nam o nhom key `unacked`.

## 3. Result Backend Key `celery-task-meta-<job_id>`

Sau khi task hoan tat, Celery result backend se ghi 1 key dang:

```text
celery-task-meta-<job_id>
```

### Payload meta tong quat

Gia tri key nay la JSON cua task meta. Dang tong quat:

```json
{
  "status": "SUCCESS",
  "result": {
    "status": "done",
    "result": {
      "...": "..."
    },
    "crew_key": "recruitment_screening"
  },
  "traceback": null,
  "children": [],
  "date_done": "2026-04-22T...",
  "task_id": "<job_id>"
}
```

### Tai sao `result.result` bi long 2 lop

Worker return object nay:

```json
{
  "status": "done",
  "result": {
    "...": "..."
  },
  "crew_key": "recruitment_screening"
}
```

Nen Celery backend se boc them 1 lop task meta:

- `status = "SUCCESS"` la status cua Celery task
- `result = {...}` la return value cua ham `run_crew(...)`

## Khong Nam Trong Redis

Theo code hien tai, cac truong sau khong duoc repo nay luu business-native vao Redis:

- `input`
- `output`
- `crew_version`
- `status` kieu app-level trong `TaskRepository`

Ly do:

- `app/infrastructure/db/repository.py` dang la in-memory store `_store`, khong phai Redis
- Redis chi duoc Celery dung lam broker/backend

Noi cach khac:

- Payload request co di qua Redis trong broker message
- Return value cua worker co vao Redis trong result backend
- Nhung `TaskRepository` khong dang persist vao Redis

## Redis CLI De Soi Nhanh

### Xem queue message dang cho

```bash
redis-cli LRANGE celery 0 -1
```

### Xem binding cua queue default

```bash
redis-cli SMEMBERS _kombu.binding.celery
```

### Xem message dang duoc worker xu ly

```bash
redis-cli HGETALL unacked
redis-cli ZRANGE unacked_index 0 -1 WITHSCORES
```

### Xem result meta cua 1 job

```bash
redis-cli GET celery-task-meta-<job_id>
```

## Mapping Nhanh Tu Code Sang Redis

- `POST /api/v1/tasks/` hoac `POST /api/v1/jobs/`
  - enqueue 1 Celery task
- `JobService.submit_job(...)`
  - tao `job_id`
  - goi `run_crew.apply_async(...)`
- Redis list `celery`
  - chua broker message pending
- Worker `app.workers.worker.run_crew`
  - consume message tu queue
- Redis key `celery-task-meta-<job_id>`
  - chua task result meta

## References

Local files:

- `agentsmarket/src/ai-recruitment-system/app/infrastructure/queue/celery_app.py`
- `agentsmarket/src/ai-recruitment-system/app/application/services/job_service.py`
- `agentsmarket/src/ai-recruitment-system/app/workers/worker.py`
- `agentsmarket/src/ai-recruitment-system/app/infrastructure/db/repository.py`
- `agentsmarket/src/ai-recruitment-system/tests/run_api_contract_test.py`
- `agentsmarket/src/ai-recruitment-system/requirements.txt`

Primary docs used to derive the Redis layout:

- Celery message protocol v2: https://docs.celeryq.dev/en/latest/internals/protocol.html
- Celery default queue config: https://docs.celeryq.dev/en/v5.3.6/userguide/configuration.html
- Kombu virtual message structure: https://docs.celeryq.dev/projects/kombu/en/latest/_modules/kombu/transport/virtual/base.html
- Kombu Redis transport internals: https://docs.celeryq.dev/projects/kombu/en/stable/_modules/kombu/transport/redis.html
- Celery result backend meta format: https://docs.celeryq.dev/en/main/_modules/celery/backends/base.html
