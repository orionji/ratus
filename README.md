# Ratus

[![Go](https://github.com/hyperonym/ratus/actions/workflows/go.yml/badge.svg)](https://github.com/hyperonym/ratus/actions/workflows/go.yml)
[![codecov](https://codecov.io/gh/hyperonym/ratus/branch/master/graph/badge.svg?token=6HJKAQ9XR1)](https://codecov.io/gh/hyperonym/ratus)
[![Go Reference](https://pkg.go.dev/badge/github.com/hyperonym/ratus.svg)](https://pkg.go.dev/github.com/hyperonym/ratus)
[![Swagger Validator](https://img.shields.io/swagger/valid/3.0?specUrl=https%3A%2F%2Fraw.githubusercontent.com%2Fhyperonym%2Fratus%2Fmaster%2Fdocs%2Fswagger.json)](https://hyperonym.github.io/ratus/)
[![Go Report Card](https://goreportcard.com/badge/github.com/hyperonym/ratus)](https://goreportcard.com/report/github.com/hyperonym/ratus)
[![Status](https://img.shields.io/badge/status-alpha-yellow)](https://github.com/hyperonym/ratus)

Ratus is a RESTful asynchronous task queue server. It translated concepts of distributed task queues into a set of resources that conform to [REST principles](https://en.wikipedia.org/wiki/Representational_state_transfer) and provides an easy-to-use [HTTP API](https://hyperonym.github.io/ratus/).

The key features of Ratus are:

* Guaranteed at-least-once execution of tasks.
* Automatic recovery of timed out tasks.
* Simple language agnostic RESTful API.
* Time-based task scheduling.
* Naturally load balanced across consumers.
* Support dynamic topology changes.
* Scaling through replication and partitioning.
* Pluggable storage engine architecture.
* Prometheus integration for observability.

## Concepts

### Data Model

* **[Task](https://pkg.go.dev/github.com/hyperonym/ratus#Task)** references an idempotent unit of work that should be executed asynchronously.
* **[Topic](https://pkg.go.dev/github.com/hyperonym/ratus#Topic)** refers to an ordered subset of tasks with the same topic name property.
* **[Promise](https://pkg.go.dev/github.com/hyperonym/ratus#Promise)** represents a claim on the ownership of an active task.
* **[Commit](https://pkg.go.dev/github.com/hyperonym/ratus#Commit)** contains a set of updates to be applied to a task.

### Workflow

* **Producer** client pushes **tasks** with their desired date-of-execution (scheduled times) to a **topic**.
* **Consumer** client makes a **promise** to execute a **task** polled from a **topic** and acknowledges with a **commit** upon completion.

### Topology

* Both **producer** and **consumer** clients can have multiple instances running simultaneously.
* **Consumer** instances can be added dynamically to increase throughput, and **tasks** will be naturally load balanced among consumers.
* **Consumer** instances can be removed (or crash) at any time without risking to lose the task being executing: a **task** that has not received a **commit** after the **promised** deadline will be picked up and executed again by other consumers.

### Task States

* **pending** (0): The task is ready to be executed or is waiting to be executed in the future.
* **active** (1): The task is being processed by a consumer. Active tasks that have timed out will be automatically reset to the `pending` state. Consumer code should handle failure and set the state to `pending` to retry later if necessary.
* **completed** (2): The task has completed its execution. If the storage engine implementation supports TTL, completed tasks will be automatically deleted after the retention period has expired.
* **archived** (3): The task is stored as an archive. Archived tasks will never be deleted due to expiration.

### Behavior

* **Task IDs across all topics share the same namespace** ([ADR](https://github.com/hyperonym/ratus/blob/master/docs/ARCHITECTURAL_DECISION_RECORDS.md#task-ids-should-be-unique-across-all-topics)). Topics are simply subsets generated based on the `topic` properties of the tasks, so topics do not need to be created explicitly.
* Ratus is a task scheduler when consumers can keep up with the task generation speed, or a priority queue when consumers cannot keep up with the task generation speed.
* Tasks will not be executed until the scheduled time arrives. After the scheduled time, excessive tasks will be executed in the order of the scheduled time.

## Engines

Ratus provides a consistent API for various storage engine implementations, allowing users to choose a specific engine based on their needs without having to modify client-side code.

| Name | Persistent | Replication | Partitioning | TTL
| --- | :---: | :---: | :---: | :---: |
| `mongodb` | ● | ● | ● | ● |

### MongoDB

Ratus works best with **MongoDB version ~4.4**. MongoDB 5.0+ is also supported but requires additional considerations, see [Implementation Details](https://github.com/hyperonym/ratus/blob/master/README.md#implementation-details) to learn more.

> 💭 **TL;DR** set `MONGODB_DISABLE_ATOMIC_POLL=true` when using Ratus with MongoDB 5.0+.

#### Replication

When using the MongoDB storage engine, the Ratus instance itself is stateless. For high availability, **start multiple instances of Ratus and connect them to the same MongoDB replica set**.

All Ratus instances should run behind load balancers configured with health checks. **Producer and consumer clients should connect to the load balancer**, not directly to the instances.

#### Partitioning

Horizontal scaling could be achieved through sharding the task collection. However, with the help of the TTL mechanism, **partitioning is not necessary in most cases**. The best performance and the strongest atomicity can only be obtained without sharding.

If the amount of data exceeds the capacity of a single node or replica set, choose from the following sharding options:

* If there is a large number of topics, **use a hashed index on the `topic` field as the shard key**, this will also enable the best polling performance on a sharded cluster.
* If there is a huge amount of tasks in a few topics, **use a hashed index on the `_id` field as the shard key**, this will also result in a more balanced data distribution.

#### Implementation Details

* When using the MongoDB storage engine, **tasks across all topics are stored in the same collection**.
* Task is the only concrete data model in the MongoDB storage engine, while topics and promises are just conceptual entities for enforcing the RESTful design principles.
* Since the resolution of the scheduled time in MongoDB is in millisecond level and is affected by the instance's own clock, **the order in which consumers receive tasks is not strictly guaranteed**.
* TTL cannot be disabled for `completed` tasks, in order to preserve a task forever, set it to the `archived` state.
* It is not recommended to upsert tasks on sharded collections using the `topic` field as the shard key. Due to MongoDB's own [limitations](https://www.mongodb.com/docs/v4.4/reference/method/db.collection.replaceOne/#shard-key-modification), atomic operations cannot be used in this case, and only a fallback scheme equivalent to delete before insert can be used, so atomicity and performance cannot be guaranteed. This problem can be circumvented by using simple inserts in conjunction with fine-tuned TTL settings.
* By default, polling is implemented through `findAndModify`. In the event of a conflict, MongoDB's native [optimistic concurrency control](https://www.mongodb.com/docs/v4.4/faq/concurrency/#how-granular-are-locks-in-mongodb-) (OCC) will transparently retry the operation. But in MongoDB 5.0 and above, the retry will report a `WriteConflict` error in the database server's log (although the operation is still successful from the client's perspective). You can choose to ignore this error, or circumvent the problem by **setting `MONGODB_DISABLE_ATOMIC_POLL=true` when using MongoDB 5.0+**. This option will make Ratus to not use `findAndModify` for polling and instead rely on the application-level OCC layer to ensure atomicity.

#### Index Models

| Key Patterns | Partial Filter Expression | TTL |
| --- | --- | --- |
| `{"topic": "hashed"}` | - | - |
| `{"topic": 1, "scheduled": 1}` | `{"state": 0}` | - |
| `{"deadline": 1}` | `{"state": 1}` | - |
| `{"topic": 1}` | `{"state": 1}` | - |
| `{"consumed": 1}` | `{"state": 2}` | `MONGODB_RETENTION_PERIOD` |

## Observability

### Metrics and Labels

Ratus exposes the following [Prometheus](https://prometheus.io) metrics on the `/metrics` endpoint:

| Name | Type | Labels |
| --- | --- | --- |
| **ratus_request_duration_seconds** | histogram | `topic`, `method`, `endpoint`, `status_code` |
| **ratus_chore_duration_seconds** | histogram | - |
| **ratus_task_schedule_delay_seconds** | gauge | `topic`, `producer`, `consumer` |
| **ratus_task_execution_duration_seconds** | gauge | `topic`, `producer`, `consumer` |
| **ratus_task_produced_count_total** | counter | `topic`, `producer` |
| **ratus_task_consumed_count_total** | counter | `topic`, `producer`, `consumer` |
| **ratus_task_committed_count_total** | counter | `topic`, `producer`, `consumer` |

### Liveness and Readiness

Ratus supports [liveness and readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) via HTTP GET requests:

* The `/livez` endpoint returns a status code of **200** if the instance is running.
* The `/readyz` endpoint returns a status code of **200** if the instance is ready to accept traffic.

## Caveats

* 🚨 **Topic names and task IDs must not contain plus signs ('+') due to [gin-gonic/gin#2633](https://github.com/gin-gonic/gin/issues/2633).**
* It is not recommended to use Ratus as the main storage of tasks. Instead, consider storing the complete task record in a database, and **use a minimal descriptor as the payload for Ratus.**
* The `completed` state only indicates that the task has been executed, it does not mean the task was successful.
* Ratus is a simple and reliable alternative to task queues like [Celery](https://docs.celeryq.dev/). Consider to use [RabbitMQ](https://www.rabbitmq.com/) or [Kafka](https://kafka.apache.org/) if you need high-throughput message passing without task management.

## Frequently Asked Questions

For more details, see [Architectural Decision Records](https://github.com/hyperonym/ratus/blob/master/docs/ARCHITECTURAL_DECISION_RECORDS.md).

### Why HTTP API?

> Asynchronous task queues are typically used for long background tasks, so the overhead of the HTTP API is not significant compared to the time spent by the tasks themselves. On the other hand, the HTTP-based RESTful API can be easily accessed by all languages without using dedicated client libraries.

### How to poll from multiple topics?

> If the number of topics is limited and you don't care about the priority between them, you can choose to create multiple threads/goroutines to listen to them simultaneously. Alternatively, you can create a ***topic of topics*** to get the topic names in turn and then get the next task from the corresponding topic.

## Contributing

This project is open-source. If you have any ideas or questions, please feel free to reach out by creating an issue!

Contributions are greatly appreciated, please refer to [CONTRIBUTING.md](https://github.com/hyperonym/ratus/blob/master/CONTRIBUTING.md) for more information.

## License

Ratus is available under the [Mozilla Public License Version 2.0](https://github.com/hyperonym/ratus/blob/master/LICENSE).

---

© 2022 [Hyperonym](https://hyperonym.org)
