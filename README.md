# Ratus

[![Go](https://github.com/hyperonym/ratus/actions/workflows/go.yml/badge.svg)](https://github.com/hyperonym/ratus/actions/workflows/go.yml)
[![codecov](https://codecov.io/gh/hyperonym/ratus/branch/master/graph/badge.svg?token=6HJKAQ9XR1)](https://codecov.io/gh/hyperonym/ratus)
[![Go Reference](https://pkg.go.dev/badge/github.com/hyperonym/ratus.svg)](https://pkg.go.dev/github.com/hyperonym/ratus)
[![Swagger Validator](https://img.shields.io/swagger/valid/3.0?specUrl=https%3A%2F%2Fraw.githubusercontent.com%2Fhyperonym%2Fratus%2Fmaster%2Fdocs%2Fswagger.json)](https://hyperonym.github.io/ratus/)
[![Go Report Card](https://goreportcard.com/badge/github.com/hyperonym/ratus)](https://goreportcard.com/report/github.com/hyperonym/ratus)
[![Status](https://img.shields.io/badge/status-alpha-yellow)](https://github.com/hyperonym/ratus)

Ratus is a RESTful asynchronous task queue service. It translated concepts of distributed task queues into a set of resources that conform to [REST principles](https://en.wikipedia.org/wiki/Representational_state_transfer) and provides an easy-to-use HTTP API.

## Observability

### Metrics and Labels

Ratus exposes [Prometheus](https://prometheus.io) metrics via HTTP, on the `/metrics` endpoint.

| Name | Type | Labels |
| --- | --- | --- |
| **ratus_request_duration_seconds** | histogram | `topic`, `endpoint`, `status_code` |
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

## Contributing

This project is open-source. If you have any ideas or questions, please feel free to reach out by creating an issue!

Contributions are greatly appreciated, please refer to [CONTRIBUTING.md](https://github.com/hyperonym/ratus/blob/master/CONTRIBUTING.md) for more information.

## License

Ratus is available under the [Mozilla Public License Version 2.0](https://github.com/hyperonym/ratus/blob/master/LICENSE).

---

© 2022 [Hyperonym](https://hyperonym.org)
