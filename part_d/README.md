

# 12. ARP Process

Table 20 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=20 --no-stats
 cookie=0x1020000000000, table=20, priority=200,arp,arp_tpa=10.222.0.1,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:aa:bb:cc:dd:ee:ff,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0xaabbccddeeff->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xade0001->NXM_OF_ARP_SPA[],IN_PORT
 cookie=0x1020000000000, table=20, priority=200,arp,arp_tpa=10.222.2.1,arp_op=1 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:aa:bb:cc:dd:ee:ff,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],load:0xaabbccddeeff->NXM_NX_ARP_SHA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xade0201->NXM_OF_ARP_SPA[],IN_PORT
 cookie=0x1000000000000, table=20, priority=190,arp actions=NORMAL
 cookie=0x1000000000000, table=20, priority=0 actions=drop
vmware@master:~$ 
</code></pre>

First two flows programmed in this table is to reply to ARP requests sent by Gateway0 for other Gateway IPs (basically other gateway interfaces on other nodes) Why ? Let’ s focus on second flow (10.222.1.1) first.  When a pod on worker 2 communicates to a Kubernetes Service (ClusterIP) then that needs to processed by local kube-proxy process to be load balanced (DNAT) to either a local or remote pod. If kube proxy picks a remote pod as the destination, then worker 2 needs to know how to get to that pod. Remote pod in this test runs on worker 1. Worker 1’ s pod subnet is 10.222.1.0 /24. On worker 2 there is an “on-link” route entry (slide 5) which makes the Kernel IP stack think that 10.222.1.0 is directly connected but the next hop for that route entry is 10.222.1.1. Hence worker 2 has to send an ARP entry for 10.222.1.1 (since it is a directly connected network – “on link”) to access the network 10.222.1.0. The ARP entry has a generic virtual mac aa:bb:cc:dd:ee:ff as the MAC address (instead of real MAC address of the respective Gateway0 interface of the destination node). (slide 5) This is the same on all nodes. This helps on the receiving end. Cause the worker 1 node OVS receives the traffic with this destination MAC and it can deliver the traffic right towards to the destination pod rather than the local Gateway0 interface.
Second flow entry in this flow table is for 10.222.0.1 (which is the pod subnet for master node) . Third flow instructs OVS to handle the remaining ARP requests as ordinary ARP traffic (which is Layer 2 broadcast).  Last rule is to drop all other type of traffic which got to this table. 
