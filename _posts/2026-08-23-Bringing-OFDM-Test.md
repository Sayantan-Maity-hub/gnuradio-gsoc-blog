# Bringing OFDM to Hardware: Building and Debugging a GNU Radio USRP Test

## Introduction

As part of my work on the **Hardware-in-the-Loop (HIL) CI framework for GNU Radio using CortexLab**, one of the important steps was to move an OFDM transmission and reception flowgraph from a software-only setup to real SDR hardware.

The initial OFDM example was designed around GNU Radio's virtual source and sink blocks. While this is useful for validating the signal-processing chain, it does not test whether the complete flowgraph works correctly with actual hardware.

The goal was therefore to build an OFDM TX/RX experiment that could run on CortexLab nodes equipped with USRP devices and could eventually be executed automatically as part of the HIL testing framework.

---

## From a Software Flowgraph to Real Hardware

The original OFDM example used GNU Radio's virtual connections for transmitting and receiving samples.

For the HIL experiment, I replaced this architecture with:

```text
TX Flowgraph
Message
   ↓
OFDM Modulation
   ↓
USRP Sink
   ↓
   RF
   ↓
USRP Source
   ↓
OFDM Demodulation
   ↓
File Sink
   ↓
Received Payload
```

This introduced several additional parameters that are not relevant in a purely software-based simulation.

For example:

- Sample rate
- Center frequency
- TX gain
- RX gain
- TX amplitude
- Hardware synchronization
- USRP source/sink configuration

These parameters had to be carefully coordinated between the transmitter and receiver.

---

## Parameterizing the Experiment

Instead of hard-coding experiment parameters inside the Python flowgraphs, I introduced a `parameters.json` file.

A simplified configuration looks like:

```json
{
    "message": "Hello CortexLab",
    "tx_gain": 10,
    "tx_amplitude": 0.05,
    "sample_rate": 195312,
    "rx_gain": 20,
    "center_frequency": 915000000,
    "rx_output_file": "rx_payload.bin"
}
```

The TX and RX scripts read the configuration from this file and use the values to configure their respective GNU Radio blocks.

This provides an important advantage for HIL testing: the experiment configuration can be changed without modifying the flowgraph itself.

For example, the same OFDM experiment can be executed with different:

- Messages
- Frequencies
- Gains
- Sample rates
- Output files

This also makes the experiment easier to integrate with an automated CI system.

---

## Making TX and RX Parameters Consistent

One of the most important debugging lessons from this experiment was that **OFDM parameters must match between the transmitter and receiver**.

Parameters such as:

- FFT length
- Cyclic prefix length
- Occupied carriers
- Pilot carriers
- Pilot symbols
- Header configuration
- Sync words
- Constellation/modulation parameters

are not independent on the two sides.

If the transmitter generates an OFDM frame using one configuration while the receiver expects another, the receiver may still execute successfully but fail to recognize or decode the transmitted packet.

This initially caused problems during the hardware test.

After comparing the TX and RX configurations, I found that some of the header-related parameters were not exactly matching. I updated the RX configuration so that it used the same OFDM parameters as the transmitter.

---

## Debugging the OFDM Header

After correcting the basic configuration, I started debugging the complete receive chain.

The RX flowgraph contains several stages responsible for detecting and decoding an OFDM packet.

Conceptually:

```text
USRP Source
     ↓
OFDM Synchronization
     ↓
Header Parser
     ↓
Header/Payload Demultiplexer
     ↓
Payload Decoder
     ↓
File Sink
```

One of the errors observed during debugging was:

```text
packet_headerparser_b :info:
Detected an invalid packet at item 0
```

followed by:

```text
header_payload_demux :
Parser returned #f
```

This was particularly interesting because the hardware flowgraph itself was running without a fatal execution error.

The problem was therefore not simply that the RX flowgraph failed to start. Instead, the receiver was receiving samples but was unable to interpret the OFDM frame correctly.

---

## Debugging Step by Step

Rather than treating the entire RX chain as a single black box, I decided to debug it incrementally.

The first step was to verify that the USRP receiver was actually producing samples.

I inserted a file sink directly after the USRP source:

```text
USRP Source
     ↓
File Sink
```

This allowed me to determine whether raw IQ samples were being captured successfully.

If the file contained data, it confirmed that:

1. The USRP was connected.
2. The UHD configuration was valid.
3. The RX flowgraph was receiving samples.
4. The problem was likely somewhere further downstream in the OFDM processing chain.

This approach made the debugging process much more systematic.

---

## Handling the Payload

Another part of the experiment was verifying that the transmitted message actually reached the receiver.

The transmitter converts the message into a payload representation before passing it into the OFDM transmission chain.

For example:

```text
Hello CortexLab
```

is represented as:

```text
48656c6c6f20436f727465784c6162
```

during payload processing.

The receiver eventually writes the decoded payload into:

```text
rx_payload.bin
```

The analysis stage can then inspect this file and determine whether the received payload matches the transmitted data.

This creates a simple end-to-end validation:

```text
TX Message
    ↓
OFDM Encoder
    ↓
USRP
    ↓
RF Channel
    ↓
USRP
    ↓
OFDM Decoder
    ↓
rx_payload.bin
    ↓
Analysis
    ↓
PASS / FAIL
```

---

## Separating Execution From Analysis

Another improvement was separating the actual experiment from result validation.

The RX flowgraph is responsible for receiving and storing the payload.

The analysis script is responsible for determining whether the experiment succeeded.

This separation is useful for HIL CI because the controller does not need to understand the internal details of the OFDM flowgraph.

Instead, the experiment can produce a simple result such as:

```json
{
    "status": "passed"
}
```

or:

```json
{
    "status": "failed"
}
```

The CI system can then use this result to decide whether the hardware test passed.

---

## What I Learned From the Debugging

The OFDM experiment taught me that getting a GNU Radio flowgraph to execute is only the first step.

There are several layers that need to be verified:

### 1. Hardware layer

Is the USRP correctly detected and configured?

### 2. Sample layer

Is the receiver actually obtaining IQ samples?

### 3. Synchronization layer

Can the receiver identify the OFDM frame?

### 4. Header layer

Can the receiver correctly decode the packet header?

### 5. Payload layer

Can the receiver recover the transmitted payload?

### 6. Analysis layer

Can the experiment automatically determine whether the expected data was received?

This layered approach is especially important for HIL testing because a test should distinguish between a hardware/configuration problem and an actual GNU Radio processing problem.

---

## Why This Matters for HIL CI

The final goal is not simply to make one OFDM experiment work manually.

The experiment needs to become part of an automated hardware-testing framework.

The intended workflow is:

```text
GitHub Pull Request
        ↓
Identify GNU Radio revision
        ↓
Build / select matching GNU Radio environment
        ↓
Create HIL experiment
        ↓
Reserve CortexLab nodes
        ↓
Run TX/RX OFDM experiment
        ↓
Collect rx_payload.bin
        ↓
Run analysis.py
        ↓
Generate result.json
        ↓
Return PASS / FAIL
        ↓
GitHub Actions
```

This allows a GNU Radio change to be tested against real SDR hardware instead of relying only on software-based unit tests.

---

## Current Status

The OFDM flowgraph has now been adapted for hardware-based execution.

The main changes include:

- Replacing virtual source/sink components with USRP hardware.
- Parameterizing TX and RX configurations.
- Introducing `parameters.json`.
- Matching the OFDM configuration between TX and RX.
- Debugging header synchronization and packet parsing.
- Capturing the received payload into `rx_payload.bin`.
- Separating experiment execution from result analysis.
- Preparing the experiment for integration with the CortexLab HIL CI framework.

The remaining work is to make the complete test robust enough for automated execution and ensure that the analysis stage can reliably distinguish successful and unsuccessful hardware transmissions.

---

## Conclusion

Moving the OFDM example from a software-only environment to real USRP hardware exposed several issues that would not necessarily appear during simulation.

The most important lesson was that **hardware testing requires validating the complete signal path**, not just whether the GNU Radio flowgraph starts successfully.

By parameterizing the experiment, synchronizing TX/RX configurations, debugging the receive chain stage by stage, and separating execution from analysis, the OFDM experiment is becoming a reusable building block for the GNU Radio HIL CI framework.

Ultimately, this work will allow GNU Radio changes to be tested automatically on real SDR hardware through CortexLab, bringing hardware validation closer to the normal software CI workflow.
