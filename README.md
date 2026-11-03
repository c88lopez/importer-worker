# importer-worker

> **Moved.** This service now lives in [c88lopez/importer](https://github.com/c88lopez/importer), under `cmd/worker`, together with importer-api. This repo's history was imported there; it's archived and no longer updated.

Processes the events [importer-api](../importer-api) publishes to RabbitMQ. For each `item.created` it marks the item `done` in the shared Postgres database (`status`, `processed_at`).

```
importer-api ─► exchange "importer" (topic) ─► item.created ─► queue importer-worker.item-created ─► importer-worker
                                                                   │ rejected                           │
                                                                   ▼                                     ▼
                                                    importer-worker.item-created.dlq     UPDATE items SET status = 'done'
```

The API writes events to an outbox table in the same transaction as the item, then publishes them (see importer-api's README, "Events and the outbox"). Delivery is **at least once**, so the same event can arrive twice.

## How a message is handled

The worker declares its queues on every connect: a [quorum queue](https://www.rabbitmq.com/docs/quorum-queues) bound to `item.created`, and a dead-letter queue (DLQ) for messages it gives up on. It takes one message at a time, with up to 10 more prefetched, and acks each only after the database update commits. If a pod dies mid-message, RabbitMQ hands the message to another pod.

| What happens | Result |
| --- | --- |
| Item is pending | Marked `done`, message acked |
| Item already done, or deleted | Nothing to do, message acked. This makes duplicates harmless. |
| Malformed body or unknown type | Rejected straight to the DLQ: retrying can't fix it |
| Any other error (e.g. database down) | Waits 1s, 2s, 4s… (up to 30s), then requeues. After 10 attempts, rejected to the DLQ. |
| Delivered again after the 10th attempt (the pod died processing it) | Rejected to the DLQ without trying again |

The worker counts attempts itself, using the `x-acquired-count` header that quorum queues set on redeliveries. RabbitMQ's own `x-delivery-limit` doesn't help here: in RabbitMQ 4, a requeue with AMQP 0.9.1 `basic.nack` doesn't count toward it, so a failing message would be retried forever. `TestConsumerGivesUpAfterMaxDeliveries` checks this.

Messages in the DLQ wait for a human. Inspect them in the management UI (Queues > `importer-worker.item-created.dlq` > Get messages). To retry them once the cause is fixed, use a [shovel](https://www.rabbitmq.com/docs/shovel) back to the `importer` exchange, or the management UI's "Move messages".

Queue arguments are fixed when a queue is created. Changing one in code makes the declare fail with `PRECONDITION_FAILED`, so delete the queue first, or set it with a policy instead.

## The shared database

importer-api owns the `items` table and its migrations. This worker only updates `status` and `processed_at`, which importer-api's migration `00002` adds. Deploy that migration before the worker. `itemsContract` in `process_test.go` is the schema the tests assume. If importer-api changes those columns, update it there too.

## Run locally

`make run` and `make test` use the cluster's Postgres and RabbitMQ through `kubectl port-forward`, with credentials read from their Secrets, like importer-api.

| | Postgres | RabbitMQ |
| --- | --- | --- |
| `make run` | `app`, the real database | vhost `/`: the real queue, shared with the cluster's workers |
| `make test` | `importer_worker_test` | vhost `importer-test` |

```sh
make dev-setup      # once: create importer_worker_test and the importer-test vhost
make tunnel         # in its own terminal (or importer-api's: run one of them, not both)
make run            # probes and metrics on :9465
make test
```

A worker running on your machine takes its share of the real queue, alongside the cluster's pods. Its logs show which items it processed.

CI can't reach the cluster, so it runs the same tests against throwaway Postgres and RabbitMQ containers.

## Probes, metrics and tracing

There's no API, so one port (`PORT`, default 9464) serves:

| Path | What |
| --- | --- |
| `/healthz` | Liveness: the process answers. Outages don't fail it, because the consumer reconnects on its own. |
| `/readyz` | Readiness: consuming from RabbitMQ and Postgres answers. A rollout waits for new pods to be ready, so a worker that can't consume never replaces one that can. |
| `/metrics` | Prometheus, scraped via `k8s/40-podmonitor.yaml` |

| Metric | What |
| --- | --- |
| `messaging_process_duration_seconds` | Messages processed, by `outcome` (`done`, `requeued`, `dead_lettered`) |
| `db_client_operation_duration_seconds`, `pgxpool_*` | Database calls and the connection pool |

Queue depth comes from RabbitMQ's own metrics (its ServiceMonitor in `default`).

Each message gets a `process importer-worker.item-created` span, which continues the trace from the message's `traceparent` header. In Tempo, a `POST /api/items` trace shows the request, the API's `send importer`, the worker's `process`, and the worker's `UPDATE`.

## Deploy

```sh
make secret                 # once: DATABASE_URL and RABBITMQ_URL from the cluster's Secrets
make deploy
make status
make logs
```

Releases work like importer-api's: `make release VERSION=x.y.z` tags the commit, CI builds and pushes `ghcr.io/c88lopez/importer-worker:x.y.z`, and you bump the tag in `k8s/20-deployment.yaml` and the Makefile.

On shutdown (SIGTERM), the worker stops taking messages and finishes the one in hand, with up to 30s for it. Prefetched messages go back to the queue.
