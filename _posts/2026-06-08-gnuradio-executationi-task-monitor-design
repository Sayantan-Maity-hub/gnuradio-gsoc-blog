# Weekly Progress Report: GNU Radio Execution and Task Monitoring Design

## Overview

This week I focused on learning the CorteXlab workflow for running GNU Radio applications on remote USRP nodes. Along the way, I encountered and resolved several execution issues related to task packaging, container environments, and GNU Radio configuration. I also designed an initial concept for a task monitoring system that can track experiment status and collect success/failure information across multiple nodes.

---

## 1. Creating and Running a GNU Radio Flowgraph

I created a GNU Radio flowgraph using GNU Radio Companion (GRC) based on the tutorial provided by my mentor.

After generating the Python file, I packaged it into a MINUS task and submitted it to a USRP-equipped node on CorteXlab.

This helped me understand:

* GNU Radio Companion (GRC)
* Generated Python flowgraphs
* MINUS task creation and submission
* Running experiments on remote nodes

---

## 2. Debugging Task Execution

### File Location Issue

My first attempt failed because the container could not find the Python file.

Error:

```text
python3: can't open file '/root/test_recieve.py'
```

To investigate, I used a temporary command inside the container to locate the file.

The actual path was:

```text
/cortexlab/homes/sayantan_maity/test_reciever/test_recieve.py
```

This confirmed that the task package was transferred correctly and helped identify the correct execution path.

---

### GNU Radio Import Failure

After fixing the file path issue, the next error was:

```text
ModuleNotFoundError: No module named 'gnuradio'
```

Initially, I suspected a problem with the Docker image, but further testing showed that GNU Radio was installed correctly.

The issue was that the GNU Radio environment was not initialized when Python was executed directly.

This failed:

```bash
python3 test_recieve.py
```

This worked:

```bash
bash -lc "python3 test_recieve.py"
```

The solution was to launch Python through `bash -lc`, which correctly loads the GNU Radio environment inside the container.

---

### Successful USRP Initialization

Once the environment issue was resolved, the flowgraph successfully started and connected to the USRP.

The logs showed:

* UHD initialization
* USRP device detection
* Successful communication with the radio hardware

This confirmed that:

* The Docker image was correct
* GNU Radio was installed properly
* UHD was functioning correctly
* The node could access the USRP

---

### Flowgraph Waiting for User Input

The next issue appeared after startup.

The flowgraph printed:

```text
Press Enter to quit:
```

The generated Python file had been created with the **Prompt for Exit** option in GRC.

Since tasks run non-interactively inside containers, the program waited indefinitely for keyboard input.

My mentor suggested using:

```text
Run to Completion
```

instead of:

```text
Prompt for Exit
```

when generating the Python flowgraph.

This removes the interactive prompt and allows automated execution.

---

## 3. Container Debugging Techniques Learned

During the debugging process, I learned several useful techniques:

* Reading task logs (`stdout` and `stderr`)
* Inspecting result directories
* Locating files inside containers
* Understanding container execution environments
* Diagnosing GNU Radio startup issues

These techniques significantly improved my understanding of remote experiment execution.

---

# 4. Proposed Task Monitoring System

## Motivation

As the number of experiments grows, manually checking each task becomes inefficient.

I wanted a way to answer:

* Which tasks are currently running?
* Which nodes are being used?
* Which tasks passed?
* Which tasks failed?
* Why did a task fail?

---

## Proposed Solution: JSON-Based Task Registry

I propose maintaining a centralized JSON registry on the controlling machine.

When a task is submitted, it is automatically added to the registry.

Example:

```json
{
  "24360": {
    "status": "RUNNING",
    "nodes": {
      "node17": {},
      "node03": {}
    }
  }
}
```

---

## Monitoring Process

A Python monitoring service periodically checks task status using:

```bash
minus task info <task_id>
```

and updates the registry.

Possible task states:

* WAITING
* RUNNING
* FINISHED
* ABORTED

---

## Multi-Node Task Support

For tasks running on multiple nodes, each node is monitored independently.

Example:

```json
{
  "24360": {
    "nodes": {
      "node17": {
        "status": "PASS"
      },
      "node03": {
        "status": "FAIL",
        "reason": "ModuleNotFoundError: No module named 'gnuradio'"
      }
    },
    "overall_status": "FAIL"
  }
}
```

This makes it easy to identify exactly which node failed.

---

## Failure Reason Collection

When a task completes, the monitoring service inspects:

* `stdout`
* `stderr`

and extracts useful failure information such as:

```text
ModuleNotFoundError
RuntimeError
FileNotFoundError
UHD initialization errors
```

The extracted error message is stored in the registry.

Example:

```json
{
  "node03": {
    "status": "FAIL",
    "reason": "ModuleNotFoundError: No module named 'gnuradio'"
  }
}
```

---

## Example Dashboard View

```text
Task 24359
└── node24 : PASS

Task 24360
├── node17 : PASS
└── node03 : FAIL
    Reason: ModuleNotFoundError: No module named 'gnuradio'

Task 24361
├── node08 : RUNNING
└── node12 : RUNNING
```

---

## Future Extensions

Because the registry is stored as JSON, it can later be integrated with:

* CI/CD pipelines
* Web dashboards
* Databases
* Experiment management tools

without significant redesign.

---

## Conclusion

This week I successfully learned how to deploy and debug GNU Radio experiments on CorteXlab. I resolved issues related to file paths, container environments, GNU Radio initialization, and flowgraph execution.

Additionally, I designed a JSON-based task monitoring architecture capable of tracking task states, node-level success/failure information, and failure reasons. This provides a foundation for future automation and experiment monitoring capabilities.
