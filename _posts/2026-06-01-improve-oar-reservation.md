# Improving the OAR Reservation Orchestrator

This week, I improved the initial HIL orchestration prototype by making the OAR reservation process more user-friendly and flexible.

The script now supports three reservation modes:

* Best Allocation
* Preferred Nodes
* Future Reservation

Users can configure walltime and reservation parameters interactively, while the script automatically generates the appropriate OAR command.

To simplify node selection, users only need to enter node numbers (e.g., `31,36`), and the script converts them into the corresponding CortexLab hostnames automatically.

I also added an interactive TX/RX assignment step. After nodes are allocated, the script displays the available nodes and allows users to choose which node should act as the transmitter (TX) and which should act as the receiver (RX). This provides more flexibility when preparing HIL experiments involving multiple nodes.

These improvements establish a cleaner foundation for the next phase: automatic scenario generation, remote flowgraph execution, and result collection for Hardware-in-the-Loop testing.

LINK: https://github.com/Sayantan-Maity-hub/gnuradio-hardware-in-loop/blob/main/cortexlab/orchestration.py
