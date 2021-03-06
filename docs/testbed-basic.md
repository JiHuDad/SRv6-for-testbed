# SERVICE FUNCTIONS CHAINING

We consider a Service Function Chaining scenario supported by SRv6. In our
scenario, a service chain is an ordered set of Virtual Network Functions
(VNFs) and each VNF is represented by SID. We assume that VNFs are hosted in
"NFV nodes".

A complete description of the use-case of service function chaining in Linux
and the testbed is available [here](https://www.slideshare.net/amsalam20/service-function-chaining-with-srv6)

The SREXT kernel module allows introducing SR-unaware VNFs in a service chain
implemented with SRv6. It removes the Segment Routing encapsulation before
handing the packets to the VNF and properly reinserts the SR encapsulation to
the packets processed by the VNF. 

## Chaining of SR-unaware VNFs 

In order to replicate the experiment of chaining of SR-unaware VNFs by using
srext, we provide a simple VirtualBox testbed using vagrant.

The testbed is composed of three Virtual Machines (VMs), all running Linux
kernel and has srext kernel module installed. The three VMs represent SRv6
ingress node, NFV node, and SR egress node: 

**SR ingress node:** is the ingress of the SR domain and it processes incoming
packets, classifies them, and enforces a per-flow VNF chain; the list of VNF
identifiers is applied by encapsulating the original packets in a new IPv6
packets with a SRH reporting as segment list of the ordered list of SIDs of
the given VNFs

**SR egress node:** is the egress of the SR domain and removes the SR
encapsulation and forwards the inner packet toward its final destination.
This allows the final destination to correctly process the original packet.

**NFV node:** is running the VNFs as network namespaces and srext will be used
to provide SR proxy, which means it will process the SR information on behalf
of the VNFs since they are SR-unaware. 

The SR proxy removes the SR encapsulation before sending the packets to the
VNF. The SR Proxy re-adds the SR encapsulation again to the packets after
being processed by the VNF.

The ingress has a client running as a network namespace and it is used to
generate traffic (either simple ICMP packets or by means of iperf), this
traffic will be encapsulated using an SR policy defined on the ingress node.

The egress node has a server running as network namespace and it is used as
sink for traffic generated by the client on the ingress node. The egress node
will decapsulate the SR traffic and send packets to the server (End.Dx6). 

### Testbed Setup 
Before starting, please make sure that you have [vagrant](https://www.vagrantup.com/downloads.html) and 
[virtualbox](https://www.virtualbox.org/wiki/Downloads) installed on your machine.

Clone the git repository in your machine: 

```
$ git clone https://github.com/netgroup/SRv6-net-prog
```
Add the srv6-net-prog vagrant box:
```
$ vagrant box add srv6-net-prog http://cs.gssi.infn.it/files/SFC/srv6-net-prog.box
```
Start the testbed:
```
$ cd SRv6-net-prog/vagrant-box/testbed1/
$ vagrant up 
```
It takes a bit of time, please be patient .......

Now we have the three VMs running with srext installed, So let's configure them for our SFC scenario 

### Ingress node setup 
Open a new terminal and log into the ingress node: 
```
$ vagrant ssh ingress 
```

Make sure that you run ssh command from SRv6-net-prog/vagrant-box/testbed1/

Run the ingress node setup script: 
```
$ cd SRv6-net-prog/srext/scripts/testbed1/
$ sudo ./setup_ingress_testbed1.sh
```
The ingress setup script does the following:
1. Creates a network namespace that will be used as a client
2. Adds an SRv6 policy to steer traffic going from the client to the server
3. Uses the srext kernel module to add an End.DX6 localsid, The End.DX6 localsid is used to 
decapsulate the return traffic coming back from the server before being received by the client

To verify that everything went well, please do the following: 
- Verify that the client name space is created correctly
```
$ sudo ip netns
client (id: 0)
```
- Veryify SR policy 
```
$ sudo ip -6 route 
b::/64  encap seg6 mode encap segs 4 [ 2::ad6:f1 2::ad6:f2 2::ad6:f3 3::d6 ] via 1:2::2 dev eth1 metric 1024 pref medium
```
- Verify srext configuration

```
$ sudo srconf localsid show
SRv6 - MY LOCALSID TABLE:
==================================================
	 SID     :        1::d6
	 Behavior:        end.dx6
	 Next hop:        a::2
	 OIF     :        veth1_1
	 Good traffic:    [0 packets : 0  bytes]
	 Bad traffic:     [0 packets : 0  bytes]
----------------------------------------------------
```
### NFV node setup 
Open a new terminal and log into the NFV node: 
```
$ vagrant ssh nfv 
```
Make sure that you run ssh command from SRv6-net-prog/vagrant-box/testbed1/

Run the nfv node setup script: 
```
$ cd SRv6-net-prog/srext/scripts/testbed1/
$ sudo ./setup_nfv_testbed1.sh
```
The nfv setup script does the following:
1. Creates three network namespaces that will be used as SR-unaware VNFS(SCs)
2. Uses the srext kernel module to add an End.AD6 localsid for each VNF 
   1. The End.AD6 localsids (proxy behavior) are used to handle the processing of SRv6 encapsulation
   in behalf of the SR-unaware VNFs
   2. For simplicity, the End.AD6 (proxy behavior) will use the same interface as Target and source 
   interface
3. Uses the srext kernel module again to add an End localsid
   1. The End localsid will be used by the return traffic from the server to the client

To verify that everything went well, please do the following: 
- Verify that the VNFS are created correctly
```
$ sudo ip netns
vnf3 (id: 2)
vnf2 (id: 1)
vnf1 (id: 0)
```
- Verify srext configuration
```
$ sudo srconf localsid show
SRv6 - MY LOCALSID TABLE:
==================================================
	 SID     :        2::ad6:f3
	 Behavior:        end.ad6
	 Next hop:        2:f3::f3
	 OIF     :        veth3_2
	 IIF     :        veth3_2
	 Good traffic:    [0 packets : 0  bytes]
	 Bad traffic:     [0 packets : 0  bytes]
------------------------------------------------------
	 SID     :        2::
	 Behavior:        end
	 Good traffic:    [0 packets : 0  bytes]
	 Bad traffic:     [0 packets : 0  bytes]
------------------------------------------------------
	 SID     :        2::ad6:f1
	 Behavior:        end.ad6
	 Next hop:        2:f1::f1
	 OIF     :        veth1_2
	 IIF     :        veth1_2
	 Good traffic:    [0 packets : 0  bytes]
	 Bad traffic:     [0 packets : 0  bytes]
------------------------------------------------------
	 SID     :        2::ad6:f2
	 Behavior:        end.ad6
	 Next hop:        2:f2::f2
	 OIF     :        veth2_2
	 IIF     :        veth2_2
	 Good traffic:    [0 packets : 0  bytes]
	 Bad traffic:     [0 packets : 0  bytes]
------------------------------------------------------
```

### Egress node setup 
Open a new terminal and log into the egress node: 
```
$ vagrant ssh egress 
```

Make sure that you run ssh command from SRv6-net-prog/vagrant-box/testbed1/

Run the egress node setup script: 
```
$ cd SRv6-net-prog/srext/scripts/testbed1/
$ sudo ./setup_egress_testbed1.sh
```
The egress setup script does the following:
1. Creates a network namespace that will be used as a server
2. Adds an SRv6 policy to steer return traffic going from the server back to the client
3. Uses the srext kernel module to add an End.DX6 localsid, The End.DX6 localsid is used to 
decapsulate the SR traffic coming from the client before being received by the server

To verify that everything went well, please do the following: 
- Veryify that the server name space is created correctly
```
$ sudo ip netns
server (id: 0)
```
- Veryify SR policy 
```
$ sudo ip -6 route 
a::/64  encap seg6 mode encap segs 2 [ 2:: 1::d6 ] via 2:3::2 dev eth1 metric 1024 pref medium
```
- Verify srext configuration
```
$ sudo srconf localsid show
SRv6 - MY LOCALSID TABLE:
==================================================
	 SID     :        3::d6
	 Behavior:        end.dx6
	 Next hop:        b::2
	 OIF     :        veth1_3
	 Good traffic:    [0 packets : 0  bytes]
	 Bad traffic:     [0 packets : 0  bytes]
------------------------------------------------------
```
### Running the SFC use-case
Now it's time to run the SFC use-case, we ping the server on the egress node from the client on 
ingress node

-From the ingress node terminal: 
```
$ sudo ip netns exec client ping6 b::2
64 bytes from b::2: icmp_seq=5 ttl=63 time=0.634 ms
64 bytes from b::2: icmp_seq=6 ttl=63 time=0.595 ms
```
The server is reachable, then let's verify that the traffic is SR encapsulated as expected

-From the nfv node terminal: 
```
$ sudo tcpdump -i eth1
IP6 a::1 > 2::ad6:f1: srcrt (len=8, type=4, segleft=3, last-entry=3,
	tag=0, [0]3::d6, [1]2::ad6:f3, [2]2::ad6:f2, [3]2::ad6:f1)
	IP6 a::2 > b::2: ICMP6, echo request, seq 25, length 64
```
As you can see the traffic coming to NFV node from the ingress node is SR encapsulated type=4, segleft=3[|srcrt]

Let's verify that the SR proxy behavior is working as expected 

-From the nfv node terminal: 
```
$ sudo tcpdump -i veth1_2
 IP6 a::2 > b::2: ICMP6, echo request, seq 539, length 64

$ sudo tcpdump -i veth2_2
IP6 a::2 > b::2: ICMP6, echo request, seq 539, length 64

$ sudo tcpdump -i veth_2
IP6 a::2 > b::2: ICMP6, echo request, seq 539, length 64
```
The traffic going to the VNF is plain IPv6 traffic without SR encapsulation 
Let's look at the traffic leaving the NFV node towards the egress node 

-From the nfv node terminal: 
```
$ sudo tcpdump -i eth2
IP6 a::1 > 3::d6: srcrt (len=8, type=4, segleft=0, last-entry=3,
	tag=0, [0]3::d6, [1]2::ad6:f3, [2]2::ad6:f2, [3]2::ad6:f1)
	IP6 a::2 > b::2: ICMP6, echo request, seq 229, length 64
```
Here we say that the traffic is SR encapsulated again after being processed by the VNFs 
SREXT counters 
You can show the localsid table and to look to the counters (number of packets matched with each SID)

-From the nfv node terminal:

```
$ sudo srconf localsid show
SRv6 - MY LOCALSID TABLE:
======================================================
	 SID     :        2::ad6:f3
	 Behavior:        end.ad6
	 Next hop:        2:f3::f3
	 OIF     :        veth3_2
	 IIF     :        veth3_2
	 Good traffic:    [923 packets : 95992  bytes]
	 Bad traffic:     [0 packets : 0  bytes]
------------------------------------------------------
	 SID     :        2::
	 Behavior:        end
	 Good traffic:    [923 packets : 169832  bytes]
	 Bad traffic:     [0 packets : 0  bytes]
------------------------------------------------------
	 SID     :        2::ad6:f1
	 Behavior:        end.ad6
	 Next hop:        2:f1::f1
	 OIF     :        veth1_2
	 IIF     :        veth1_2
	 Good traffic:    [923 packets : 95992  bytes]
	 Bad traffic:     [0 packets : 0  bytes]
------------------------------------------------------
	 SID     :        2::ad6:f2
	 Behavior:        end.ad6
	 Next hop:        2:f2::f2
	 OIF     :        veth2_2
	 IIF     :        veth2_2
	 Good traffic:    [923 packets : 95992  bytes]
	 Bad traffic:     [0 packets : 0  bytes]
------------------------------------------------------
```

From the egress node terminal:

```
$ sudo tcpdump -i veth1_3
 IP6 a::2 > b::2: ICMP6, echo request, seq 539, length 64
```

### Notes 
--Resources assigned to any of the VMs can be customized by modifying Vagrantfile 
```
 virtualbox.memory = "1024"
 virtualbox.cpus = "1"
```
--Start-up configuration of ingress node, NFV node, or egress node can be customized by modifying 
the scripts in the vagrant-box folder.
