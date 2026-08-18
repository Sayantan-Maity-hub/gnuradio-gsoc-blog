# From Form-Driven Runs to a Terminal-Controlled CortexLab Workflow

Over the last few weeks, I moved experiment submission out of the frontend and into a terminal-driven HTTP workflow. The controller now receives a JSON `POST` request, prepares a complete experiment package, runs it on CortexLab nodes, performs analysis after the node work is complete, and brings the final result back to the local machine.

The dashboard has changed with the workflow. It is no longer an experiment-submission form. It is now a focused status view for reservations, nodes, and experiment execution.

## Submitting an Experiment from the Terminal

The Flask controller exposes `POST /run-experiment` in `flask_server.py`. Its `create_experiment()` handler receives the request body and uses these values:

| Field | Purpose |
| --- | --- |
| `username` | CortexLab account used for the reservation |
| `hostname` | CortexLab gateway hostname |
| `reservation_name` | Name attached to the reservation |
| `walltime` | Requested reservation duration |
| `experiment` | Folder name under `experiment_manager/hil_experiments/` |
| `pr_id` | Identifier combined with the reservation job ID |
| `parameter` | Experiment-specific settings written to `parameters.json` |

`ci_workflow/experiment_request.sh` contains a PowerShell request example. It builds the body with `ConvertTo-Json` and sends it to the controller with `Invoke-RestMethod`:

```powershell
Invoke-RestMethod `
    -Uri "http://localhost:5678/run-experiment" `
    -Method POST `
    -ContentType "application/json" `
    -Body $body
```

On receipt, `create_experiment()` calls `reserve_nodes()` and starts `reservation_monitor()` when a reservation is needed. It then waits for the assigned node status through `wait_for_node_status()` before the experiment is allowed to start. This keeps experiment scheduling separate from the browser UI and makes the same request easy to trigger from CI.

## A Generic Experiment Runner

The orchestration entry point is `run_generic_experiment()` in `experiment_manager/generic_experiment_runner.py`. The runner treats an experiment as a directory with this shape:

```text
experiment_manager/hil_experiments/<experiment_name>/
|-- node_scripts/
|   |-- tx_chain.py
|   `-- rx_chain.py
`-- analysis.py
```

Every Python file in `node_scripts/` represents one required node. The runner discovers these scripts, counts them, selects the same number of available `ONLINE` and non-busy nodes, and maps one script to each selected node. It records that mapping in the execution registry with `create_experiment_registry()` from `cortexlab/execution/execution_registy.py`.

For example, `basic_hardware_test` contains `tx_chain.py` and `rx_chain.py`, so it requires two available nodes. The experiment ID is formed as `<pr_id>-<job_id>`, which ties the run to both the request and its CortexLab reservation.

## Building and Uploading the Run Folder

`create_experiment_folder()` in `experiment_manager/create_experiment_folder.py` creates the local run directory at:

```text
experiments/runs/<experiment_id>/
|-- parameters.json
|-- analysis.py
|-- <assigned-node-1>/
|   `-- <assigned-node-1-script>.py
`-- <assigned-node-2>/
    `-- <assigned-node-2-script>.py
```

The function copies each assigned node script into that node's folder, copies `analysis.py` into the experiment root, and writes the terminal-provided settings to `parameters.json`. It also records the local path in the execution registry.

Next, `upload_experiment_folder()` in `experiment_manager/upload_experiment_folder.py` transfers the complete generated directory through `cortexlab_Remote.upload_folder()`. The remote directory is tracked as `<job_id>/<experiment_id>`, so all node scripts, analysis code, and parameters are available together on CortexLab.

## Running Nodes, Then Analysis

`start_experiment()` in `experiment_manager/start_experiment.py` starts one thread per assigned node. Each invokes `execute_script()` in `cortexlab/execution/execution_monitor.py`.

Before executing, every node is checked, marked `PREPARING`, and made executable. The nodes then synchronize at a `threading.Barrier` and use the same future start time. This keeps multi-node work aligned instead of allowing the first available node to run significantly earlier than the others.

`finish_experiment_if_complete()` monitors the primary executions. Analysis begins only when every assigned node has finished successfully. If a node fails, the experiment is marked failed and analysis does not run.

When the primary work passes, `execute_analysis()` in `cortexlab/execution/execute_analysis.py` runs the root-level `analysis.py` on the remote experiment directory. The analysis script writes `results.json`; `execute_analysis()` downloads that file into `experiments/runs/<experiment_id>/results.json`, parses it, and updates the overall execution state to `FINISHED` with `PASS` or `FAILED`.

For `basic_hardware_test`, `experiment_manager/hil_experiments/basic_hardware_test/analysis.py` loads the captured IQ data, checks sample count, signal power, and frequency tolerance, and writes a structured result with a `passed` or `failed` status.

## A Dashboard for Observability

`templates/dashboard.html` now shows only operational status:

- Reservations from `GET /status/reservations`
- Nodes from `GET /status/nodes`
- Experiments from `GET /status/experiments`

The page refreshes these endpoints every two seconds. It displays reservation allocation and state, each node's online/busy/experiment status, and each experiment's state, assigned nodes, result, and start time. Removing the user-input section makes the boundary clear: terminals and CI submit work; the dashboard reports what the controller is doing.

## Result

The controller now follows one complete, repeatable lifecycle: terminal request, reservation, node assignment, generated run folder, remote upload, synchronized node execution, post-run analysis, and local `results.json` retrieval. New experiments can use the same pipeline by adding node scripts under `node_scripts/` and a root-level `analysis.py`.
