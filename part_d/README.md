# 12. ArpResponder Table #20

This table defines how OVS handles the ARP requests.

Table 20 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=20 --no-stats
 cookie=0x1020000000000, table=20, priority=200,arp,arp_tpa=10.222.0.1,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:aa:bb:cc:dd:ee:ff,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0xaabbccddeeff->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xade0001->NXM_OF_ARP_SPA[],IN_PORT
 cookie=0x1020000000000, table=20, priority=200,arp,arp_tpa=10.222.2.1,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:aa:bb:cc:dd:ee:ff,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0xaabbccddeeff->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xade0201->NXM_OF_ARP_SPA[],IN_PORT
 cookie=0x1000000000000, table=20, priority=190,arp actions=NORMAL
 cookie=0x1000000000000, table=20, priority=0 actions=drop
vmware@master:~$ 
</code></pre>

The first flow entry is for the ARP requests sent for the IP address 10.222.0.1 which is **master** node' s antrea-gw0 IP. The second entry is for hthe ARP requests sent for the IP address 10.222.2.1 which is the **worker2** node' s antrea-gw0 IP. Why would worker 1 send ARP requests for other nodes' gw0 interface IPs ? The answer lies in the routing table of the worker 1 node. (which was shown back in [Part A Section 3.1.1](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_a#311-worker-1)) Shown below once again.

<pre><code>
vmware@<b>worker1</b>:~$ <b>ip route</b>
default via 10.79.1.1 dev ens160 proto static
10.79.1.0/24 dev ens160 proto kernel scope link src 10.79.1.201
<b>10.222.0.0/24 via 10.222.0.1 dev antrea-gw0 onlink </b>
10.222.1.0/24 dev antrea-gw0 proto kernel scope link src 10.222.1.1
<b>10.222.2.0/24 via 10.222.2.1 dev antrea-gw0 onlink </b>
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
</code></pre>

The highlighted lines in the above output are the route entries for the pod subnets of the **other** nodes. To remind again, the pod subnets assigned by node ipam controller to each individual Kubernetes node in this Kubernetes cluster is as following : 

- master - 10.222.0.0/24
- worker1 - 10.222.1.0/24
- worker2 - 10.222.2.0/24

The bold 

arp,arp_tpa=10.222.0.1,arp_op=1 

actions=
move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],
mod_dl_src:aa:bb:cc:dd:ee:ff,
load:0x2->NXM_OF_ARP_OP[],
move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],
load:0xaabbccddeeff->NXM_NX_ARP_SHA[],
move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],
load:0xade0001->NXM_OF_ARP_SPA[],
IN_PORT




arp,arp_tpa=10.222.0.1,arp_op=1 

actions=
move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],
mod_dl_src:aa:bb:cc:dd:ee:ff,
load:0x2->NXM_OF_ARP_OP[],
move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],
load:0xaabbccddeeff->NXM_NX_ARP_SHA[],
move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],
load:0xade0001->NXM_OF_ARP_SPA[],
IN_PORT


First two flows programmed in this table is to reply to ARP requests sent by Gateway0 for other Gateway IPs (basically other gateway interfaces on other nodes) Why ? Let’ s focus on second flow (10.222.1.1) first.  When a pod on worker 2 communicates to a Kubernetes Service (ClusterIP) then that needs to processed by local kube-proxy process to be load balanced (DNAT) to either a local or remote pod. If kube proxy picks a remote pod as the destination, then worker 2 needs to know how to get to that pod. Remote pod in this test runs on worker 1. Worker 1’ s pod subnet is 10.222.1.0 /24. On worker 2 there is an “on-link” route entry (slide 5) which makes the Kernel IP stack think that 10.222.1.0 is directly connected but the next hop for that route entry is 10.222.1.1. Hence worker 2 has to send an ARP entry for 10.222.1.1 (since it is a directly connected network – “on link”) to access the network 10.222.1.0. The ARP entry has a generic virtual mac aa:bb:cc:dd:ee:ff as the MAC address (instead of real MAC address of the respective Gateway0 interface of the destination node). (slide 5) This is the same on all nodes. This helps on the receiving end. Cause the worker 1 node OVS receives the traffic with this destination MAC and it can deliver the traffic right towards to the destination pod rather than the local Gateway0 interface.
Second flow entry in this flow table is for 10.222.0.1 (which is the pod subnet for master node) . Third flow instructs OVS to handle the remaining ARP requests as ordinary ARP traffic (which is Layer 2 broadcast).  Last rule is to drop all other type of traffic which got to this table. 
