# GSoC Final Report: GNU Radio Hardware-in-the-Loop Controller for CortexLab

**Contributor:** [Sayantan Maity]

**EMail:** [maitysayantan116@gmail.com]

**Organization:** [GNU Radio]   
**Mentor(s):** [Marcus Müller, Cyrille Morin]  
**Project period:** June-August 2026  
**Repository:** [GitHub Link](https://github.com/Sayantan-Maity-hub/gnuradio-hardware-in-loop)

## Abstract

This project delivers a Python-based controller for repeatable GNU Radio hardware-in-the-loop (HIL) experiments on CortexLab. The controller turns a multi-step manual workflow—reserving radio nodes, starting a CortexLab MINUS task, staging experiment files, coordinating transmitter and receiver programs, and evaluating results—into a single HTTP request with observable status throughout the run.

The implementation provides a Flask API and web dashboard, OAR reservation and MINUS task orchestration, gateway-mediated SSH/SFTP deployment, synchronized multi-node execution, experiment/result registries, and two reusable GNU Radio experiment examples: a basic over-the-air tone test and an OFDM payload test. It also includes an early container-image build path for testing GNU Radio revisions. The project establishes the controller and experiment interface needed for eventual physical-hardware CI, while clearly leaving full CI integration, durable state, authentication, and additional end-to-end validation as future work.

## Motivation

GNU Radio software that works in local simulation still needs validation against real SDR hardware. RF behavior depends on radio drivers, timing, gain and frequency settings, the physical RF environment, and coordination between distributed transmitter and receiver nodes. CortexLab offers remotely reservable SDR nodes, but a manual test requires several systems to be operated in the correct order.

The goal was therefore to provide a small, reusable HIL control plane that can run an experiment from a structured request and expose its progress to a developer or, later, to CI. The intended workflow is:

1. Reserve compatible CortexLab nodes through OAR.
2. Create a MINUS task using a GNU Radio-capable container image.
3. Determine which allocated nodes are online and available.
4. Create a per-run workspace, upload scripts and parameters, and launch the node programs together.
5. Collect a machine-readable result from an analysis program.

## Completed Work

### Controller API and dashboard

I implemented a Flask controller that starts locally on port `5678`. Its main `POST /run-experiment` endpoint accepts reservation data, experiment selection, pull-request metadata, and parameters. It either creates or reuses a reservation, waits for the reserved nodes to become available, then delegates the run to the generic experiment runner. The project also exposes status endpoints for reservations, nodes, all experiments, and an individual experiment; the browser dashboard refreshes this information every two seconds.

This makes the state of a run inspectable rather than leaving users to infer progress from remote shell sessions. The execution identifier follows `<pr-id>-<commit-prefix>-<oar-job-id>`, enabling a hardware run to be traced back to the revision that requested it.

Relevant implementation: [`flask_server.py`](flask_server.py), [`templates/dashboard.html`](templates/dashboard.html), and [`experiment_manager/generic_experiment_runner.py`](experiment_manager/generic_experiment_runner.py).

### Reservation, task, and node lifecycle

The CortexLab integration submits and monitors OAR reservations and creates MINUS scenarios/tasks for them. A background reservation monitor queries `oarstat -fj`, records reservation state, derives scheduled wait time, and parses assigned CortexLab hostnames. Node monitoring updates the in-memory registry with online/offline state and association with an experiment.

The generic runner selects only online, non-busy nodes, verifies that enough nodes exist for the requested experiment, and marks selected nodes busy before staging. It releases them after primary node execution is complete. This avoids assigning a node already in use by another tracked run.

Relevant implementation: [`cortexlab/reservation/`](cortexlab/reservation/), [`cortexlab/nodes/`](cortexlab/nodes/), and [`cortexlab/remote_connections/`](cortexlab/remote_connections/).

### Reusable experiment packaging and remote deployment

An experiment is defined as a directory containing `node_scripts/`, `analysis.py`, and an example parameter file. Every Python script in `node_scripts/` is mapped to one available radio node. The controller creates an isolated local folder for each run, copies the selected scripts and analysis program into it, writes `parameters.json`, and uploads the workspace to the CortexLab reservation area over SSH/SFTP.

This layout separates controller behavior from experiment-specific GNU Radio programs. Adding a new experiment does not require changing the orchestration path: the experiment author provides node scripts, parameters, and analysis logic following the same contract.

Relevant implementation: [`experiment_manager/create_experiment_folder.py`](experiment_manager/create_experiment_folder.py), [`experiment_manager/upload_experiment_folder.py`](experiment_manager/upload_experiment_folder.py), and [`experiment_manager/hil_experiments/`](experiment_manager/hil_experiments/).

### Synchronized distributed execution and result handling

The controller launches one thread per node script. A `threading.Barrier` and a shared future start time coordinate scripts so that the transmitter and receiver begin together rather than serially. Execution state, start/end timestamps, standard output/error, and results are recorded per node and at experiment level.

After every primary node finishes successfully, the controller runs `analysis.py`. Analysis is expected to produce a JSON result with `status`, `reason`, and `metrics`; a result whose status is `passed` is treated as a successful experiment. Failures from node execution, analysis, malformed result output, or insufficient nodes are recorded in the experiment registry.

Relevant implementation: [`experiment_manager/start_experiment.py`](experiment_manager/start_experiment.py), [`cortexlab/execution/`](cortexlab/execution/), and [`cortexlab/execution/execute_analysis.py`](cortexlab/execution/execute_analysis.py).

### HIL experiment implementations

Two example experiment families are included.

- **Basic hardware test:** a transmitter and receiver run a tone-based SDR test. The analysis reads captured complex IQ samples, checks received sample count, finite samples, average signal power, and dominant-frequency error. It writes metrics such as RMS amplitude, frequency resolution, tolerance, and pass/fail checks to `results.json`.
- **OFDM hardware test:** GNU Radio transmitter and receiver flowgraphs plus parameterized Python scripts exercise an OFDM payload path. The analysis compares the received binary payload with the expected UTF-8 message and reports byte-level diagnostics and a machine-readable verdict.

The accompanying `.grc` flowgraphs, node scripts, analysis scripts, and `parameter.json` files make these examples useful both as smoke tests and as templates for future experiments.

Relevant implementation: [`experiment_manager/hil_experiments/basic_hardware_test/`](experiment_manager/hil_experiments/basic_hardware_test/) and [`experiment_manager/hil_experiments/ofdm_hardware_test/`](experiment_manager/hil_experiments/ofdm_hardware_test/).

### Initial CI and image-build groundwork

The repository now includes a Dockerfile and an image-build utility that accepts a GNU Radio commit reference, verifies the image, and can optionally push it to a registry. This is an important foundation for the project’s longer-term objective: validate a GNU Radio pull request or revision on real CortexLab radios.

The work intentionally stops short of claiming complete CI integration. No checked-in CI workflow currently builds an image, passes it through the experiment request to MINUS, runs a HIL experiment, and reports the result back to a pull request. The current scenario path still uses the configured default image at run time.

Relevant implementation: [`ci_workflow/build_gnuradio_image.py`](ci_workflow/build_gnuradio_image.py) and [`ci_workflow/docker_image/Dockerfile.pr`](ci_workflow/docker_image/Dockerfile.pr).

## Architecture

```text
Developer / CI request
          |
          v
Flask API + Dashboard
          |
          +-- OAR reservation monitor --> assigned CortexLab nodes
          |
          +-- MINUS scenario/task --> GNU Radio container image
          |
          +-- Experiment manager --> per-run folder + parameters.json
          |                              |
          |                              v
          |                         SSH/SFTP staging
          |                              |
          v                              v
Execution registry <--- synchronized node scripts (TX/RX)
          |
          v
analysis.py --> results.json --> final status and metrics
```

## Development Timeline

The repository history shows continuous development from June through late August 2026. The work progressed from remote SSH access, OAR reservation and scenario generation, and a reservation dashboard, through task creation, node monitoring, file upload, execution tracking, synchronized execution groups, analysis/result collection, and finally the basic and OFDM experiment implementations plus image-build parameterization.

Notable milestones include:

- June: remote gateway SSH workflow, OAR reservation/scenario generation, initial Flask status UI, and early tests.
- Early July: reservation/task monitoring, node state monitoring, remote file upload, dashboard improvements, and execution logging/tracking.
- Mid to late July: execution state model and synchronized execution groups.
- Early August: result downloading, result normalization, and dashboard status improvements.
- Mid to late August: basic hardware analysis, OFDM TX/RX implementation, parameterized GNU Radio image building, and CortexLab node-configuration updates.

## Testing and Validation

The repository contains local tests covering reservation parsing, scenario generation, registry behavior, Flask status handling, experiment-folder creation, and experiment-directory structure. A fresh `pytest -q` invocation during final-report preparation did not complete collection because several tests use stale import paths (for example, `cortexlab.flask_server` and a top-level `create_experiment_folder`) that do not match the current repository layout. Setting the repository root on `PYTHONPATH` resolves some package imports but leaves those stale paths unresolved.

This is a packaging/test-maintenance issue, not evidence that the physical HIL workflow passed or failed. The README also correctly distinguishes local automated tests from end-to-end hardware validation. A next step should be to repair the test imports, add deterministic unit tests around the execution state machine and result parser, and run a documented end-to-end CortexLab smoke test for both example experiments.

## Challenges and Lessons Learned

The core difficulty was orchestration across shared, remote hardware: reservation state is asynchronous, assigned nodes may not yet be usable, transfer and execution occur through a gateway, and transmitter/receiver timing matters. The solution treats state as a first-class concern through reservation, node, and execution registries; background monitors; explicit state transitions; and synchronized start coordination.

Another lesson is that a useful HIL framework needs a clear experiment contract. Keeping experiment-specific code in a standard folder structure while the controller handles allocation, transfer, start, and collection makes the framework practical to extend. Finally, real-hardware CI is a system integration problem, not merely a Docker build problem: image naming, registry access, MINUS image selection, secrets, hardware availability, cleanup, and PR reporting all need an integrated design.

## Remaining Work

The main remaining tasks are:

Integrate a production CI workflow that builds and publishes the requested GNU Radio image, passes its image reference to MINUS scenario generation, runs the controller, and reports the verdict back to the originating pull request.
Add more tests to cover all hardware-relevant blocks.
Refine the multi-experiment mechanism to properly handle cases where a task is already running.
Maintain a record of which tests cover particular blocks and add a mechanism to request tests by block name.
Improve failure cleanup, retries, timeouts, and artifact-retention policies for long-running shared-hardware jobs.

## Conclusion

The project has delivered the essential control path for GNU Radio HIL testing on CortexLab: one request can reserve hardware, stage a parameterized experiment, start coordinated radio programs, collect analysis artifacts, and expose a final machine-readable result. The basic hardware and OFDM experiments demonstrate that the controller supports both signal-level validation and payload-level verification.

The resulting codebase is a credible foundation for physical-hardware regression testing of GNU Radio changes. Completing CI integration and production hardening will turn that foundation into a repeatable, secure, and maintainable HIL validation service.

## Acknowledgements

Thank you to my mentors, Marcus Müller and Cyrille Morin, the CortexLab operators, and the GNU Radio community for their guidance, support, and infrastructure.
