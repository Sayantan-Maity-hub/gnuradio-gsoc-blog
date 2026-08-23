---
title: "Initial Hardware Test with CortexLab"
layout: post
---

# Initial Hardware Test with CortexLab

The first milestone of my GNU Radio HIL CI work was building a basic hardware test using **CortexLab and USRP devices**.

The goal was to create a complete pipeline that could configure the hardware, run a GNU Radio TX/RX experiment, capture IQ samples, analyze the received signal, and return a **PASS/FAIL** result.

## What I Implemented

### Configurable USRP TX/RX

I added configurable GNU Radio flowgraphs for transmitting a tone and capturing IQ samples using USRP hardware.

```text
TX Node                 RX Node
Signal Source           USRP Source
     ↓                       ↓
USRP Sink               File Sink
     └────── RF ─────────────┘
                 ↓
              Analysis
```

The experiment parameters, such as frequency, sample rate, gain, and amplitude, can be configured externally.

### IQ Analysis

I added an analysis stage to validate the received signal by calculating:

- Signal power
- RMS amplitude
- Dominant frequency
- Frequency error
- Validation status

This provides an automated way to determine whether the hardware transmission was successful.

### Experiment Management

The framework was extended to manage the complete experiment lifecycle:

```text
Request
  ↓
Create Experiment
  ↓
Reserve Nodes
  ↓
Upload Scripts
  ↓
Run TX/RX
  ↓
Collect IQ Data
  ↓
Analyze
  ↓
PASS / FAIL
```

### Controller and Project Structure

I also added Flask APIs and a basic dashboard for monitoring reservations, nodes, and experiments.

The `cortexlab` package was reorganized into separate modules for:

- Execution
- Nodes
- Reservations
- Remote connections

Registries were also added to track reservations, nodes, and executions.

## Initial Issues Identified

The prototype also exposed several issues that needed improvement:

- SSH credentials initially used a hard-coded private-key path.
- The remote execution interface had a mismatch between returned values and what callers expected.
- `create_experiment_folder()` initially calculated the project root incorrectly.
- Reservation and task management used busy-wait loops with limited error handling.
- Some tests were written as standalone scripts and did not integrate cleanly with `pytest`.

These issues helped identify the areas that needed to be improved before making the framework suitable for reliable CI execution.

## Next Step

This initial hardware test established the foundation for more complex HIL experiments.

After validating the basic USRP TX/RX pipeline, I moved to a more realistic **OFDM communication experiment**, including configurable TX/RX flowgraphs, synchronized OFDM parameters, payload validation, and end-to-end message recovery.
