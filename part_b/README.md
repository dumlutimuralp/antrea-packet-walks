# 4. SCENARIO 1 - Phase 1 - Frontend Pod to Service

The flow that will be explained in this section is shown below.

![](2020-09-16-17-49-42.png)

Basically this flow is frontend pod accessing the backendsvc service on TCP port 80. Backendsvc service is backed by backend1 and backend2 pod, as shown in kubectl outputs in section 3.2.

At this stage, the current flow has the following values in the Ethernet and IP headers.

- Source IP = 10.222.1.48 (Frontend pod IP)
- Destination IP = 10.104.65.133 (Backendsvc service IP)
- Source MAC = be:2c:bf:e4:ec:c5 (Frontend pod MAC)
- Destination MAC = 4e:99:08:c1:53:be (antrea-gw0 interface MAC on Worker 1)

This flow will be matched against a flow entry in each OVS Table, processed top to bottom in each individual table, based on the priority value of the flow entry in the table.

## 4.1 Classifier Table #0

Table #0 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=0 --no-stats
 cookie=0x1000000000000, priority=200,in_port="antrea-gw0" actions=load:0x1->NXM_NX_REG0[0..15],resubmit(,10)
 cookie=0x1000000000000, priority=200,in_port="antrea-tun0" actions=move:NXM_NX_TUN_METADATA0[28..31]->NXM_NX_REG9[28..31],load:0->NXM_NX_REG0[0..15],load:0x1->NXM_NX_REG0[19],resubmit(,30)
 cookie=0x1030000000000, priority=190,in_port="coredns--3e3abf" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)
 cookie=0x1030000000000, priority=190,in_port="antrea-o-830766" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)
 cookie=0x1030000000000, priority=190,in_port="backend1-bab86f" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)
 <b>cookie=0x1030000000000, priority=190,in_port="frontend-a3ba2f" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)</b>
 cookie=0x1000000000000, priority=0 actions=drop
vmware@master:~$
</code></pre>

This table is to classify the traffic by matching it on the ingress port and then setting the register NXM_NX_REG0[0..15] bits as following; "0" for tunnel, "1" for local gateway and "2" for local pod.  The current flow in this scenario, which is initiated from frontend pod and comes to OVS on the frontend pod interface, matches the <b>sixth</b> flow entry in the above output (which is highlighted). First action in this flow entry is to set the value of the register reg0[0..15] to "0x2", which means flow comes from a local pod. Second action in the same flow entry is to hand over the flow to next table which is Table 10. (resubmit(,10))  

So next stop is Table 10.

**Note :** "0x" means that it is hexadecimal. For example, 

- "0x2" is 2 × (16 to the power of 0) = 2 x 1 = 2
- or "0x20" is 2 x (16 to the power of 1) + 0 x (16 to the power of 0)  =  32 + 0 = 32

## 4.2 Spoofguard Table #10

Table #10 on Worker 1 node is shown below. "nw_src" refers to the IP address and "dl_src" refers to the MAC address of the pod connected to the respective OF port.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=10 --no-stats
 cookie=0x1000000000000, table=10, priority=200,ip,in_port="antrea-gw0" actions=resubmit(,30)
 cookie=0x1000000000000, table=10, priority=200,arp,in_port="antrea-gw0",arp_spa=10.222.1.1,arp_sha=4e:99:08:c1:53:be actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,arp,in_port="coredns--3e3abf",arp_spa=10.222.1.2,arp_sha=f2:82:cc:96:da:bd actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,arp,in_port="antrea-o-830766",arp_spa=10.222.1.3,arp_sha=6e:9e:5a:3e:3f:e8 actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,arp,in_port="backend1-bab86f",arp_spa=10.222.1.47,arp_sha=f2:32:d8:07:e2:a6 actions=resubmit(,20)
 <b>cookie=0x1030000000000, table=10, priority=200,arp,in_port="frontend-a3ba2f",arp_spa=10.222.1.48,arp_sha=be:2c:bf:e4:ec:c5 actions=resubmit(,20)</b>
 cookie=0x1030000000000, table=10, priority=200,ip,in_port="coredns--3e3abf",dl_src=f2:82:cc:96:da:bd,nw_src=10.222.1.2 actions=resubmit(,30)
 cookie=0x1030000000000, table=10, priority=200,ip,in_port="antrea-o-830766",dl_src=6e:9e:5a:3e:3f:e8,nw_src=10.222.1.3 actions=resubmit(,30)
 cookie=0x1030000000000, table=10, priority=200,ip,in_port="backend1-bab86f",dl_src=f2:32:d8:07:e2:a6,nw_src=10.222.1.47 actions=resubmit(,30)
 <b>cookie=0x1030000000000, table=10, priority=200,ip,in_port="frontend-a3ba2f",dl_src=be:2c:bf:e4:ec:c5,nw_src=10.222.1.48 actions=resubmit(,30)</b>
 cookie=0x1000000000000, table=10, priority=0 actions=drop
vmware@master:~$ 
</code></pre>

What this table does is verifying if the source IP and MAC of the the traffic matches the IP and MAC assigned to the Pod by Antrea CNI plugin during initial Pod wiring. It implements this check both for IP and ARP traffic.

Highlighted lines in the above output are the respective ARP and IP check entries for the OF port which the frontend pod is connected to. In this instance this is an IP flow from frontend pod to backend service hence the current flow will match the second line from the bottom. Notice, in the same flow entry, once the spoofguard check is successful then the flow is handed over to Table 30 (actions=resubmit(,30)). Table 30 is the next stop.

## 4.3 Conntrack Table #30

Table #30 on Worker 1 node is shown below.

Conntrack table' s job is to start tracking all the traffic. ("ct" action means connection tracking) Any flow that goes through this table will be in "tracked" (trk) state. In generic networking security terms this functionality makes the OVS a stateful connection aware component. Flow then gets handed over to the next table which is Table 31; as seen in the "actions" section of the flow entry (actions=ct(table=31,)). Next stop is Table 31.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=30 --no-stats
 cookie=0x1000000000000, table=30, priority=200,ip <b>actions=ct(table=31,zone=65520)</b>
vmware@master:~$ 
</code></pre>

**Note :** "Zone" is explained [here](https://github.com/vmware-tanzu/antrea/blob/master/docs/ovs-pipeline.md#conntracktable-30) as following : "A ct_zone is simply used to isolate connection tracking rules. It is similar in spirit to the more generic Linux network namespaces, but ct_zone is specific to conntrack and has less overhead." 

## 4.4 ConntrackState Table #31

Table #31 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=31 --no-stats
 cookie=0x1000000000000, table=31, priority=210,<b>ct_state=-new+trk</b>,ct_mark=0x20,ip,reg0=0x1/0xffff actions=resubmit(,40)
 cookie=0x1000000000000, table=31, priority=200,<b>ct_state=-new+trk</b>,ct_mark=0x20,ip actions=load:0x4e9908c153be->NXM_OF_ETH_DST[],resubmit(,40)
 cookie=0x1000000000000, table=31, priority=190,<b>ct_state=+inv+trk</b>,ip actions=drop
 <b>cookie=0x1000000000000, table=31, priority=0 actions=resubmit(,40)</b>
vmware@master:~$ 
</code></pre>

ConntrackState table processes all the flows that are in tracked state (basically which were handed over by the Conntrack table 30). The first and second flow entries shown above process the flows where the flow is NOT new AND tracked. (ct_state=-new means not new, +trk means being tracked) The third flow entry processes the flows where the respective flow is INVALID and TRACKED, basically it drops all those flows. 

The current flow is a completely NEW flow from frontend pod to backendsvc service. Hence the current flow will match the <b>last entry</b> in the flow table highlighted above. Notice the action in that flow entry is handing the flow over to the next table which is table 40. (resubmit(,40)) Next stop is Table 40.

## 4.5 DNAT Table #40

Table #40 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=40 --no-stats
 cookie=0x1040000000000, table=40, priority=200,ip,<b>nw_dst=10.96.0.0/12</b> actions=mod_dl_dst:4e:99:08:c1:53:be,load:0x2->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,105)
 cookie=0x1000000000000, table=40, priority=0 actions=resubmit(,50)
vmware@master:~$ 
</code></pre>

The table has only two flow entries. The first flow entry checks whether if the destination IP of the flow is part of the service CIDR range configured in the cluster (which is 10.96.0.0/12); if it does, then certain actions are taken on the flow to steer the flow to the antrea-gw0 interface on the Worker 1 node.  

The IP of the backendsvc service can be checked as below.

<pre><code>
vmware@master:~$ kubectl get svc
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
<b>service/backendsvc</b>   ClusterIP   <b>10.104.65.133</b>   <none>        80/TCP    2d1h
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   7d1h
vmware@master:~$ 
</code></pre>

As shown above, 10.104.65.133 falls into the service CIDR range <b>hence the current flow will match the first flow entry</b> in the table and multiple actions will be taken on the flow.

The first action specified in the flow entry is rewriting the destination mac of the flow with "4e:99:08:c1:53:be" (mod_dl_dst:4e:99:08:c1:53:be); which is the MAC address of the gw0 (antrea-gw0) interface of the Worker 1 node. 

The second action specified in the flow entry is to set the "NXM_NX_REG1" bit to "0x2" (by load:0x2 in hex, which is 2 in decimal). This register represents the egress OF Port ID of the flow. (ie which OF port should be the output port for this flow) And "2" is the OF Port ID for the antrea-gw0. The outputs and diagrams shown in Section 3.4 can be reviewed again to see OF Port ID.

The third action specified in the flow entry is setting the "NXM_NX_REG0[16]" to "1" (by load:0x1). This value in Reg0[16] means that OVS knows the destination MAC address. In other words this MAC address exists in OVS MAC address table (Table 80), which is explained in a seperate section. 

The fourth action in the flow entry is handing the flow over to the next table (resubmit(,105)) which is table 105. (resubmit(,105)) Next stop for this flow is Table 105.

**Note 1:** The flow is handed over to Table #105 and Egress tables (related to network policies), which are Table 45-49 and 50, are just bypassed, why ? Will be explained in a seperate section.

**Note 2:** About the first action mentioned in the flow entry above, since the destination MAC in this flow is already the MAC address of the antrea-gw0 interface (cause frontend pod' s gateway is gw0 interface IP) this rewrite action actually does not make any sense in this environment. Apparently this functionality was developed for a different use case and still exists but it actually does not affect anything. There is a PR for this which can be read [here](https://github.com/vmware-tanzu/antrea/issues/1264).


## 4.6 ConnTrackCommit Table #105

Table 105 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=105 --no-stats
 cookie=0x1000000000000, table=105, priority=200,ct_state=+new+trk,ip,reg0=0x1/0xffff actions=ct(commit,table=110,zone=65520,exec(load:0x20->NXM_NX_CT_MARK[]))
 cookie=0x1000000000000, table=105, priority=190,ct_state=+new+trk,ip actions=ct(commit,table=110,zone=65520)
 cookie=0x1000000000000, table=105, priority=0 actions=resubmit(,110)
vmware@master:~$ 
</code></pre>

This table is for committing all the new flows for tracking them. This action is specifically explained as "Commit the connection to the connection tracking module which will be stored beyond the lifetime of packet in the pipeline." [in OVS docs](https://docs.openvswitch.org/en/latest/tutorials/ovs-conntrack/).

The first flow entry checks whether if the flow is a new flow (+new) and if it is a tracked flow (+trk). The same flow entry also checks if the flow is coming from the gateway interface. This is represented by reg0=0x1/0xffff. Explanation of this is as following. The second part "0xffff" instructs OVS to verify the first 16 bits in reg0, meaning reg0[0..15] and then the first part "0x1" means the result of that check should be "1". So it basically checks reg0[0..15]. The Reg0[0..15] is set in Classifier Table 10 and it is set to "1" if the flow comes from antrea-gw0 interface. However this flow is not coming from the gateway interface, it is coming from a local pod interface (frontend pod interface on OVS) 

The second flow entry checks whether if the flow is a new flow (+new) and if it is a tracked flow (+trk). The action for this flow entry is committing this tracked flow to conntrack table (actions=ct(commit,..)) and then handing the flow over to the next table (,table=110).

In this case the current flow matches this <b>second</b> flow entry. It is a new flow from frontend pod to backendsvc service and it is being tracked (previously from Conntrack Table 30). Hence the current flow is committed to conntrack table (actions=ct(commit,..)) and then it is handed over to the next table (,table=110). So next stop is Table 110. 

## 4.7 L2ForwardingOut Table #110

Table 110 on Worker 1 node is shown below.

This table' s job is simple. First flow entry in this table first reads the value in register reg0[16]. The reg0[16] was set to "1" back in DNAT Table (Table 40) . "1" means the destination MAC address in the current flow is known to OVS and the flow should be able to get forwarded. (otherwise it would get dropped) The same flow entry has an action defined as "actions=output:NXM_NX_REG1[]". What this action does is it reads the value in "NXM_NX_REG1" to see which OF port is the output port for this flow. The value of REG1 was set to "2" back in DNAT Table 40. "2" is the OF Port ID of antrea-gw0 interface. Hence the OVS sends this flow onwards to the antrea-gw0 interface (as the output port on the OVS).

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=110 --no-stats
 cookie=0x1000000000000, table=110, priority=200,ip,reg0=0x10000/0x10000 actions=output:NXM_NX_REG1[]
 cookie=0x1000000000000, table=110, priority=0 actions=drop
vmware@master:~$
</code></pre>

**Note :** The second flow entry in this table obviously drops the flows which do not have their "reg0[16]" register set.

The logic of "reg0=-0x10000/0x10000" in the flow entry is that the first 0x10000 is the desired value of thie register. The second 0x10000 is the exact instructions on which specific bit(s) should be verified to match the desired value. Since "0x" is hexadecimal, below is a detailed explanation of the position of the bit that is verified. 

<pre><code>
23            16  15            8   7             0
0 0 0 0 0 0 0 1   0 0 0 0 0 0 0 0   0 0 0 0 0 0 0 0
</code></pre>

## 4.8 IPTables

At this stage, the Kernel IP stack of Worker 1 node receives the flow from antrea-gw0 interface and the flow will  is processed by iptables NAT rules. Iptables NAT rules on Worker 1 node are shown below.

Basically Kube Services chain "KUBE-SVC-EKL7ZEFK3VFJKKGJ" applies destination NAT on flows destined to backendsvc service IP and picks either backend1 or backend2 as the destination. This is typical Kube-proxy managed, iptables driven "ClusterIP" type of Kubernetes service functionality which is providing distributed load balancing for flows within a Kubernetes cluster.

<pre><code>
vmware@worker1:~$ sudo iptables -t nat -L -n
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination         
KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
DOCKER     all  --  0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
KUBE-POSTROUTING  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0           
ANTREA-POSTROUTING  all  --  0.0.0.0/0            0.0.0.0/0            /* Antrea: jump to Antrea postrouting rules */

Chain ANTREA-POSTROUTING (1 references)
target     prot opt source               destination         
MASQUERADE  all  --  10.222.1.0/24        0.0.0.0/0            /* Antrea: masquerade pod to external packets */ ! match-set ANTREA-POD-IP dst

Chain DOCKER (2 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           

Chain KUBE-KUBELET-CANARY (0 references)
target     prot opt source               destination         

Chain KUBE-MARK-DROP (0 references)
target     prot opt source               destination         
MARK       all  --  0.0.0.0/0            0.0.0.0/0            MARK or 0x8000

Chain KUBE-MARK-MASQ (19 references)
target     prot opt source               destination         
MARK       all  --  0.0.0.0/0            0.0.0.0/0            MARK or 0x4000

Chain KUBE-NODEPORTS (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  tcp  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/antrea-octant: */ tcp dpt:31067
KUBE-SVC-A2RN3UXPG7GRS3AU  tcp  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/antrea-octant: */ tcp dpt:31067

Chain KUBE-POSTROUTING (1 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0            mark match ! 0x4000/0x4000
MARK       all  --  0.0.0.0/0            0.0.0.0/0            MARK xor 0x4000
MASQUERADE  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service traffic requiring SNAT */

Chain KUBE-PROXY-CANARY (0 references)
target     prot opt source               destination         

Chain KUBE-SEP-2MIG7YSQRKRPLGGX (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.222.1.2           0.0.0.0/0            /* kube-system/kube-dns:dns */
DNAT       udp  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns */ udp to:10.222.1.2:53

<b>Chain KUBE-SEP-6PRWOLZVS5LKSHLK (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.222.1.47          0.0.0.0/0            /* default/backendsvc: */
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/backendsvc: */ tcp to:10.222.1.47:80</b>

Chain KUBE-SEP-BCZG4RHMLPD3XZC5 (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.222.2.2           0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */ tcp to:10.222.2.2:53

Chain KUBE-SEP-EVFFBIHEP2PDYG44 (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.79.1.200          0.0.0.0/0            /* default/kubernetes:https */
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/kubernetes:https */ tcp to:10.79.1.200:6443

Chain KUBE-SEP-NRWLC3D3JGTRYOSQ (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.79.1.202          0.0.0.0/0            /* kube-system/antrea: */
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/antrea: */ tcp to:10.79.1.202:10349

Chain KUBE-SEP-NS6VF4EO5FNJEZ3Z (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.222.1.3           0.0.0.0/0            /* kube-system/antrea-octant: */
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/antrea-octant: */ tcp to:10.222.1.3:80

Chain KUBE-SEP-QQKVVTQCCVWQJVWT (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.222.2.2           0.0.0.0/0            /* kube-system/kube-dns:dns */
DNAT       udp  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns */ udp to:10.222.2.2:53

<b>Chain KUBE-SEP-R5BOSGFC7D2XSIZA (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.222.2.34          0.0.0.0/0            /* default/backendsvc: */
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* default/backendsvc: */ tcp to:10.222.2.34:80</b>

Chain KUBE-SEP-U2DUVZDMAC5YMHOK (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.222.1.2           0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */ tcp to:10.222.1.2:53

Chain KUBE-SEP-Z2WIYZ27U5AUKQEZ (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.222.1.2           0.0.0.0/0            /* kube-system/kube-dns:metrics */
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:metrics */ tcp to:10.222.1.2:9153

Chain KUBE-SEP-ZP6MZIYCX7J4FSPR (1 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  all  --  10.222.2.2           0.0.0.0/0            /* kube-system/kube-dns:metrics */
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:metrics */ tcp to:10.222.2.2:9153

Chain KUBE-SERVICES (2 references)
target     prot opt source               destination         
KUBE-MARK-MASQ  tcp  -- !10.222.0.0/16        10.96.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
KUBE-MARK-MASQ  tcp  -- !10.222.0.0/16        10.109.110.219       /* kube-system/antrea-octant: cluster IP */ tcp dpt:80
KUBE-SVC-A2RN3UXPG7GRS3AU  tcp  --  0.0.0.0/0            10.109.110.219       /* kube-system/antrea-octant: cluster IP */ tcp dpt:80
KUBE-MARK-MASQ  udp  -- !10.222.0.0/16        10.96.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
KUBE-MARK-MASQ  tcp  -- !10.222.0.0/16        10.110.254.249       /* kube-system/antrea: cluster IP */ tcp dpt:443
KUBE-SVC-ACVYUMUVQZGITN4Q  tcp  --  0.0.0.0/0            10.110.254.249       /* kube-system/antrea: cluster IP */ tcp dpt:443
<b>KUBE-MARK-MASQ  tcp  -- !10.222.0.0/16        10.104.65.133        /* default/backendsvc: cluster IP */ tcp dpt:80
KUBE-SVC-EKL7ZEFK3VFJKKGJ  tcp  --  0.0.0.0/0            10.104.65.133        /* default/backendsvc: cluster IP */ tcp dpt:80</b>
KUBE-MARK-MASQ  tcp  -- !10.222.0.0/16        10.96.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  0.0.0.0/0            10.96.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
KUBE-MARK-MASQ  tcp  -- !10.222.0.0/16        10.96.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
KUBE-SVC-JD5MR3NA4I4DYORP  tcp  --  0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
KUBE-NODEPORTS  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL

Chain KUBE-SVC-A2RN3UXPG7GRS3AU (2 references)
target     prot opt source               destination         
KUBE-SEP-NS6VF4EO5FNJEZ3Z  all  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/antrea-octant: */

Chain KUBE-SVC-ACVYUMUVQZGITN4Q (1 references)
target     prot opt source               destination         
KUBE-SEP-NRWLC3D3JGTRYOSQ  all  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/antrea: */

<b>Chain KUBE-SVC-EKL7ZEFK3VFJKKGJ (1 references)
target     prot opt source               destination         
KUBE-SEP-6PRWOLZVS5LKSHLK  all  --  0.0.0.0/0            0.0.0.0/0            /* default/backendsvc: */ statistic mode random probability 0.50000000000
KUBE-SEP-R5BOSGFC7D2XSIZA  all  --  0.0.0.0/0            0.0.0.0/0            /* default/backendsvc: */</b>

Chain KUBE-SVC-ERIFXISQEP7F7OF4 (1 references)
target     prot opt source               destination         
KUBE-SEP-U2DUVZDMAC5YMHOK  all  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */ statistic mode random probability 0.50000000000
KUBE-SEP-BCZG4RHMLPD3XZC5  all  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns-tcp */

Chain KUBE-SVC-JD5MR3NA4I4DYORP (1 references)
target     prot opt source               destination         
KUBE-SEP-Z2WIYZ27U5AUKQEZ  all  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:metrics */ statistic mode random probability 0.50000000000
KUBE-SEP-ZP6MZIYCX7J4FSPR  all  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:metrics */

Chain KUBE-SVC-NPX46M4PTMTKRN6Y (1 references)
target     prot opt source               destination         
KUBE-SEP-EVFFBIHEP2PDYG44  all  --  0.0.0.0/0            0.0.0.0/0            /* default/kubernetes:https */

Chain KUBE-SVC-TCOU7JCQXEZGVUNU (1 references)
target     prot opt source               destination         
KUBE-SEP-2MIG7YSQRKRPLGGX  all  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns */ statistic mode random probability 0.50000000000
KUBE-SEP-QQKVVTQCCVWQJVWT  all  --  0.0.0.0/0            0.0.0.0/0            /* kube-system/kube-dns:dns */
vmware@worker1:~$ 
</code></pre>

Next phase is the flow being sent from backendsvc service to one of the backend pods backing that service and the processing of that flow is explained in the next section. 
