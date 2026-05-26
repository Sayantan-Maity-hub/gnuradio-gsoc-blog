Automating OAR Node Allocation Using Python

In my HIL/GNU Radio workflow, I created a small Python automation script to submit an OAR job, extract the job ID, and retrieve allocated compute nodes automatically.

The main purpose of this script is:

Extract the Job ID for getting detailed information about the submitted OAR job
Extract allocated hostnames/nodes for future automatic scenario YAML generation
Dynamically assign TX and RX nodes for SDR experiments

Link: https://github.com/Sayantan-Maity-hub/gnuradio-hardware-in-loop
