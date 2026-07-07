# Progress Update: Building a Distributed Controller for HIL TEST

First of all, I would like to apologize for not writing a blog for quite a long time. During the past few weeks, most of my time has been spent discussing ideas, exploring different approaches, and gradually refining the architecture of the project.

Initially, I always came up with simple script-based solutions. My instinct was to automate each task with a single script and connect everything together. However, every time I presented these ideas, my mentor encouraged me to think beyond scripts and instead focus on designing a complete distributed system composed of multiple interacting components. Looking back, that advice completely changed the direction of this project.

After several discussions, Cyrill suggested an interesting observation. Since **minus** starts an SSH server on the reserved node when no command is specified in the scenario, we could directly access the running node through SSH after the task starts. Based on this idea, we decided to build the communication layer using **Paramiko** for SSH and **Flask** for exposing a REST API that powers the dashboard.

Marcus also helped by recommending both Paramiko and Flask as suitable technologies for implementing this architecture. These discussions helped define a much clearer roadmap for the project.

---

# Development Milestones

## Milestone Group 1 – Controller to Runner Communication

### Milestone 1 – Establish Controller–Runner Communication

#### Goal

Enable the Controller to communicate with reserved CortexLab nodes through SSH while exposing node information through a REST API.

#### Design

The architecture is based on the following workflow:

- When no command is specified in the scenario, **minus** starts an SSH server on the reserved node.
- The Controller opens SSH connections to every reserved node using **Paramiko**.
- The Controller gathers basic information such as:
  - Hostname
  - Operating System
- The information is stored inside an in-memory registry.
- **Flask** exposes this registry through REST endpoints.

Example:

```http
GET /status/nodes
```

This allows the dashboard to query the current status of every connected node.

One interesting challenge here was thread safety. The Flask server runs independently from the SSH communication threads, so shared data structures had to be designed carefully to avoid race conditions while node information is being updated.

---

### Milestone 2 – Single Node Job Execution

Once node communication was established, the next objective was allowing the Controller to execute work remotely.

#### Execution Pipeline

1. Upload the execution script.
2. Transfer the script through SFTP over the existing SSH connection.
3. Execute the script remotely.
4. Capture **stdout** and **stderr**.
5. Store execution results inside an in-memory execution registry.
6. Expose the execution state through REST APIs.

Example execution script:

```bash
echo "Hello CortexLab"
echo "::STATUS:Basic Test:PASS:"
exit 0
```

The Controller monitors the execution and reports state transitions such as:

- Job Assigned
- Running
- Finished

along with the final execution result.

---

### Milestone 3 – Multi-Node Job Distribution

The final milestone extends the same execution model to multiple nodes simultaneously.

Instead of managing a single Runner, the Controller now coordinates several nodes concurrently while keeping their execution states synchronized through the dashboard.

---

# Current Progress

At the time of writing, I have almost completed the implementation up to **Milestone 3**.

The system is now capable of managing the complete workflow from reservation to remote execution.

---

## Reservation Management

Users can create a reservation by providing:

- Username
- SSH private key
- Preferred nodes *(optional)*
- Walltime
- Reservation type

After submission, the dashboard continuously displays:

- Reservation state
- Assigned nodes
- Waiting time
- Job information

---

## Task Creation

After the reservation becomes active, users can create a task by simply entering a task description.

The Controller automatically generates a **scenario.yaml** for all assigned nodes with **no command specified**. This is intentional because it allows **minus** to launch an SSH server inside the Docker image instead of immediately executing a command.

The Controller then automatically:

- Generates the scenario
- Uploads it to CortexLab
- Creates the task
- Submits the task

Once submitted, the dashboard displays:

- Task ID
- Description
- Remote working folder
- Current task state

---

## Live Node Discovery

As soon as a task enters the **Running** state, the Controller connects to every reserved node using:

```bash
ssh -p 2222 root@mnode<node>
```

Through SSH, it executes commands such as:

```bash
hostname
cat /etc/os-release
```

If communication succeeds, the dashboard immediately marks the node as **Online** and displays its information using dedicated node cards.

This confirms that the Controller has successfully established communication with the live Docker containers running on the reserved CortexLab nodes.

---

## Remote Script Upload

Each node card allows uploading an execution script.

The uploaded script is transferred to the shared CortexLab home directory:

```text
/cortexlab/homes/<username>/<job>/<task>/<node>/
```

using SFTP over the existing SSH connection.

This means every node receives its own execution script while sharing the same Controller interface.

---

## Remote Execution

Each node card also contains an **Execute** button.

Depending on the uploaded file extension, the Controller automatically selects the appropriate interpreter.

| Extension | Runner |
|-----------|--------|
| `.sh` | `bash` |
| `.py` | `python3` |
| `.pl` | `perl` |
| `.rb` | `ruby` |
| `.php` | `php` |
| `.js` | `node` |

The script is executed directly on the live CortexLab node through SSH.

During execution, the Controller continuously captures:

- Standard Output
- Standard Error
- Exit Code
- Execution Status

The complete execution log is saved as:

```text
execution.log
```

inside the user's CortexLab home directory, allowing future download, parsing, or automated result extraction.

The backend infrastructure is already capable of storing all execution metadata, outputs, and logs in memory.

---

# What's Next?

Although the core functionality up to **Milestone 3** is almost complete, there are still a few improvements remaining before it can be considered fully finished.

The remaining work for **Milestone 3** includes:

- Display live execution results directly in the dashboard.
- Provide a dedicated execution panel showing every script execution.
- Allow users to download execution log files (`execution.log`).
- Parse execution logs into structured experiment results.
- Improve visualization for multiple concurrent executions across different nodes.

Once these features are completed, the Controller will support the complete workflow from reservation to multi-node remote execution with real-time monitoring.

---

# Milestone 4 – Experiment State Tracking

## Goal

Track experiment progress and make it available through REST endpoints under `/status/...`.

## Problem

The Controller must always know the current state of every experiment.

## Experiment States

```text
CREATED
    ↓
ASSIGNED
    ↓
PREPARING
    ↓
READY
    ↓
RUNNING
    ↓
FINISHED
```

If an error occurs, the final state becomes:

```text
FAILED
```

## Implementation

- Runners report state changes.
- Controller updates experiment status.
- Dashboard queries the current experiment state through REST APIs.

## Demonstration

```text
Experiment 17

CREATED
    ↓
ASSIGNED
    ↓
PREPARING
    ↓
READY
    ↓
RUNNING
    ↓
FINISHED
```

## Estimated Effort

**8–12 Hours**

---

## Project Status

✅ Controller–Runner communication

✅ REST API using Flask

✅ SSH communication using Paramiko

✅ Reservation management

✅ Automatic scenario generation

✅ Task creation and submission

✅ Live node discovery

✅ Script upload through SFTP

✅ Remote script execution

✅ Execution monitoring backend

⏳ Dashboard execution panel

⏳ Execution log download

⏳ Experiment state tracking

⏳ Multi-node execution visualization
