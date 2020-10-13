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

The first flow entry is for the ARP requests sent for the IP address 10.222.0.1 which is **master** node' s antrea-gw0 IP. The second flow entry is for the ARP requests sent for the IP address 10.222.2.1 which is the **worker2** node' s antrea-gw0 IP. **Why would worker1 node send an ARP request for another node' s gw0 interface IP ?** The answer lies in the routing table of the worker 1 node. (which was shown back in [Part A Section 3.1.1](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_a#311-worker-1)) Shown below once again.

<pre><code>
vmware@<b>worker1</b>:~$ ip route
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

Looking at "10.222.2.0/24 via 10.222.2.1 dev antrea-gw0 onlink" , this route entry basically means that the next hop for the subnet 10.222.2.0/24 is 10.222.2.1 and it can be reached through antrea-gw0 interface. (dev antrea-gw0) However 10.222.2.1 is not part of a directly connected subnet on worker1 node. So normally it would perform recursive routing to check how it can reach 10.222.2.1. The interesting trick here is "onlink", this instructs the Linux Kernel IP stack to think as if 10.222.2.1 is a next hop which is directly connected to the network which antrea-gw0 interface is also connected to. **Hence when worker1 node sends any data destined to subnet 10.222.2.0/24, it would first send an ARP request for 10.222.2.1.**

Looking at "10.222.0.0/24 via 10.222.0.1 dev antrea-gw0 onlink" , this route entry is for the pod subnet on master node. 

Similarly whenever a new node is added to the Kubernetes cluster, Antrea Node Controller would detect that by watching the Kubernetes API and add similar route entries on worker1 node' s route table. (Explained [here](https://github.com/vmware-tanzu/antrea/blob/master/docs/architecture.md#antrea-agent))

Until now, the reason a node sends an ARP request for another node' s gw0 interace IP is explained. What about how OVS handles these ARP requests ? For that the second flow entry in Table 20 is explained in more detail below.

<pre><code>
cookie=0x1020000000000, table=20, priority=200,arp,arp_tpa=10.222.2.1,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:aa:bb:cc:dd:ee:ff,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0xaabbccddeeff->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xade0201->NXM_OF_ARP_SPA[],IN_PORT
</code></pre>

This flow entry checks if the current flow that comes ingress to OVS is an arp flow (arp), and if the ARP request is sent for IP address 10.222.2.1 (arp_tpa=10.222.2.1, tpa stands for target IP address), and if it is an arp request (arp_op=1 is for arp request). Once all these match with the flow then the following actions are taken on the flow : 

* <b>move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[] :</b> This action moves the MAC address, seen in the source address field of the ethernet header, to the destination address field in the ethernet header of the response
* <b>mod_dl_src:aa:bb:cc:dd:ee:ff :</b> This action puts the MAC address "aa:bb:cc:dd:ee:ff" into the source address field in the ethernet header of the response
* <b>load:0x2->NXM_OF_ARP_OP[] :</b> This action basically sets "0x2" which is "2" in decimal for "arp_op" and "arp_op=2" is used for ARP response
* <b>move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[] :</b> This action moves the MAC address, seen in the sender MAC address field in the ARP header of the ARP request, to the target MAC address field in the ARP header of the response 
* <b>load:0xaabbccddeeff->NXM_NX_ARP_SHA[] :</b> This action puts the MAC address "aa:bb:cc:dd:ee:ff" into the sender MAC address field in the ARP header of the response
* <b>move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[] :</b> This action moves the IP address, seen in the sender IP address field of the ARP header, to the target IP address field in the ARP header of the response 
* <b>load:0xade0201->NXM_OF_ARP_SPA[] :</b> This action puts the IP address "10.222.2.1" (conversion from hex 0xade0201) into the the sender IP address field of the ARP header of the response 
* <b>IN_PORT :</b> This action means "forward the flow back to the port where it came" (which is the antrea-gw0 interface port on OVS on the worker1 node in this case)

Note : More detailed info about these fields can be found in the "ovs-fields" section on [this page](https://docs.openvswitch.org/en/latest/ref/?highlight=fields#man-pages).


First two flows programmed in this table is to reply to ARP requests sent by Gateway0 for other Gateway IPs (basically other gateway interfaces on other nodes) Why ? Let’ s focus on second flow (10.222.1.1) first.  When a pod on worker 2 communicates to a Kubernetes Service (ClusterIP) then that needs to processed by local kube-proxy process to be load balanced (DNAT) to either a local or remote pod. If kube proxy picks a remote pod as the destination, then worker 2 needs to know how to get to that pod. Remote pod in this test runs on worker 1. Worker 1’ s pod subnet is 10.222.1.0 /24. On worker 2 there is an “on-link” route entry (slide 5) which makes the Kernel IP stack think that 10.222.1.0 is directly connected but the next hop for that route entry is 10.222.1.1. Hence worker 2 has to send an ARP entry for 10.222.1.1 (since it is a directly connected network – “on link”) to access the network 10.222.1.0. The ARP entry has a generic virtual mac aa:bb:cc:dd:ee:ff as the MAC address (instead of real MAC address of the respective Gateway0 interface of the destination node). (slide 5) This is the same on all nodes. This helps on the receiving end. Cause the worker 1 node OVS receives the traffic with this destination MAC and it can deliver the traffic right towards to the destination pod rather than the local Gateway0 interface.
Second flow entry in this flow table is for 10.222.0.1 (which is the pod subnet for master node) . Third flow instructs OVS to handle the remaining ARP requests as ordinary ARP traffic (which is Layer 2 broadcast).  Last rule is to drop all other type of traffic which got to this table. 
