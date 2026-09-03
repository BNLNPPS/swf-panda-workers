# CLAUDE.md — Project guide for Claude Code

## Project overview

**swf-panda-workers** is a transceiver agent for EIC workflow management. It bridges run lifecycle messages (from the EIC control system) with iDDS/PanDA worker creation and scaling.

Package entrypoint: `swf_panda_workers.transceiver:main` → CLI command `swf-panda-workers`.

## Repository layout

```
config/
  swf_panda_workers.yaml          # Runtime configuration (all tunables)
lib/swf_panda_workers/
  transceiver.py                  # Transceiver agent: main loop, broker wiring, on_message dispatch
  brokers/
    activemq.py                   # Publisher / Subscriber wrappers (STOMP over ActiveMQ)
  prompt/handlers/
    workerhandler.py              # Message handlers + publish helpers (create/adjust/close)
    panda.py                      # PandaClient: iDDS REST API wrappers
  utils/
    cache.py                      # PersistentTTLCache: SQLite-backed TTL cache
pyproject.toml                    # Build config; install with `pip install -e .`
```

## Key architecture decisions

### Transceiver mode (`transceiver.mode` in yaml)

Two mutually exclusive modes — anything else raises `ValueError` at startup:

| Mode | Outbound iDDS calls |
|------|-------------------|
| `message` | Publishes STOMP messages to `/topic/panda.workers` |
| `rest` | Calls iDDS REST API directly via `PandaClient` |

Mode is validated once in `Transceiver.__init__` and again defensively in `worker_handler`. `panda_client` is always instantiated (regardless of mode) and passed through `handler_kwargs` — besides gating iDDS calls in `rest` mode, it also issues `add_target_slots` calls (see below), which are a direct PanDA server REST call independent of the mode setting.

### Message flow

```
run_imminent            → create_workflow_task to /topic/panda.workers  (message mode)
                          or PandaClient.idds_create_workflow_task       (rest mode)
created_workflow_task   → cache run_id → {request_id, transform_id, workload_id}
slice_result            → adjust_worker to /topic/panda.workers          (message mode)
                          or PandaClient.idds_adjust_worker               (rest mode)
run_end/run_stop/end_run → stop_transformer broadcast to /topic/panda.transformer
                         + close_workflow_task to /topic/panda.workers   (message mode)
                           or PandaClient.idds_close_workflow_task        (rest mode)
```

### Caches (PersistentTTLCache, 3-day TTL, in `Transceiver`)

Both run-level caches are backed by SQLite (`cache.path` in yaml, default `~/.cache/swf_panda_workers/cache.db`) and survive process restarts.

- `run_to_idds_ids_cache`: `"<namespace>:<run_id>" → {run_id, request_id, transform_id, workload_id}`
- `run_to_core_count_cache`: `"<namespace>:<run_id>" → current core_count` (seeded from `run_imminent`, updated by `slice_result` scaling)
- `site_to_core_count_cache`: `site → {initial_core_count, current_core_count, changed}` — keyed by physical PanDA site only (not namespace-scoped), since target slots are a shared PanDA-server resource. Seeded from `run_imminent_worker`, updated by `slice_result` scaling. Whenever the `current_core_count` for a site changes, `PandaClient.add_target_slots(site, core_count)` is called to update the target slot count on the PanDA server.

Run-level cache keys are namespace-scoped (`Transceiver.cache_key` / `workerhandler._cache_key`) because a `Transceiver` with `namespace: all` (the default) processes messages from every namespace, so identical `run_id`s from different namespaces (e.g. `prod` vs `dev_alice`) would otherwise collide. The namespace used for the key comes from the STOMP header (where `Publisher.publish()` actually injects it), falling back to the message body and then to the `Transceiver`'s own configured namespace.

### Namespace (`transceiver.namespace` in yaml, `NAMESPACE_ALL` in `brokers/activemq.py`)

`NAMESPACE_ALL = "all"` is the sentinel default (also the fallback used by `utils/config.py` and `Transceiver.__init__`). A `Transceiver` configured with a real namespace (e.g. `"prod"`) filters inbound messages via the STOMP subscription selector (`Subscriber._build_selector`) and tags outbound messages with that namespace (`Publisher.publish`). `NAMESPACE_ALL` disables both: no selector clause is added and no `namespace` header is injected, so the agent sends/receives across every namespace.

### Worker scaling (`slice_result`)

Configured via `slice.processing_time` (default 30 s):
- actual > 1.5 × threshold → scale core_count × 1.5
- actual > 1.2 × threshold → scale core_count × 1.2
- otherwise → no action

## Topics & message types

| Topic / Queue | Direction | Message types |
|---|---|---|
| `/topic/panda.workers` | inbound | `run_imminent`, `created_workflow_task`, `run_end`, `run_stop`, `end_run`, `transformer_heartbeat` |
| `/topic/panda.workers` | outbound (message mode) | `create_workflow_task`, `adjust_worker`, `close_workflow_task` |
| `/topic/panda.transformer` | outbound | `stop_transformer` (broadcast) |
| `/queue/panda.results.worker` | inbound | `slice_result` |

## handler_kwargs keys

Passed from `Transceiver._run_worker` into every `worker_handler` call:

| Key | Type | Description |
|---|---|---|
| `namespace` | `str\|None` | The `Transceiver`'s own configured namespace; used by `workerhandler._cache_key` as the fallback when a message carries no namespace of its own |
| `transformer_broadcaster` | `Publisher\|None` | Broadcasts `stop_transformer` |
| `panda_workers_publisher` | `Publisher\|None` | Publishes to `/topic/panda.workers` (message mode) |
| `panda_attributes` | `dict` | PanDA task params forwarded from `panda:` config section |
| `timetolive` | `int` | STOMP message TTL in ms |
| `slice_config` | `dict` | `{processing_time: <seconds>}` |
| `core_count_cache` | `TTLCache` | Shared mutable cache; `handle_slice_result` updates it in-place |
| `site_core_count_cache` | `TTLCache` | `site → core_count` cache; updated by `handle_slice_result`, triggers `add_target_slots` on change |
| `mode` | `str` | `"message"` or `"rest"` |
| `panda_client` | `PandaClient` | Always set; used for iDDS REST calls in `rest` mode and for `add_target_slots` in either mode |

## PandaClient (panda.py)

iDDS server URL resolved from (in order): `$IDDS_SERVER` env var → `panda.idds_server` in `panda.cfg`.

Client obtained via:
```python
import pandaclient.idds_api as idds_api
import idds.common.utils as idds_utils
client = idds_api.get_api(idds_utils.json_dumps, idds_host=..., compress=True, manager=True)
```

Methods mirror the three outbound message types:
- `idds_create_workflow_task(run_id, content)`
- `idds_adjust_worker(run_id, idds_ids, content)`
- `idds_close_workflow_task(run_id, idds_ids)`

`add_target_slots(panda_queue, slots, global_share=None, resource_type=None, expiration_date=None)` calls the PanDA server's `POST /v1/harvester/add_target_slots` directly via `pandaclient.Client.http_request_decorator` (not iDDS). Called from `Transceiver._dispatch` (`run_imminent_worker`) and `handle_slice_result` whenever the cached `current_core_count` for a site changes.

## Logging convention

All functions in `workerhandler.py` accept `logger=None` and fall back to the module-level `_logger = logging.getLogger(__name__)` via `logger = logger or _logger` at the top of each function body.

## Configuration file search order

1. `--config` CLI argument
2. `$SWF_PANDA_WORKERS_CONFIG` env var
3. `~/.config/swf_panda_workers/swf_panda_workers.yaml`
4. `$PREFIX/etc/swf_panda_workers/swf_panda_workers.yaml`
5. `<repo>/config/swf_panda_workers.yaml`

`${VAR}` placeholders in the yaml are expanded via `os.path.expandvars`.

## Install & run

```bash
pip install -e .
swf-panda-workers --config config/swf_panda_workers.yaml --log-level DEBUG
```
