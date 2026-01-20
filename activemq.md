
## ActiveMQ / Broker addresses for SWF Panda workers

        This document describes the message addresses (topics and queues) used by the SWF Panda workers, transformer and related agents. It includes the address configuration, the intended message flows, and short example payloads.

        Notes:
        - The examples below use ActiveMQ/Artemis-style address/queue XML snippets. Adapt them to your broker configuration as needed.
        - When creating topic fan-out (a topic that routes to named queues) ensure the queue addresses are defined before the topic's multicast block; some brokers require the queue address to exist first.

---

### /topic/panda.workers — worker lifecycle events

        Purpose: control and scaling messages for Panda workers.

        Address XML (multicast topic):

        ```xml
        <address name="/topic/panda.workers">
          <multicast/>
        </address>
        ```

        Typical message flows:
        1. EIC -> `run_imminent` -> `idds.worker_handler` (request to create workers for a run)
        2. `idds.worker_handler` -> `adjust_worker` -> `harvester.adjust_worker` (start pilots / adjust worker count)
        3. `transformer` -> `heartbeat` -> `idds.worker_handler` (optional health/heartbeat)
        4. EIC -> `end_run` -> `idds.worker_handler` (signal run end)
        5. `idds.worker_handler` -> broadcast run-end to `/topic/panda.transformer`

        Example `run_imminent` payload (JSON):

        ```json
        {
          "instance": "prod",
          "msg_type": "run_imminent",
          "run_id": "2026-01-20-001",
          "created_at": "2026-01-20T12:00:00Z",
          "content": {
            "num_workers": 20,
            "num_cores_per_worker": 1,
            "num_ram_per_core": 2000.0,  
            "site": "T2_TEST"
          }
        }
        ```

        Example `adjuster_worker` payload (start/stop or change worker counts):

        ```json
        {
          "instance": "prod",
          "msg_type": "adjuster_worker",
          "run_id": "2026-01-20-001",
          "created_at": "2026-01-20T12:00:10Z",
          "content": {
            "num_workers": 25,
            "num_cores_per_worker": 1,
            "num_ram_per_core": 2000.0,
            "requested_at": "2026-01-20T12:00:00Z"
          }
        }
        ```

---

### /topic/panda.transformer — transformer control messages

        Purpose: messages targeted at the transformer component (e.g., stop requests and broadcasts).

        Address XML:

        ```xml
        <address name="/topic/panda.transformer">
          <multicast/>
        </address>
        ```

        Typical messages:
        - `slice` — work items intended for transformer processing (often routed through fan-out queues; see slices section).
        - `end_run` — instructs the transformer to enter stopping mode and exit once existing slices are processed.

---

### /topic/panda.slices and its fan-out queues

        Purpose: publish slice messages (work units) to a topic that fans out into two queues: one consumed by iDDS and one by the transformer.

        Important: define the queue addresses before the topic fan-out to ensure the queues are created and bound correctly.

        Queue addresses (anycast queues):

        ```xml
        <address name="/queue/panda.slices.idds">
          <anycast/>
        </address>

        <address name="/queue/panda.slices.transformer">
          <anycast/>
        </address>
        ```

        Topic fan-out (multicast with explicit queues):

        ```xml
        <address name="/topic/panda.slices">
          <multicast>
            <queue name="/queue/panda.slices.idds"/>
            <queue name="/queue/panda.slices.transformer"/>
          </multicast>
        </address>
        ```

        Flow:
        - EIC (or upstream) publishes `slice` messages to `/topic/panda.slices`.
        - The broker routes each message to both `/queue/panda.slices.idds` and `/queue/panda.slices.transformer`.
        - Consumers on those queues (idds workers and transformer) process messages independently.

        Example `slice` payload (JSON):

        ```json
        {
          "instance": "prod",
          "msg_type": "slice",
          "run_id": "2026-01-20-001",
          "created_at": "2026-01-20T12:01:00Z",
          "content": {
            "slice_id": "slice-0001",
            "filename": "swf.20260120.120100.slice-0001.stf",
            "start": "20260120120000",
            "end": "20260120120100",
            "payload": { /* task-specific data */ }
          }
        }
        ```

---

### /topic/panda.results and its fan-out queues

        Purpose: publish processing results which should be routed to different backends (e.g., iDDS and a fast-processing path).

        Queue addresses:

        ```xml
        <address name="/queue/panda.results.idds">
          <anycast/>
        </address>

        <address name="/queue/panda.results.fastprocessing">
          <anycast/>
        </address>
        ```

        Topic fan-out:

        ```xml
        <address name="/topic/panda.results">
          <multicast>
            <queue name="/queue/panda.results.idds" />
            <queue name="/queue/panda.results.fastprocessing" />
          </multicast>
        </address>
        ```

        Result message example:

        ```json
        {
          "instance": "prod",
          "msg_type": "slice_result",
          "run_id": "2026-01-20-001",
          "created_at": "2026-01-20T12:02:00Z",
          "content": {
            "slice_id": "slice-0001",
            "requested_at": "2026-01-20T12:01:00Z",
            "processing_start_at": "2026-01-20T12:01:05Z",
            "processed_at": "2026-01-20T12:01:45Z",
            "result": { /* any result metadata */ }
          }
        }
        ```
---

### Message format & schema (reference)

        The canonical message contract used across iDDS components is defined in the iDDS `prompt.md` (message contract and examples). See:

        https://github.com/wguanicedew/iDDS/blob/dev/main/prompt.md

        Key points (short summary):

        - All messages MUST include `instance`, `msg_type`, and `run_id` (and typically `created_at`). Use `instance` to identify the deployment (e.g., `prod`, `dev_<userid>`), `msg_type` to indicate purpose, and `run_id` to scope messages.
        - The message body is commonly placed under `content` and is message-specific (e.g., `num_workers` for `run_imminent`, timing fields for `slice_result`).
        - Use broker-side selectors to filter messages before delivery. Example STOMP selector header:

        ```python
        headers['selector'] = "instance='prod' AND run_id='12345'"
        ```

        - Timing fields for results (inside `content`) include `requested_at`, `processing_start_at`, and `processed_at` to support latency measurement.

        Suggested header example (Python dict):

        ```python
        headers = {
          'persistent': 'true',
          'ttl': timetolive_ms,    # e.g. 12 * 3600 * 1000
          'vo': 'eic',
          'instance': instance_name,
          'msg_type': message_type,
          'run_id': run_id,
        }
        ```

        Refer to the full `prompt.md` for more examples (run_imminent/start, slice, slice_result, transformer lifecycle messages, adjuster messages and recommended consumer subscription patterns).

