# Introducing Execution Groups for Synchronized Experiment Execution

This week, I implemented **Execution Groups**, a new feature that allows multiple scripts to be managed and executed together as a single experiment. This lays the foundation for synchronized multi-node experiments, such as TX/RX communication, in the Hardware-in-the-Loop (HIL) framework.

## What I Implemented

### Execution Group Registry
Created a dedicated registry to manage execution groups. Each group stores:
- Job ID and Task ID
- Group name
- Assigned nodes
- Execution IDs
- Synchronization state
- Overall execution status and timestamps

### Group Creation API
Added new REST endpoints to:
- Create execution groups
- List existing execution groups
- Start synchronized execution for a group

The API also validates that:
- Selected nodes belong to the reservation.
- Nodes are not already used by another active execution group.
- Duplicate node selection is prevented.

### Script Upload Integration
Updated the upload workflow so scripts can be uploaded directly into an execution group. Each uploaded script is automatically linked with the corresponding execution group and node.

The execution group tracks upload progress and moves to the **READY_TO_RUN** state once every required node has received its script.

### Synchronized Execution
Implemented synchronized execution using Python's `threading.Barrier`.

The workflow is:
1. Prepare every node.
2. Wait until all nodes reach the **READY** state.
3. Release all nodes simultaneously.
4. Start execution together using a common start time.

This ensures that distributed experiments begin in a coordinated manner.

### Execution Group Monitoring
Added logic to monitor all executions belonging to a group. Once every execution finishes, the framework automatically computes the final group result:
- PASS if every execution succeeds.
- FAILED if any execution fails.

### Dashboard Support
Extended the web dashboard with a new **Synchronized Execution Groups** section, allowing users to:
- Create execution groups
- Upload one script per node
- View synchronization status
- Start synchronized execution
- Monitor group progress

## Summary

This implementation introduces a higher-level abstraction for managing distributed experiments. Instead of handling each execution independently, related executions can now be organized, synchronized, and monitored as a single execution group, providing the foundation for future multi-node GNU Radio experiments.
