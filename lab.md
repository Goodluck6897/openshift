https://crc.dev/docs/using/


OVS runs inside the ovnkube-node pod: Open vSwitch isn't running as a standalone pod. Look closely at your output: pod/ovnkube-node-lg52f has 8/8 containers running.One of those 8 hidden containers inside that exact pod is ovs-daemons, which runs the OVS processes.


ou can see the ovn-controller in the exact same place as OVS—it runs as one of the 8 containers inside your ovnkube-node-lg52f pod.The ovn-controller is the local daemon that connects to the Southbound Database (sbdb) and programs OpenFlow rules into the local Open vSwitch (ovs-daemons) engine.
