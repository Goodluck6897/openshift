https://crc.dev/docs/using/


OVS runs inside the ovnkube-node pod: Open vSwitch isn't running as a standalone pod. Look closely at your output: pod/ovnkube-node-lg52f has 8/8 containers running.One of those 8 hidden containers inside that exact pod is ovs-daemons, which runs the OVS processes.
