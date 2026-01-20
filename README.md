# swf-panda-workers

Lightweight agents that manage the lifecycle of Panda workers and the transformer pipeline via message topics.

## Overview

This repository contains components that listen for run lifecycle messages and coordinate the creation, scaling, and shutdown of Panda workers. The agents interact with messaging topics to: start runs, adjust worker counts, submit iDDS/Panda tasks, and notify the transformer to stop when a run finishes.

## Architecture (high level)

- A controller listens to /topic/panda.workers for run lifecycle messages.
- On receiving a `run_imminent` message the controller ensures the desired number of Panda workers are created.
- The controller submits an iDDS workflow and a Panda task; Panda jobs produced by the task are consumed by pilots/workers which run the transformer from `BNLNPPS/swf-transform`.
- The controller publishes `adjust_worker` messages to /topic/panda.workers to request scaling. Messages include a `msg_type` filter so agents can ignore self-published events if needed.
- The transformer component (`BNLNPPS/swf-transform`) listens on /topic/panda.transformer for `end_run` messages; when received it transitions to a stopping mode and exits once no more `slice` messages remain.

## Topics & message types

- `/topic/panda.workers`
	- `run_imminent` — request to create workers for a new run. Typical payloads include `target_worker_count` and may include `core_count`, `memory_per_core`, and `site`.
	- `adjust_worker` — request to increase/decrease workers. Agents should use message metadata (e.g., `msg_type`) to filter their own messages if necessary.

- `/topic/panda.transformer`
	- `slice` — payloads consumed by the transformer pipeline for processing.
	- `end_run` — signals that the run should stop; the transformer will finish processing existing slices then exit.

## License

This project is licensed under the terms in the `LICENSE` file.

## Contact

Repository: BNLNPPS/swf-panda-workers
For questions or to request features, please open an issue.
