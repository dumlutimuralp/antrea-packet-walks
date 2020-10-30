# PART B

This section explains the packet flow between frontend and backend pods, which are **both on the same Kubernetes node**, in four main steps as shown below.

- [4. Frontend to Service](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#4-Frontend-to-service)
- [5. Service to Backend Pod](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#5-Service-to-backend-pod)
- [6. Backend Pod to Service](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#6-Backend-pod-to-service)
- [7. Service to Frontend](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#7-Service-to-frontend)

# 4. Frontend to Service
[Back to table of contents](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#part-b)

The flow that will be explained in this section is shown below.

![](2020-09-16-17-49-42.png)

As shown in [Part A Section 32](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#32-test-application), a simple "curl backendsvc" request on frontend pod would initiate this request. Below.

<pre><code>
vmware@master:~$ k exec -it frontend -- sh
/ # curl backendsvc
<b>OUTPUT OMITTED</b>
</code></pre>

This flow comes to OVS on the frontend pod port. Basically this flow is frontend pod accessing the backendsvc service on TCP port 80. Backendsvc service is backed by backend1 and backend2 pods, as shown in kubectl outputs in [Part A Section 3.2](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_a#32-test-application).

While performing "curl" on the frontend pod (as shown previously), use another SSH session to the master node to get tcpdump on the frontend pod, as shown below.
<pre><code>
vmware@master:~$ k exec -it frontend -- sh
/ # tcpdump -en
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
<b>OUTPUT OMITTED</b>
11:28:35.995562 <b>be:2c:bf:e4:ec:c5 > 4e:99:08:c1:53:be</b>, ethertype IPv4 (0x0800), length 74: <b>10.222.1.48.54444 > 10.104.65.133.80</b>: Flags [S], seq 486483242, win 64860, options [mss 1410,sackOK,TS val 782534940 ecr 0,nop,wscale 7], length 0
<b>OUTPUT OMITTED</b>
</code></pre>

As seen above, in the highlighted line of the tcpdump output, the flow has the following values in the Ethernet and IP headers.

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

This table is to classify the current flow by matching it on the ingress port and then setting the register NXM_NX_REG0[0..15] bits as following; "0" for tunnel, "1" for local gateway and "2" for local pod.  The current flow in this scenario, which is initiated from frontend pod and comes to OVS on the frontend pod interface, matches the <b>sixth</b> flow entry in the above output (which is highlighted). First action in this flow entry is to set the value of the register reg0[0..15] to "0x2", which means flow comes from a local pod. Second action in the same flow entry is to hand the flow over to next table which is Table 10. (resubmit(,10))  

So next stop is Table 10.

**Note :** "0x" means that it is hexadecimal. For example, 

- "0x2" is 2 × (16 to the power of 0) = 2 x 1 = 2 (in decimal)
- or "0x20" is 2 x (16 to the power of 1) + 0 x (16 to the power of 0)  =  32 + 0 = 32 (in decimal)

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

What this table does is verifying if the source IP and MAC of the current flow matches the IP and MAC assigned to the Pod by Antrea CNI plugin during initial Pod connectivity. It implements this check both for IP and ARP traffic.

Highlighted lines in the above output are the respective ARP and IP check entries for the OF port which the frontend pod is connected to. In this instance this is an IP flow from frontend pod to backend service hence the current flow will match the **tenth flow entry**, second line from the bottom. Notice, in the same flow entry, the action is to hand the flow over to Table 30 (actions=resubmit(,30)). Table 30 is the next stop.

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

Although it is shown back in [Part A Section 3.2](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_a#32-test-application), to verify again, the IP of the backendsvc service can be checked as shown below.

<pre><code>
vmware@master:~$ kubectl get svc
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
<b>service/backendsvc</b>   ClusterIP   <b>10.104.65.133</b>   <none>        80/TCP    2d1h
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   7d1h
vmware@master:~$ 
</code></pre>

As shown above, 10.104.65.133 falls into the service CIDR range <b>hence the current flow will match the first flow entry</b> in the table and multiple actions will be taken on the flow.

The first action specified in the flow entry is rewriting the destination mac of the flow with "4e:99:08:c1:53:be" (mod_dl_dst:4e:99:08:c1:53:be); which is the MAC address of the gw0 (antrea-gw0) interface of the Worker 1 node. 

The second action specified in the flow entry is to set the "NXM_NX_REG1" bit to "0x2" (by load:0x2 in hex, which is 2 in decimal). This register represents the egress OF Port ID which this flow will be sent through. And "2" is the OF Port ID for the antrea-gw0. The outputs and diagrams shown in [Part A Section 3.4](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_a#34-identifying-ovs-port-ids-of-port-ids) can be reviewed again to see OF Port ID.

The third action specified in the flow entry is setting the "NXM_NX_REG0[16]" to "0x1" (by load:0x1 in hex, which is 1 in decimal). This value in Reg0[16] means that OVS knows the destination MAC address. In other words this MAC address exists in OVS MAC address table (Table 80), which is explained in a seperate section. 

The fourth action in the flow entry is handing the flow over to the next table which is table 105 (resubmit(,105)). Next stop for this flow is Table 105.

**Note 1:** The flow is handed over to Table #105 and Egress tables (related to network policies), which are Table 45,49 and 50, are just bypassed, why ? Will be explained in an upcoming section.

**Note 2:** About the first action mentioned in the flow entry above, since the destination MAC in this flow is already the MAC address of the antrea-gw0 interface (cause frontend pod' s default gateway is gw0 interface IP) this rewrite action actually does not make any sense in this environment. Apparently this functionality was developed for a different use case and still exists but it actually does not affect anything. There is a PR for this which can be read [here](https://github.com/vmware-tanzu/antrea/issues/1264).

**Note 3:** The Service CIDR range (10.96.0.0/12) is aligned to the kubeadm default service cidr range. If a specific range is used in "--service-cluster-ip-range" for kube-apiserver then the "serviceCIDR" variable in antrea configmap should also be configured with the same range so that the respective flow entry in Table 40 gets configured with the correct range.


## 4.6 ConnTrackCommit Table #105

Table 105 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=105 --no-stats
 cookie=0x1000000000000, table=105, priority=200,ct_state=+new+trk,ip,reg0=0x1/0xffff actions=ct(commit,table=110,zone=65520,exec(load:0x20->NXM_NX_CT_MARK[]))
 <b>cookie=0x1000000000000, table=105, priority=190,ct_state=+new+trk,ip actions=ct(commit,table=110,zone=65520)</b>
 cookie=0x1000000000000, table=105, priority=0 actions=resubmit(,110)
vmware@master:~$ 
</code></pre>

This table is for committing all the new flows for tracking them. This action, ct(commit,), is specifically explained as "Commit the connection to the connection tracking module which will be stored beyond the lifetime of packet in the pipeline." [in OVS docs](https://docs.openvswitch.org/en/latest/tutorials/ovs-conntrack/).

The first flow entry checks whether if the flow is a new flow (+new) and if it is a tracked flow (+trk). The same flow entry also checks if the flow is coming from the gateway interface. This is represented by reg0=0x1/0xffff. Explanation of this is as following. The second part "0xffff" instructs OVS to verify the first 16 bits in reg0, meaning reg0[0..15] and then the first part "0x1" means the result of that check should be "1" in decimal. So it basically checks reg0[0..15]. The Reg0[0..15] is set by Classifier Table 10 and it is set to "1" if the flow comes from antrea-gw0 interface. 

The second flow entry checks whether if the flow is a new flow (+new) and if it is a tracked flow (+trk). 

In this case the current flow matches this <b>second</b> flow entry. It is not coming from the gateway interface; it is coming from a local pod interface (frontend pod interface on OVS).   It is a new flow from frontend pod to backendsvc service and it is being tracked (previously from Conntrack Table 30 and 31). Hence the current flow is committed to conntrack table (actions=ct(commit,..)) and then it is handed over to the next table (,table=110). So next stop is Table 110. 

## 4.7 L2ForwardingOut Table #110

Table 110 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=110 --no-stats
 cookie=0x1000000000000, table=110, priority=200,ip,reg0=0x10000/0x10000 actions=output:NXM_NX_REG1[]
 cookie=0x1000000000000, table=110, priority=0 actions=drop
vmware@master:~$
</code></pre>

This table' s job is simple. First flow entry in this table first reads the value in register reg0[16]. If the value of this register is "1" in decimal, that means the destination MAC address is known to OVS and the flow should be able to get forwarded (otherwise it would get dropped). The same flow entry has an action defined as "actions=output:NXM_NX_REG1[]". What this action does is it reads the value in "NXM_NX_REG1" to determine the OF port this flow will be sent through and then sends the flow onwards to that port.

The current flow' s reg0[16] bit was set to "0x1" (1 in decimal)  back in DNAT Table 40 and also the value of REG1 was set to "0x2" (2 in decimal) back in DNAT Table 40. "2" is the OF Port ID of antrea-gw0 interface. **Hence the current flow is sent onwards to the antrea-gw0 by OVS.**

**Note :** The second flow entry in this table obviously drops the flows which do not have their "reg0[16]" register set.

The logic of "reg0=-0x10000/0x10000" in the flow entry is that the first 0x10000 is the desired value of this register. The second 0x10000 is the exact instructions on which specific bit(s) should be verified to match the desired value. Since "0x" is hexadecimal, below is a detailed explanation of the position of the bit that is verified. 

<pre><code>
23              16  15               8   7               0
0 0 0 0 | 0 0 0 1   0 0 0 0 |  0 0 0 0   0 0 0 0 | 0 0 0 0
</code></pre>

So bit 16 must be "1", and that is being verified in "reg0". The first four bits on the left hand side is not worth to mention hence the desired value and actual value are both shown as "0x10000". 


## 4.8 IPTables

At this stage, the Kernel IP stack of Worker 1 node receives the flow from antrea-gw0 interface and the flow is processed by kube-proxy managed iptables NAT rules. Iptables NAT rules on Worker 1 node are shown below.

Basically Kube Services chain **"KUBE-SVC-EKL7ZEFK3VFJKKGJ"** applies destination NAT on flows destined to backendsvc service IP and picks either backend1 pod or backend2 pod as the destination. This is typical kube-proxy managed, iptables driven "ClusterIP" type of Kubernetes service functionality which is providing distributed load balancing for flows within a Kubernetes cluster.

- "-t nat" pulls the NAT table 
- "-L" lists all the rules in the given table
- "-n" displayes IP address and port numbers (rather than hostnames and service names)

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

Next step is the flow to be sent from backendsvc service to one of the backend pods backing that service and the processing of that flow and that is explained in the next section. 

# 5. Service to Backend Pod
[Back to table of contents](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#part-b)

**The assumption at this stage is, in the previous step, kube-proxy managed NAT rules in iptables translated the backendsvc service IP (10.104.65.133) to the backend1 pod' s IP (10.222.1.47, which is on the same node)** to service the request that came from the frontend pod in the previous section. The flow that will be explained in this section is shown below.

**Note 1:** The other scenario, in which iptables translates the flow to backend2 pod IP, is explained in [Part C](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_c).

![](2020-09-21-18-05-57.png)

To verify the Ethernet and IP headers of this flow, a quick tcpdump on the antrea-gw0 interface of the Worker 1 node would reveal the source and destination IP/MAC of this flow. 

<pre><code>
vmware@master:~$ kubectl exec -it frontend -- sh
/ # curl backendsvc
Praqma Network MultiTool (with NGINX) - backend1 - 10.222.1.47/24
</code></pre>

While performing "curl backendsvc" on frontend pod (shown above), connect to the Kubernetes Worker 1 node and get tcpdump. As shown below.

<pre><code>
vmware@worker1:~$ sudo tcpdump -en -i antrea-gw0 host 10.222.1.47
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on antrea-gw0, link-type EN10MB (Ethernet), capture size 262144 bytes
<b>OUTPUT OMITTED</b>
21:37:41.789071 <b>4e:99:08:c1:53:be > f2:32:d8:07:e2:a6</b>, ethertype IPv4 (0x0800), length 74: <b>10.222.1.48.53128 > 10.222.1.47.80</b>: Flags [S], seq 1929465462, win 64860, options [mss 1410,sackOK,TS val 3904428111 ecr 0,nop,wscale 7], length 0
<b>OUTPUT OMITTED</b>
</code></pre>

**Note 2:** For simplicity, the ARP requests/replies between antrea-gw0, frontend pod and backend1 pod are not shown in the above output. ARP process will be explained in [Part D Section 12](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_d).

**Note 3 :** Notice that not only the destination IP but also the source and destination MAC addresses also have changed (from the previous step, Section 4).

Basically the current flow has the following values in the Ethernet and IP headers.

- Source IP = 10.222.1.48 (frontend pod IP)
- Destination IP = 10.222.1.47 (backend1 pod IP)
- Source MAC = 4e:99:08:c1:53:be (antrea-gw0 interface MAC on Worker 1)
- Destination MAC = f2:32:d8:07:e2:a6 (backend1 Pod MAC) 

This flow will be matched against a flow entry in each OVS Table, processed top to bottom in each individual table, based on the priority value of the flow entry in the table.

## 5.1 Classifier Table #0

Table #0 on Worker 1 node is shown below.  

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=0 --no-stats
 <b>cookie=0x1000000000000, priority=200,in_port="antrea-gw0" actions=load:0x1->NXM_NX_REG0[0..15],resubmit(,10)</b>
 cookie=0x1000000000000, priority=200,in_port="antrea-tun0" actions=move:NXM_NX_TUN_METADATA0[28..31]->NXM_NX_REG9[28..31],load:0->NXM_NX_REG0[0..15],load:0x1->NXM_NX_REG0[19],resubmit(,30)
 cookie=0x1030000000000, priority=190,in_port="coredns--3e3abf" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)
 cookie=0x1030000000000, priority=190,in_port="antrea-o-830766" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)
 cookie=0x1030000000000, priority=190,in_port="backend1-bab86f" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)
 cookie=0x1030000000000, priority=190,in_port="frontend-a3ba2f" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)
 cookie=0x1000000000000, priority=0 actions=drop
vmware@master:~$
</code></pre>

This table is to classify the current flow by matching it on the ingress port and then setting the register NXM_NX_REG0[0..15] bits as following; "0" for tunnel, "1" for local gateway and "2" for local pod. 

The current flow came from antrea-gw0 interface hence it matches the <b>first</b> flow entry in the above output (which is highlighted). The first action in this flow entry is to set the register reg0[0..15] to "0x1" since the flow comes from the local gateway. The second action in this flow entry is to hand the flow over to Table 10. (resubmit(,10)) Hence next stop is Table 10.

## 5.2 Spoofguard Table #10

Table #10 on Worker 1 node is shown below. "nw_src" refers to the IP address and "dl_src" refers to the MAC address of the pod connected to the respective OF port.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=10 --no-stats
 <b>cookie=0x1000000000000, table=10, priority=200,ip,in_port="antrea-gw0" actions=resubmit(,30)</b>
 cookie=0x1000000000000, table=10, priority=200,arp,in_port="antrea-gw0",arp_spa=10.222.1.1,arp_sha=4e:99:08:c1:53:be actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,arp,in_port="coredns--3e3abf",arp_spa=10.222.1.2,arp_sha=f2:82:cc:96:da:bd actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,arp,in_port="antrea-o-830766",arp_spa=10.222.1.3,arp_sha=6e:9e:5a:3e:3f:e8 actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,arp,in_port="backend1-bab86f",arp_spa=10.222.1.47,arp_sha=f2:32:d8:07:e2:a6 actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,arp,in_port="frontend-a3ba2f",arp_spa=10.222.1.48,arp_sha=be:2c:bf:e4:ec:c5 actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,ip,in_port="coredns--3e3abf",dl_src=f2:82:cc:96:da:bd,nw_src=10.222.1.2 actions=resubmit(,30)
 cookie=0x1030000000000, table=10, priority=200,ip,in_port="antrea-o-830766",dl_src=6e:9e:5a:3e:3f:e8,nw_src=10.222.1.3 actions=resubmit(,30)
 cookie=0x1030000000000, table=10, priority=200,ip,in_port="backend1-bab86f",dl_src=f2:32:d8:07:e2:a6,nw_src=10.222.1.47 actions=resubmit(,30)
 cookie=0x1030000000000, table=10, priority=200,ip,in_port="frontend-a3ba2f",dl_src=be:2c:bf:e4:ec:c5,nw_src=10.222.1.48 actions=resubmit(,30)
 cookie=0x1000000000000, table=10, priority=0 actions=drop
vmware@master:~$ 
</code></pre>

What this table does is verifying if the source IP and MAC of the current flow matches the IP and MAC assigned to the Pod by Antrea CNI plugin during initial Pod connectivity. It implements this check both for IP and ARP traffic.

Since the flow comes from the antrea-gw0 interface, the flow will match the <b>first</b> flow entry in this table. The source IP of the current flow, which is coming from antrea-gw0 interface, is still the frontend pod IP (requestor of the original flow), hence spoofguard does not do any checks in this instance. The only action specified in the first flow entry is handing the flow over to Table 30 (actions=goto_table:30). So next stop is Table 30.

**Note :** Spoofguard does not do any checks for IP packets on the antrea-gw0 port. However it still checks the ARP flows on that port. Second entry in the table is used for that purpose. It basically checks the ARP requests/replies sent by the antrea-gw0 interface.

## 5.3 ConntrackTable Table #30

Table #30 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=30 --no-stats
 cookie=0x1000000000000, table=30, priority=200,ip <b>actions=ct(table=31</b>,zone=65520)
vmware@master:~$ 
</code></pre>

Conntrack table' s job is to start tracking all the traffic. ("ct" action means connection tracking). Any flow that goes through this table will be in "tracked" (trk) state. In generic networking security terms this functionality makes the OVS a stateful connection aware component. Flow then gets handed over to the next table which is Table 31; as seen in the "actions" section of the flow entry. (actions=ct(table=31,)) So next stop is Table 31.

**Note :** Zone ID is explained [here](https://github.com/vmware-tanzu/antrea/blob/master/docs/ovs-pipeline.md#conntracktable-30) as "A ct_zone is simply used to isolate connection tracking rules. It is similar in spirit to the more generic Linux network namespaces, but ct_zone is specific to conntrack and has less overhead." 

## 5.4 ConnTrackState Table #31

Table #31 on Worker 1 is node shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=31 --no-stats
 cookie=0x1000000000000, table=31, priority=210,<b>ct_state=-new+trk</b>,ct_mark=0x20,ip,reg0=0x1/0xffff actions=resubmit(,40)
 cookie=0x1000000000000, table=31, priority=200,<b>ct_state=-new+trk</b>,ct_mark=0x20,ip actions=load:0x4e9908c153be->NXM_OF_ETH_DST[],resubmit(,40)
 cookie=0x1000000000000, table=31, priority=190,<b>ct_state=+inv+trk</b>,ip actions=drop
 <b>cookie=0x1000000000000, table=31, priority=0 actions=resubmit(,40)</b>
vmware@master:~$ 
</code></pre>

ConntrackState table processes all the flows that are in tracked state (basically which were handed over by the Conntrack table 30). The first and second flow entries shown above process the flows where the flow is NOT new AND tracked. (ct_state=-new means not new, +trk means being tracked) The third flow entry processes the flows where the respective flow is INVALID and TRACKED, basically it drops all those flows.

The current flow is a completely NEW flow - from frontend pod to backend1 pod. Hence the flow matches the <b>last entry</b> in the flow table highlighted above. Notice the action in the same flow entry is handing the flow over to the next table which is table 40 (resubmit(,40)). So next stop is Table 40.

**Note :** The current flow is actually the continuity of the previous flow, which was from frontend pod to backendsvc service. However that previous flow has been subject to DNAT process by iptables in Section 4.8. The destination IP of that previous flow has been changed (to backend1 pod IP) so the current flow, whose source IP is frontend pod IP and destination IP is now the backend1 pod IP, is a new flow. Hence it matches the last flow entry in Table 31. 

## 5.5 DNAT Table #40

Table #40 on Worker 1 is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=40 --no-stats
 cookie=0x1040000000000, table=40, priority=200,ip,w_dst=10.96.0.0/12 actions=mod_dl_dst:4e:99:08:c1:53:be,load:0x2->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,105)
 <b>cookie=0x1000000000000, table=40, priority=0 actions=resubmit(,50)</b>
vmware@master:~$ 
</code></pre>

This table in essence checks whether if the flow is destined to a Kubernetes service so that it can redirect the flow to the antrea-gw0. 

The table has only two flow entries. The first flow entry checks whether if the destination IP of the flow is part of the service CIDR range configured in the cluster (which is 10.96.0.0/12); if it does, then certain actions are taken on the flow to steer the flow to the antrea-gw0 interface on the Worker 1 node.  

The destination IP of the current flow is backend1 pod IP (10.222.1.47) and it does not fall into the service CIDR range in the first flow entry in Table 40. Hence the current flow will match the second/last entry. The action specified in the last flow entry is to basically hand the flow over to Table 50 (actions=resubmit(,50)). So next stop is Table 50.

**Note :** In the OVS Pipeline diagram [here](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#2-ovs-pipeline), there are tables 45,49 before Table50. However those tables are in use only when Antrea Network Policy feature of Antrea is used. In this Antrea environment, it is not used. 

## 5.6 EgressRule Table #50

At this stage, the flow is in Table 50. 

Table 50 has flow entries which correspond to the egress rules configured in all the Kubernetes Network Policies applied to all the pods on Worker 1 node. There is a frontend pod and a backend1 pod on the Worker 1 node. "frontendpolicy" is applied to frontend pod and "backendpolicy" is applied to backend1 pod and backend2 pod (on Worker2) . 

In the previous section (Section 4) the flow could not be matched against this table, instead it bypassed all the EgressRule tables. Because that flow' s destination IP was still the backendsvc service IP and Kubernetes Network Policy would not make an accurate check in that case and legitimate flows would have been blocked. This is the reason as to why the flow in Section 4 bypassed all the Egress Rule tables and got redirected to antrea-gw0 interface onwards to Kernel IP stack for iptables rule processing. Only after kube-proxy managed iptables rules applied DNAT on that flow (in Section 4.8), which essentially translated the destination backendsvc service IP to one of the backend pods IP, then the flow now can be processed by this Table 50 as will be explained in this step. 

The content of the "frontendpolicy" is shown below.

<pre><code>
vmware@master:~$ kubectl get netpol frontendpolicy
NAME             POD-SELECTOR    AGE
frontendpolicy   role=frontend   6d5h
vmware@master:~$ kubectl get netpol frontendpolicy -o yaml
apiVersion: networking.Kubernetes.io/v1
kind: NetworkPolicy
metadata:
  name: <b>frontendpolicy</b>
  namespace: default
spec:
  <b>egress:
  - ports:
    - port: 80
      protocol: TCP
    to:
    - podSelector:
        matchLabels:
          role: backend
  - ports:
    - port: 53
      protocol: UDP</b>
  ingress:
  - ports:
    - port: 80
      protocol: TCP
  podSelector:
    matchLabels:
      role: frontend
  policyTypes:
  - Egress
  - Ingress
vmware@master:~$ 
vmware@master:~$ k describe netpol frontendpolicy
Name:         frontendpolicy
Namespace:    default
Created on:   2020-09-14 15:27:33 +0000 UTC
Labels:       <none>
Annotations:  Spec:
  PodSelector:     role=frontend
  Allowing ingress traffic:
    To Port: 80/TCP
    From: <any> (traffic not restricted by source)
  <b>Allowing egress traffic:
    To Port: 80/TCP
    To:
      PodSelector: role=backend
    ----------
    To Port: 53/UDP
    To: <any> (traffic not restricted by source)</b>
  Policy Types: Egress, Ingress
vmware@master:~$ 
vmware@master:~$ 
</code></pre>

The egress section of this network policy is highlighted in the above output. In this network policy, frontend pod can initiate HTTP to backend pods and it can initiate DNS to anywhere. All other egress traffic from frontend pod will be denied. 

**Note :** The reason DNS is also allowed is, when the frontend pod sends a request to the backendsvc service by its name, a DNS query is sent by frontend pod to Kubernetes DNS so that frontend pod can call out one of the backend pods by their IPs, hence DNS has to be allowed to make this work. **As mentioned before, this article focuses on the actual application flow which occurs after the DNS query. In this section the flow from frontend pod to backend1 pod on TCP port 80 is explained.**

The Table 50 on Worker 1 node is shown below. 

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=50 --no-stats
 cookie=0x1000000000000, table=50, priority=210,ct_state=-new+est,ip actions=resubmit(,70)
 cookie=0x1050000000000, table=50, priority=200,ip actions=conjunction(2,2/3)
 cookie=0x1050000000000, table=50, priority=200,udp,tp_dst=53 actions=conjunction(2,3/3)
 cookie=0x1050000000000, table=50, priority=200,tcp,tp_dst=80 actions=conjunction(1,3/3)
 cookie=0x1050000000000, table=50, priority=200,ip,nw_src=10.222.1.48 actions=conjunction(1,1/3),conjunction(2,1/3)
 cookie=0x1050000000000, table=50, priority=200,ip,nw_src=10.222.1.47 actions=conjunction(5,1/2)
 cookie=0x1050000000000, table=50, priority=200,ip,nw_dst=10.222.2.34 actions=conjunction(1,2/3)
 cookie=0x1050000000000, table=50, priority=200,ip,nw_dst=10.222.1.47 actions=conjunction(1,2/3)
 cookie=0x1050000000000, table=50, priority=190,conj_id=2,ip actions=load:0x2->NXM_NX_REG5[],resubmit(,70)
 cookie=0x1050000000000, table=50, priority=190,conj_id=1,ip actions=load:0x1->NXM_NX_REG5[],resubmit(,70)
 cookie=0x1050000000000, table=50, priority=190,conj_id=5,ip actions=load:0x5->NXM_NX_REG5[],resubmit(,70)
 cookie=0x1000000000000, table=50, priority=0 actions=resubmit(,60)
vmware@master:~$ 
</code></pre>

First thing to understand from above table is OVS uses "conjunctive" match across multiple fields alongside "conjunction" action. This enables OVS to optimize policy implementation without consuming too many flow entries.

The first flow entry in Table 50 above checks whether if the flow is an already established flow (-new,+est); if it is then there is no need to process the flow against the remaining flow entries in this table since Kubernetes Network Policy is STATEFUL by nature. However the current flow is a NEW flow hence it does NOT match this first flow entry.

The second flow entry matches against a given source or destination IP set. But there is no specific IP that this flow entry checks upon apparently. Same flow entry has a conjunction action with a conjunction id of "2".

The third flow entry matches against destination protocol, which is UDP 53 (DNS) in this case. Same flow entry has a conjunction action with a conjunction id of "2". 

The fourth flow entry matches against destination protocol, which is TCP 80 (HTTP) in this case. Same flow entry has a conjunction action with a conjunction id of "1".

The fifth flow entry matches against a specific source IP, which is frontend pod IP - 10.222.1.48. The same flow entry has a conjunction action with both an id of "1" and "2".

**Note :** The x/y notation in the conjunctions are to represent whether if the conjunction has multiple conditions to match and which condition the current flow represents.For example 3,2/3 means the given flow entry is the second out of three conjunctions for the conjunction id 3.

<b>So conjunction 2 is fully implemented by second, third and fifth flow entries.</b> Conjunction 2 checks if the source IP address is 10.222.1.48 and if the destination protocol is UDP 53. This corresponds to the DNS specific rule in the egress section of the Kubernetes network policy named as "frontendpolicy" shown earlier. DNS is allowed egress to any destination IP hence the second flow entry above does not have any specific IP to match against. 

The sixth flow entry matches against source IP, which is 10.222.1.47, which is the backend1 pod IP in this case. The same flow entry has a conjunction action with a conjunction id of "5".

The seventh and eight flow entries match against the destination IP to see if it is either of the IPs; 10.222.1.47 or 10.222.2.34. These flow entries has a conjunction action with a conjunction id of "1".

<b> So conjunction 1 is fully implemented by fourth, fifth, seventh and eighth flow entries.</b> Conjunction 1 checks if the source IP is 10.222.1.48 and then if the destination is TCP 80 and then if the destination IP is either 10.222.1.47 or 10.222.2.34. This corresponds to TCP 80 specific rule in the egress section of the Kubernetes network policy named as "frontendpolicy".

**Note :** The sixth flow entry mentioned above has got nothing to do with Kubernetes Network Policy "frontendpolicy", it actually corresponds to the egress rule used in the Kubernetes Network Policy "backendpolicy" which is not relevant here.

The ninth flow entry defines two actions for conjunction 2 as soon as all fields of conjunction 2 (which is explained above with second, third and fifth flow entries) is matched. First action is to set the register NXM_NX_REG5 with the conjunction id of "2" (0x2). The second action in the same flow entry is to hand the flow over to Table 70. (resubmit(,70))

The tenth flow entry defines two actions for conjunction 1 as soon as all fields of conjunction 1 (which is explained above with fourth, fifth,seventh and eigth flow entries) is matched. First action is to set the register NXM_NX_REG5 with the conjunction id of "1" (0x1). The second action in the same flow entry is to hand the flow over to Table 70. (resubmit(,70))

**Note :** NXM_NX_REG5 register is used to cache the conjunction id which is mapped to the egress Network Policy Rule in Antrea agent and also then written back to the Antrea custom resource definition in Kubernetes API for another feature of Antrea called "traceflow". More info on it can be found [here](https://github.com/vmware-tanzu/antrea/blob/master/docs/traceflow-guide.md).

**The current flow, which is from frontend pod IP to backend1 pod IP on protocol TCP 80, matches conjunction 1 (defined in tenth flow entry) and it will be handed over to Table 70. So next stop is Table 70. (explained in Section 5.7)**

For reference, remaining flow entries are explained below : 

The eleventh flow entry also defines two actions for conjunction 5 as soon as all fields of conjunction 5 is matched. As mentioned before, this conjunction id has got nothing to do with "frontendpolicy" and it is not relevant here. It actually corresponds to the egress rules configured in "backendpolicy".

Last flow entry in this table defines that if the flow does not match any of the above entries then the flow is handed over to Table 60 which is EgressDefaultTable (resubmit(,60)). Table 60 is for isolation rules. Basically when a Kubernetes Network Policy is applied to a pod, the flows which do not match any of the entries in Table 50 will be dropped by Table 60.

For reference, EgressDefault Table #60 on Worker 1 node is shown  below. As the current flow matches conjunction 1 in Table 50, Table 60 will be bypassed.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=60 --no-stats
 cookie=0x1000000000000, table=60, priority=200,ip,nw_src=10.222.1.48 actions=drop
 cookie=0x1000000000000, table=60, priority=200,ip,nw_src=10.222.1.47 actions=drop
 cookie=0x1000000000000, table=60, priority=0 actions=resubmit(,70)
vmware@master:~$
</code></pre>

Reason there are two source IPs in two different flow entries here is, there is an egress rule used in two different Kubernetes Network Policies; one is "frontendpolicy" applied to frontend pod, the other is "backendpolicy" applied to backend pods. Each of the first two flow entries in this table used to deny the traffic from the respective pod running on Worker 1 node. 

The last flow entry in Table 60 basically hands all other flows (which did not match any of the conjunctions in Table 50 nor the drop flow entries in Table 60) over to the next table , Table 70.

## 5.7 L3Forwarding Table #70

The Table 70 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=70 --no-stats
 cookie=0x1000000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.1 actions=mod_dl_dst:4e:99:08:c1:53:be,resubmit(,80)
 cookie=0x1030000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.2 actions=mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:f2:82:cc:96:da:bd,dec_ttl,resubmit(,80)
 cookie=0x1030000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.3 actions=mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:6e:9e:5a:3e:3f:e8,dec_ttl,resubmit(,80)
 cookie=0x1030000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.47 actions=mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:f2:32:d8:07:e2:a6,dec_ttl,resubmit(,80)
 cookie=0x1030000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.48 actions=mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:be:2c:bf:e4:ec:c5,dec_ttl,resubmit(,80)
 cookie=0x1020000000000, table=70, priority=200,ip,nw_dst=10.222.0.0/24 actions=dec_ttl,mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:aa:bb:cc:dd:ee:ff,load:0x1->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],load:0xa4f01c8->NXM_NX_TUN_IPV4_DST[],resubmit(,105)
 cookie=0x1020000000000, table=70, priority=200,ip,nw_dst=10.222.2.0/24 actions=dec_ttl,mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:aa:bb:cc:dd:ee:ff,load:0x1->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],load:0xa4f01ca->NXM_NX_TUN_IPV4_DST[],resubmit(,105)
 <b>cookie=0x1000000000000, table=70, priority=0 actions=resubmit(,80)</b>
vmware@master:~$ 
</code></pre>

Basically each flow entry in this flow table checks either the destination IP address or destination MAC address (in some entries both) to make a forwarding decision. 

The current flow' s source and destination MAC and IP address values are still as they are shown back in Section 5. Shown below again. 

- Source IP = 10.222.1.48 (frontend pod IP)
- Destination IP = 10.222.1.47 (backend1 pod IP)
- Source MAC = 4e:99:08:c1:53:be (antrea-gw0 interface MAC on Worker 1)
- Destination MAC = f2:32:d8:07:e2:a6 (backend1 Pod MAC) 

Based on the flow' s source and destination MAC/IP values the flow matches the last flow entry (eigth entry) in Table 70 (since the prior flow entries match against a different destination MAC). The destination IP and MAC is local to Worker 1 node and clearly there is no L3 forwarding needed. The action in the last flow entry is "resubmit(,80)" which basically hands the flow over to Table 80. Hence next stop is Table 80.

**Note :** The first five flow entries in this table are related to ARP processing (with "dl_dst=aa:bb:cc:dd:ee:ff") and will be explained in [Part D Section 12](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_d). The sixth and seventh flow entries in this table are for inter node flow patterns and it will be explained in Part C Section 9.

## 5.8 L2ForwardingCalc Table #80

Table 80 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=80 --no-stats
 cookie=0x1000000000000, table=80, priority=200,dl_dst=4e:99:08:c1:53:be actions=load:0x2->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)
 cookie=0x1030000000000, table=80, priority=200,dl_dst=f2:82:cc:96:da:bd actions=load:0x3->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)
 cookie=0x1030000000000, table=80, priority=200,dl_dst=6e:9e:5a:3e:3f:e8 actions=load:0x4->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)
 <b>cookie=0x1030000000000, table=80, priority=200,dl_dst=f2:32:d8:07:e2:a6 actions=load:0x30->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)</b>
 cookie=0x1030000000000, table=80, priority=200,dl_dst=be:2c:bf:e4:ec:c5 actions=load:0x31->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)
 cookie=0x1000000000000, table=80, priority=0 actions=resubmit(,90)
vmware@master:~$ 
</code></pre>

What this table does is not that different than a typical IEEE 802.1d transparent bridge. This is basically the MAC address table of the OVS. Based on the destination MAC address of the flow OVS decides which OF Port the flow should be sent to. 

Each flow entry in this table sets two registers, both of which mentioned in earlier sections, will be explained here once again. 

- Reg1 is used to store OF port ID of the the OVS port which the flow should be sent to. Based on the destination MAC address of the flow this register is set with the respective OF port ID.  This register will be used later on in Table 110 (L2ForwardingOut Table).
- The way Reg0[16] is used is that if it is set to "1" then that indicates that the given flow has a matching destination address in this table, which is known to OVS, and it should be forwarded. 

As seen in the highlighted flow entry above in the table, the current flow, which has a destination MAC address of f2:32:d8:07:e2:a6 (the MAC of backend1 pod), matches the fourth flow entry. The actions in the fourth flow entry are as following : 

- set the reg1 register to "0x30".  0x30 in hexadecimal corresponds to [3 x (16 to the power of 1) + 0 x (16 to the power of 0)] = 48. And "48" is the OF port id of backend1 pod on the OVS. (which can be verified in [Part A Section 3.4](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_a#34-identifying-ovs-port-ids-of-port-ids) Worker 1 OVS Port output)
- set the reg0[16] register to "1" (Hex : 0x1) 
- hand over the flow to Table 90 by "resubmit(,90)"

Hence next stop is Table 90.

Just to emphasize once more, as a result of the actions mentioned in above bullets, OVS now knows that the destination of this flow is OF port 48 and the destination MAC address is known. However there is still more processing that needs to be done, as Table 90 and onwards.

**Note :** In the OVS Pipeline diagram [here](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#2-ovs-pipeline), there are tables 85,89 before Table 90. However those tables are in use only when Antrea Network Policy feature of Antrea is used. In this Antrea environment, it is not used. 

## 5.9 IngressRule Table #90

At this stage, the flow is in Table 90. 

Table 90 has flow entries which correspond to the ingress rules configured in all the Kubernetes Network Policies applied to all the pods on Worker 1 node. There is a frontend pod and a backend1 pod on the Worker 1 node. "frontendpolicy" is applied to frontend pod and "backendpolicy" is applied to backend1 pod and backend2 pod (on Worker2) . 

Kubernetes Network Policy named as "backendpolicy" is shown below.

<pre><code>
vmware@master:~$ kubectl get netpol backendpolicy
NAME            POD-SELECTOR   AGE
backendpolicy   role=backend   7d20h
vmware@master:~$ kubectl get netpol backendpolicy -o yaml
apiVersion: networking.Kubernetes.io/v1
kind: NetworkPolicy
metadata:
   name: <b>backendpolicy</b>
  namespace: default
spec:
  <b>ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - port: 80
      protocol: TCP</b>
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  - Egress
vmware@master:~$ kubectl describe netpol backendpolicy
Name:         backendpolicy
Namespace:    default
Created on:   2020-09-14 15:27:33 +0000 UTC
Labels:       <none>
Annotations:  Spec:
  PodSelector:     role=backend
  <b>Allowing ingress traffic:
    To Port: 80/TCP
    From:
      PodSelector: role=frontend</b>
  Allowing egress traffic:
    <none> (Selected pods are isolated for egress connectivity)
  Policy Types: Ingress, Egress
vmware@master:~$ 
</code></pre>

The Table 90 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=90 --no-stats
 cookie=0x1000000000000, table=90, priority=210,ct_state=-new+est,ip actions=resubmit(,105)
 cookie=0x1000000000000, table=90, priority=210,ip,nw_src=10.222.1.1 actions=resubmit(,105)
 cookie=0x1050000000000, table=90, priority=200,ip,nw_src=10.222.1.48 actions=conjunction(3,1/3)
 cookie=0x1050000000000, table=90, priority=200,tcp,tp_dst=80 actions=conjunction(4,3/3),conjunction(3,3/3)
 cookie=0x1050000000000, table=90, priority=200,ip,reg1=0x30 actions=conjunction(3,2/3)
 cookie=0x1050000000000, table=90, priority=200,ip,reg1=0x31 actions=conjunction(4,2/3)
 cookie=0x1050000000000, table=90, priority=200,ip actions=conjunction(4,1/3)
 cookie=0x1050000000000, table=90, priority=190,conj_id=3,ip actions=load:0x3->NXM_NX_REG6[],resubmit(,105)
 cookie=0x1050000000000, table=90, priority=190,conj_id=4,ip actions=load:0x4->NXM_NX_REG6[],resubmit(,105)
 cookie=0x1000000000000, table=90, priority=0 actions=resubmit(,100)
vmware@master:~$ 
</code></pre>

The first flow entry checks whether if the flow is an already established flow (-new,+est); if it is then there is no need to process the flow against the remaining flow entries, since Kubernetes Network Policy is STATEFUL by nature. However the current flow is a NEW flow hence it does NOT match this first flow entry. 

The second flow entry matches on the source IP of 10.222.1.1 (antrea-gw0 IP). The same flow entry has an action of handing the flow over to Table 105. This flow entry is used by the kubelet process on the Worker 1 node to probe local pods. More info can be found [here](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/). 

The third flow entry matches on source IP of 10.222.1.48 (frontend pod IP). The same flow entry has a conjunction action with a conjunction id of "3".

The fourth flow entry matches on destination protocol and port which is TCP 80. The same flow entry has a conjunction action with a conjunction id of "3" and "4". 

**NOTE:** Conjunction 4 is not the focus of this section since it addresses the ingress traffic to the "frontend pod" which is configured in the ingress section of the "frontendpolicy" but it is not relevant here since the current flow is ingress to backend1 pod. 

The fifth flow entry matches on the OF port id where the flow will be delivered to (reg1=0x30). 0x30 corresponds to 48 in decimal, which is the OF port that the backend pod is connected to. This is an additional check (verifying the OF port ID of the receiver) that the ingress table implements in addition to source IP check. The same flow entry has a conjunction action with a conjunction id of "3".

<b>So conjunction 3 is fully implemented by third, fourth and fifth flow entries.</b> Conjunction 3 checks if the source IP is 10.222.1.48 and then if the destination is TCP 80 and then if the destination OF port id is "48". This actually corresponds to TCP 80 specific rule in the ingress section of the Kubernetes Network Policy named as "backendpolicy".

The eighth flow entry defines two actions for conjunction 3 as soon as all fields of conjunction 3 (which is explained above with third, fourth and fifth flow entries) is matched. First action is to set the register NXM_NX_REG6 with the conjunction id of "3" (0x3). The second action in the same flow entry is to hand the flow over to the next table which is Table 105. (resubmit(,105))

**Note :** NXM_NX_REG6 register is used to cache the conjunction id which is mapped to the ingress Network Policy Rule in Antrea agent and also then written back to the Antrea custom resource definition in Kubernetes API for another feature of Antrea called "traceflow". More info on it can be found [here](https://github.com/vmware-tanzu/antrea/blob/master/docs/traceflow-guide.md).

**The current flow (which is from frontend pod IP to backend1 pod IP on protocol TCP 80) matches conjunction 3 (defined in eigth flow entry) and it will be handed over to Table 105. So next stop is Table 105. (explained in Section 5.10 below)**

For reference, remaining flow entries are explained below : 

Last flow entry in Table 90 defines that if the flow does not match any of the above entries then the flow will be handed over to Table 100 which is IngressDefaultTable (resubmit(,100)). Table 100 is for isolation rules. Basically when a network policy is applied to a pod, the flows which do not match any of the flows in Table 90 will be dropped by Table 100.

For reference, IngressDefault table on Worker 1 node is shown below. As the current flow matched conjunction 3 in Table 90, it bypasses Table 100.

The Table 100 on Worker 1 is shown below. 

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=100 --no-stats
 cookie=0x1000000000000, table=100, priority=200,ip,reg1=0x30 actions=drop
 cookie=0x1000000000000, table=100, priority=200,ip,reg1=0x31 actions=drop
 cookie=0x1000000000000, table=100, priority=0 actions=resubmit(,105)
vmware@master:~$ 
</code></pre>

Reason there are two different OF port IDs in the first two flow entries here is, there is an ingress rule used in two different Kubernetes Network Policies; one is "frontendpolicy" applied to frontend pod, the other is "backendpolicy" applied to backend pods. Each of the first two flow entries in this table applies to the respective pod' s OVS interface, running on Worker 1 node. 

The last flow entry in Table 100 basically hands all the flows, which do not match any of the conjunctions in Table 90 or the first flow entries in Table 100, over to the next table - Table 105. 

## 5.10 ConntrackCommit Table #105

Table 105 on Worker 1 is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=105 --no-stats
 <b>cookie=0x1000000000000, table=105, priority=200,ct_state=+new+trk,ip,reg0=0x1/0xffff actions=ct(commit,table=110,zone=65520,exec(load:0x20->NXM_NX_CT_MARK[]))</b>
 cookie=0x1000000000000, table=105, priority=190,ct_state=+new+trk,ip actions=ct(commit,table=110,zone=65520)
 cookie=0x1000000000000, table=105, priority=0 actions=resubmit(,110)
vmware@master:~$ 
</code></pre>

This table is for committing all the new flows for tracking them. This action is specifically explained as "Commit the connection to the connection tracking module which will be stored beyond the lifetime of packet in the pipeline." [in OVS docs](https://docs.openvswitch.org/en/latest/tutorials/ovs-conntrack/).

The first flow entry checks whether if the flow is a new flow (+new) and if it is a tracked flow (+trk). The same flow entry also checks if the flow is coming from the gateway interface. This is represented by reg0=0x1/0xffff. Explanation of this is as following. The second part "0xffff" instructs OVS to verify the first 16 bits in reg0, meaning reg0[0..15] and then the first part "0x1" means the result of that check should be "1". So it basically checks reg0[0..15]. The Reg0[0..15] is set in Classifier Table 10 and it is set to "1" if the flow comes from antrea-gw0 interface. 

The second flow entry checks whether if the flow is a new flow (+new) and if it is a tracked flow (+trk). 

The current flow is a NEW and TRACKED flow and additionally it is coming from the gateway interface hence its reg0[0..15] was already set to "1" earlier in Table 0 (Section 5.1). <b>So the current flow matches the first flow entry in Table 105</b>. The actions in the first flow entry are as following :

- commit this tracked flow to conntrack table and hand it over to Table 110 (actions=ct(commit,table=110..)) 
- set the NXM_NX_CT_MARK[] register to 0x20 (load:0x20)

The next stop is Table 110.

**Note :** <b>NSM_NX_CT_MARK register is used at a later stage for identifying the response from the backend1 pod and take the appropriate action on that return traffic.</b> The reason is backend1 pod' s response to the current flow will have to be steered back to antrea-gw0 interface for ip tables NAT rule processing. (will be explained in Section 6.4) Why ? Because the current flow has come from the backendsvc service (from iptables processing) and will be delivered to the backend1 pod; however the actual source IP of the current flow is frontend pod IP and destination IP is backend1 pod IP.

## 5.11 L2ForwardingOut Table #110

Table 110 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=110 --no-stats
 cookie=0x1000000000000, table=110, priority=200,ip,reg0=0x10000/0x10000 actions=output:NXM_NX_REG1[]
 cookie=0x1000000000000, table=110, priority=0 actions=drop
vmware@master:~$
</code></pre>

This table' s job is simple. First flow entry in this table first reads the value of register reg0[16] in the flow. If the value of this register is "1" in decimal, that means the destination MAC address is known to OVS and the flow should be able to get forwarded (otherwise it would get dropped). The same flow entry has an action defined as "actions=output:NXM_NX_REG1[]". What this action does is it reads the value in "NXM_NX_REG1" to determine the OF port this flow will be sent through and then sends the flow onwards to that port.

<b>The value reg0[16] in the current flow was set to "1" back in L2ForwardingCalc Table #80. The value of REG1 in the current flow was set to "0x30" (which is "48" in decimal) also back in L2ForwardingCalc Table #80. "48" is the OF Port ID of backend1 pod interface. Hence the OVS sends the current flow onwards to the backend1 pod on OF port 48.</b>

**Note :** The second flow entry in this table obviously drops the flows which do not have their "reg0[16]" register set.

The logic of "reg0=-0x10000/0x10000" in the flow entry is that the first 0x10000 is the desired value of this register. The second 0x10000 is the exact instructions on which specific bit(s) should be verified to match the desired value. Since "0x" is hexadecimal, below is a detailed explanation of the position of the bit that is verified. 

<pre><code>
23              16  15               8   7               0
0 0 0 0 | 0 0 0 1   0 0 0 0 |  0 0 0 0   0 0 0 0 | 0 0 0 0
</code></pre>

So bit 16 must be "1", and that is being verified in "reg0". The first four bits on the left hand side is not worth to mention hence the desired value and actual value are both shown as "0x10000". 

# 6. Backend Pod to Service
[Back to table of contents](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#part-b)

In this section the response from backend1 pod to the frontend pod will be explained. However the title above says "Backend Pod to Service" ? Why ? 

The flow which made its way to the backend1 pod (in Section 5) had the source IP of frontend pod (10.222.1.48) and destination IP of backend1 pod (10.222.1.47). Hence backend1 pod will reply to frontend pod with its own IP (10.222.1.47) which is expected by any TCP/IP based communication. OVS could easily deliver the response to frontend pod directly.  **But this would break the communication.** The reason is when the frontend pod had initiated the connection (in Section 4) the source IP of the flow was frontend pod IP but the destination IP was backendsvc service IP (10.104.65.133). This destination service IP got DNATed by iptables to the destination IP of backend1 pod (10.222.1.47) in Section 4.8.  From frontend pod' s point of view it is communicating with backendsvc service IP. Because of this; return flow from backend1 pod to frontend pod should be **SNATed** now by iptables and be delivered to the frontend pod with the IP of the backendsvc service IP as the source IP. Hence OVS needs to steer the return flow from backend1 pod, which is destined to frontend pod, to the Worker 1 node' s Kernel IP Stack for iptables processing.

To verify how backend1 pod responds to requests from the frontend pod, a quick tcpdump on the backend1 pod would reveal the source and destination IP/MAC of this flow. It is shown below.

<pre><code>
vmware@master:~$ k exec -it frontend -- sh
/ # curl backendsvc
Praqma Network MultiTool (with NGINX) - backend1 - 10.222.1.47/24
</code></pre>

while performing curl on frontend pod (as shown above), in another ssh session to the Kubernetes master node :

<pre><code>
vmware@master:~$ k exec -it backend1 -- sh
/ # tcpdump -en
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
15:36:41.452668 <b>4e:99:08:c1:53:be</b> (oui Unknown) > <b>f2:32:d8:07:e2:a6</b> (oui Unknown), ethertype IPv4 (0x0800), length 74: <b>10.222.1.48.38994</b> > <b>10.222.1.47.80</b>: Flags [S], seq 811052796, win 64860, options [mss 1410,sackOK,TS val 2845949950 ecr 0,nop,wscale 7], length 0
15:36:41.452703 <b>f2:32:d8:07:e2:a6</b> (oui Unknown) > <b>be:2c:bf:e4:ec:c5</b> (oui Unknown), ethertype IPv4 (0x0800), length 74: <b>10.222.1.47.80</b> > <b>10.222.1.48.38994</b>: Flags [S.], seq 944115724, ack 811052797, win 64308, options [mss 1410,sackOK,TS val 684640496 ecr 2845949950,nop,wscale 7], length 0
<b>OUTPUT OMITTED</b>
</code></pre>

**Note 1:** For simplicity, the ARP requests/replies between antrea-gw0, frontend pod and backend1 pod are not shown in the above output. 

- The source and destination MAC addresses of the first line in the tcpdump output are antrea-gw0 MAC and backend1 pod MAC (the flow explained in Section 5)
- The source and destination MAC addresses of the second line in the tcpdump output are backend1 pod MAC and frontend pod MAC (the flow that will be explained in this section)
- The source or destination IP addresses are always frontend pod IP and backend1 pod IP

**Note 2:** Reason that backend1 pod populates the destination MAC address with frontend pod' s MAC (rather than antrea-gw0 MAC which the request came in with as source MAC) is that frontend pod and backend1 pod are on the same subnet, hence frontend pod replies to backend1 pod' s ARP request directly. So backend1 pod has the frontend pod IP/MAC in its ARP table.

The response that backend1 pod generates has the following values in the Ethernet and IP headers and this is the flow that this section focuses on.

- Source IP = 10.222.1.47 (backend1 pod IP)
- Destination IP = 10.222.1.48 (frontend pod IP)
- Source MAC = f2:32:d8:07:e2:a6 (backend1 pod MAC)
- Destination MAC = be:2c:bf:e4:ec:c5 (frontend pod MAC)

This flow will come to OVS on backend1 pod port. OVS will steer this flow as shown below (to make it processed by Kube-proxy managed iptables NAT rules again)

![](2020-09-22-21-44-43.png)

This flow will be matched against a flow entry in each OVS Table, processed top to bottom in each individual table, based on the priority value of the flow entry in the table.

## 6.1 Classifier Table #0

Table #0 on Worker 1 node is shown below. 

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=0 --no-stats
 cookie=0x1000000000000, priority=200,in_port="antrea-gw0" actions=load:0x1->NXM_NX_REG0[0..15],resubmit(,10)
 cookie=0x1000000000000, priority=200,in_port="antrea-tun0" actions=move:NXM_NX_TUN_METADATA0[28..31]->NXM_NX_REG9[28..31],load:0->NXM_NX_REG0[0..15],load:0x1->NXM_NX_REG0[19],resubmit(,30)
 cookie=0x1030000000000, priority=190,in_port="coredns--3e3abf" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)
 cookie=0x1030000000000, priority=190,in_port="antrea-o-830766" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)
 <b>cookie=0x1030000000000, priority=190,in_port="backend1-bab86f" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)</b>
 cookie=0x1030000000000, priority=190,in_port="frontend-a3ba2f" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)
 cookie=0x1000000000000, priority=0 actions=drop
vmware@master:~$
</code></pre>

This table is to classify the current flow by matching it on the ingress port and then setting the register NXM_NX_REG0[0..15] bits as following; "0" for tunnel, "1" for local gateway and "2" for local pod. 

The current flow came from backend1 pod interface which is the local pod hence it matches the <b>fifth</b> flow entry in the above output (which is highlighted). The first action in this flow entry is to set the register reg0[0..15] to "0x2" since the flow comes from a local pod. The second action in this flow entry is to hand the flow over to Table 10 (resubmit(,10)). Hence next stop is Table 10. 

## 6.2 Spoofguard Table #10

Table #10 on Worker 1 node is shown below. "nw_src" refers to the IP address and "dl_src" refers to the MAC address of the pod connected to the respective OF port.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=10 --no-stats
 cookie=0x1000000000000, table=10, priority=200,ip,in_port="antrea-gw0" actions=resubmit(,30)
 cookie=0x1000000000000, table=10, priority=200,arp,in_port="antrea-gw0",arp_spa=10.222.1.1,arp_sha=4e:99:08:c1:53:be actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,arp,in_port="coredns--3e3abf",arp_spa=10.222.1.2,arp_sha=f2:82:cc:96:da:bd actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,arp,in_port="antrea-o-830766",arp_spa=10.222.1.3,arp_sha=6e:9e:5a:3e:3f:e8 actions=resubmit(,20)
 <b>cookie=0x1030000000000, table=10, priority=200,arp,in_port="backend1-bab86f",arp_spa=10.222.1.47,arp_sha=f2:32:d8:07:e2:a6 actions=resubmit(,20)</b>
 cookie=0x1030000000000, table=10, priority=200,arp,in_port="frontend-a3ba2f",arp_spa=10.222.1.48,arp_sha=be:2c:bf:e4:ec:c5 actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,ip,in_port="coredns--3e3abf",dl_src=f2:82:cc:96:da:bd,nw_src=10.222.1.2 actions=resubmit(,30)
 cookie=0x1030000000000, table=10, priority=200,ip,in_port="antrea-o-830766",dl_src=6e:9e:5a:3e:3f:e8,nw_src=10.222.1.3 actions=resubmit(,30)
 <b>cookie=0x1030000000000, table=10, priority=200,ip,in_port="backend1-bab86f",dl_src=f2:32:d8:07:e2:a6,nw_src=10.222.1.47 actions=resubmit(,30)</b>
 cookie=0x1030000000000, table=10, priority=200,ip,in_port="frontend-a3ba2f",dl_src=be:2c:bf:e4:ec:c5,nw_src=10.222.1.48 actions=resubmit(,30)
 cookie=0x1000000000000, table=10, priority=0 actions=drop
vmware@master:~$ 
</code></pre>

What this table does is verifying if the source IP and MAC of the current flow matches the IP and MAC assigned to the Pod by Antrea CNI plugin during initial Pod connectivity. It implements this check both for IP and ARP traffic.

Since the flow comes from the backend1 pod with backend1 pod' s IP and MAC address, the flow matches the <b>ninth</b> flow entry in this table and the spoofguard check will succeed. The only action specified in the ninth flow entry is handing the flow over to Table 30 (actions=goto_table:30). So next stop is Table 30.

## 6.3 ConntrackTable Table #30

Table #30 on Worker 1 is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=30 --no-stats
 cookie=0x1000000000000, table=30, priority=200,ip <b>actions=ct(table=31</b>,zone=65520)
vmware@master:~$ 
</code></pre>

Conntrack table' s job is to start tracking all the traffic. ("ct" action means connection tracking) Any flow that goes through this table will be in "tracked" (trk) state. In generic networking security terms this functionality makes the OVS a stateful connection aware component. Flow then gets handed over to the next table which is Table 31; as seen in the "actions" section of the flow entry. (actions=ct(table=31,)) So next stop is Table 31.

**Note :** Zone ID is explained [here](https://github.com/vmware-tanzu/antrea/blob/master/docs/ovs-pipeline.md#conntracktable-30) as "A ct_zone is simply used to isolate connection tracking rules. It is similar in spirit to the more generic Linux network namespaces, but ct_zone is specific to conntrack and has less overhead." 

## 6.4 ConnTrackState Table #31

Table #31 on Worker 1 node is shown below. 

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=31 --no-stats
 cookie=0x1000000000000, table=31, priority=210,ct_state=-new+trk,ct_mark=0x20,ip,reg0=0x1/0xffff actions=resubmit(,40)
 cookie=0x1000000000000, table=31, priority=200,<b>ct_state=-new+trk,ct_mark=0x20,ip actions=load:0x4e9908c153be->NXM_OF_ETH_DST[],resubmit(,40)</b>
 cookie=0x1000000000000, table=31, priority=190,ct_state=+inv+trk,ip actions=drop
 cookie=0x1000000000000, table=31, priority=0 actions=resubmit(,40)
vmware@master:~$ 
</code></pre>

ConntrackState table processes all the flows that are in tracked state (basically which were handed over by the Conntrack table 30). 

The first flow entry implements three checks. First check is if the flow is not new and tracked ("ct_state=-new means not new, +trk means being tracked) . Second check is if the flow's "ct_mark" field is set to "0x20". Third check is if the flow comes from antrea-gw0 interface. (by checking the reg0[0..15] register of the flow to see if it is set to "0x1" which is "1" in decimal, representing local gateway, as explained back in Table 0) 

The second flow entry checks whether if the flow is not new and tracked ("ct_state=-new means not new, +trk means being tracked). It also checks if the flow's "ct_mark" field is set to "0x20". 

The third flow entry checks if the flow is INVALID but TRACKED, basically it drops all these types of flows.

The current flow from backend1 pod to frontend pod is NOT NEW, it is the response to the flow explained in Section 5. The current flow is also a TRACKED flow, so its "ct_state" is "-new+trk". The "ct_mark" field of the flow was set to "0x20" as explained back in section 5.10 (when the request from frontend pod to service to backend1 pod communication was processed by Table 105 previously in Section 5) 

Hence the current flow will match all the conditions in the **second** flow entry in the flow table highlighted above. There are two actions specified in that second flow entry. First action is to set the destination MAC to "4e:99:08:c1:53:be" which is the antrea-gw0 MAC on the Worker 1 node. (by 0x4e9908c153be->NXM_OF_ETH_DST[]) The second action in the same flow entry is handing the flow over to the next table which is table 40 (resubmit(,40). So next stop is Table 40.

**Note :** The reason for the destination MAC rewrite (from original destination MAC of frontend pod to new destination MAC which is gw0 MAC) is to steer the flow back to Linux Kernel IP stack for the flow to be processed kube-proxy managed iptables NAT rules again. (Cause this is the return traffic for the same flow which was DNATed back in 4.8, so this time iptables will perform SNAT. Since the frontend pod should receive the response from backendsvc service IP)

## 6.5 DNAT Table #40

Table #40 on Worker 1 is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=40 --no-stats
 cookie=0x1040000000000, table=40, priority=200,ip,nw_dst=10.96.0.0/12 actions=mod_dl_dst:4e:99:08:c1:53:be,load:0x2->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,105)
 <b>cookie=0x1000000000000, table=40, priority=0 actions=resubmit(,50)</b>
vmware@master:~$ 
</code></pre>

This table in essence checks whether if the flow is destined to a Kubernetes service so that it can redirect the traffic to the antrea-gw0 on the Worker 1 node. 

The table has only two flow entries. The first flow entry checks whether if the destination IP of the flow is part of the service CIDR range configured in the cluster (which is 10.96.0.0/12); if it does, then certain actions are taken on the flow to steer the flow to the antrea-gw0 interface on the node.  

The destination IP of the current flow is frontend pod IP (10.222.1.48) and it does not fall into the service CIDR range in the first flow entry in Table 40. Hence the current flow will match the second/last entry which basically hands over the flow to Table 50 (actions=resubmit(,50)) . So next stop is Table 50.

**Note :** In the OVS Pipeline diagram [here](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#2-ovs-pipeline), there are tables 45,49 before Table50. However those tables are in use only when Antrea Network Policy feature of Antrea is used. In this Antrea environment, it is not used. 

## 6.6 EgressRule Table #50

Table 50 on Worker 1 node is shown below. 

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=50 --no-stats
 <b>cookie=0x1000000000000, table=50, priority=210,ct_state=-new+est,ip actions=resubmit(,70)</b>
 cookie=0x1050000000000, table=50, priority=200,ip actions=conjunction(2,2/3)
 cookie=0x1050000000000, table=50, priority=200,udp,tp_dst=53 actions=conjunction(2,3/3)
 cookie=0x1050000000000, table=50, priority=200,tcp,tp_dst=80 actions=conjunction(1,3/3)
 cookie=0x1050000000000, table=50, priority=200,ip,nw_src=10.222.1.48 actions=conjunction(1,1/3),conjunction(2,1/3)
 cookie=0x1050000000000, table=50, priority=200,ip,nw_src=10.222.1.47 actions=conjunction(5,1/2)
 cookie=0x1050000000000, table=50, priority=200,ip,nw_dst=10.222.2.34 actions=conjunction(1,2/3)
 cookie=0x1050000000000, table=50, priority=200,ip,nw_dst=10.222.1.47 actions=conjunction(1,2/3)
 cookie=0x1050000000000, table=50, priority=190,conj_id=2,ip actions=load:0x2->NXM_NX_REG5[],resubmit(,70)
 cookie=0x1050000000000, table=50, priority=190,conj_id=1,ip actions=load:0x1->NXM_NX_REG5[],resubmit(,70)
 cookie=0x1050000000000, table=50, priority=190,conj_id=5,ip actions=load:0x5->NXM_NX_REG5[],resubmit(,70)
 cookie=0x1000000000000, table=50, priority=0 actions=resubmit(,60)
vmware@master:~$ 
</code></pre>

The first flow entry in Table 50 above checks whether if the flow is an already established flow (-new,+trk); if it is then there is no need to process the flow against network policy, since Kubernetes Network Policy is STATEFUL by nature. 

The current flow is actually the response of backend1 pod to frontend pod as part of the previous request from frontend pod (the flow which is explained in previous section 5); because of this reason the current flow is not NEW and it is part of an already ESTABLISHED flow.

Hence the current flow will match the first flow entry in this table ("ct_state=-new+est"). The action specified in this first flow entry is handing the flow over to Table 70 (actions=resubmit(,70)). So next stop is Table 70. 

## 6.7 L3Forwarding Table #70

The Table 70 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=70 --no-stats
 cookie=0x1000000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.1 actions=mod_dl_dst:4e:99:08:c1:53:be,resubmit(,80)
 cookie=0x1030000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.2 actions=mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:f2:82:cc:96:da:bd,dec_ttl,resubmit(,80)
 cookie=0x1030000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.3 actions=mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:6e:9e:5a:3e:3f:e8,dec_ttl,resubmit(,80)
 cookie=0x1030000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.47 actions=mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:f2:32:d8:07:e2:a6,dec_ttl,resubmit(,80)
 cookie=0x1030000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.48 actions=mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:be:2c:bf:e4:ec:c5,dec_ttl,resubmit(,80)
 cookie=0x1020000000000, table=70, priority=200,ip,nw_dst=10.222.0.0/24 actions=dec_ttl,mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:aa:bb:cc:dd:ee:ff,load:0x1->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],load:0xa4f01c8->NXM_NX_TUN_IPV4_DST[],resubmit(,105)
 cookie=0x1020000000000, table=70, priority=200,ip,nw_dst=10.222.2.0/24 actions=dec_ttl,mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:aa:bb:cc:dd:ee:ff,load:0x1->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],load:0xa4f01ca->NXM_NX_TUN_IPV4_DST[],resubmit(,105)
 <b>cookie=0x1000000000000, table=70, priority=0 actions=resubmit(,80)</b>
vmware@master:~$ 
</code></pre>

Basically each flow entry in this flow table checks either the destination IP address or destination MAC address (in some entries both) to make a forwarding decision. 

The current flow' s source and destination MAC and IP address values are still as they are shown back at the beginning of this section. (with a slight change) Shown below again. 

- Source IP = 10.222.1.47 (backend1 pod IP)
- Destination IP = 10.222.1.48 (frontend pod IP)
- Source MAC = f2:32:d8:07:e2:a6 (backend1 Pod MAC)
- Destination MAC =  4e:99:08:c1:53:be (antrea-gw0 MAC on Worker 1) (destination MAC was be:2c:bf:e4:ec:c5 at the beginning of this section but it got rewritten to the antrea-gw0 MAC on Worker 1 node, back in Section 6.4)

Based on the current flow' s source and destination IP/MAC values, the current flow matches the last flow entry in Table 70 (since the prior flow entries match against a different destination MAC). The action in the last flow entry is "resubmit(,80)" which basically hands over the flow to Table 80. Hence next stop is Table 80.

**Note :** The first five flow entries in this table are related to ARP processing (with "dl_dst=aa:bb:cc:dd:ee:ff") and will be explained in [Part D Section 12](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_d). The sixth and seventh flow entries in this table are for inter node flow patterns and it will be explained in Part C Section 9.

## 6.8 L2ForwardingCalc Table #80

Table 80 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=80 --no-stats
 <b>cookie=0x1000000000000, table=80, priority=200,dl_dst=4e:99:08:c1:53:be actions=load:0x2->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)</b>
 cookie=0x1030000000000, table=80, priority=200,dl_dst=f2:82:cc:96:da:bd actions=load:0x3->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)
 cookie=0x1030000000000, table=80, priority=200,dl_dst=6e:9e:5a:3e:3f:e8 actions=load:0x4->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)
 cookie=0x1030000000000, table=80, priority=200,dl_dst=f2:32:d8:07:e2:a6 actions=load:0x30->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)
 cookie=0x1030000000000, table=80, priority=200,dl_dst=be:2c:bf:e4:ec:c5 actions=load:0x31->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)
 cookie=0x1000000000000, table=80, priority=0 actions=resubmit(,90)
vmware@master:~$ 
</code></pre>

What this table does is not that different than a typical IEEE 802.1d transparent bridge. This is basically the MAC address table of the OVS. Based on the destination MAC address of the flow OVS decides which OF Port the flow should be sent to. 

Each flow entry in this table sets two registers, both of which mentioned in earlier sections, will be explained here once again. 

- Reg1 is used to store OF port ID of the flow (the OVS port which the flow should be sent to). This register is set with the respective OF port ID based on the destination MAC address of the flow. This register which stores the OF port ID will be used later on in Table 110 (L2ForwardingOut Table).
- Reg0[16] is used and it is set to "1" to indicate that the given flow has a matching destination address in this table, which is known to OVS, and it should be forwarded. 

The current flow matches the first flow entry in (since the flow' s destination MAC address is 4e:99:08:c1:53:be). The actions in the first flow entry are as following : 

- set the reg1 register to Hex : 0x2. 0x2 in hexadecimal corresponds to [2 x (16 to the power of 0)] = 2. And "2" is the OF port id of antrea-gw0 interface on the OVS. (which can be verified in [Part A Section 3.4](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_a#34-identifying-ovs-port-ids-of-port-ids) Worker 1 output)
- set the reg0[16] register to "1" (Hex : 0x1)  
- hand the flow over to Table 90 by "resubmit(,90)"

Hence next stop is Table 90.

Just to emphasize once more, as a result of the actions mentioned in above bullets, OVS now knows that the destination of fhis flow is egress OF port 2 and the destination MAC address is known. However there is still more processing that needs to be done, as Table 90 and onwards.

**Note :** In the OVS Pipeline diagram [here](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#2-ovs-pipeline), there are tables 85,89 before Table 90. However those tables are in use only when Antrea Network Policy feature of Antrea is used. In this Antrea environment, it is not used. 

## 6.9 IngressRule Table #90

The Table 90 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=90 --no-stats
 <b>cookie=0x1000000000000, table=90, priority=210,ct_state=-new+est,ip actions=resubmit(,105)</b>
 cookie=0x1000000000000, table=90, priority=210,ip,nw_src=10.222.1.1 actions=resubmit(,105)
 cookie=0x1050000000000, table=90, priority=200,ip,nw_src=10.222.1.48 actions=conjunction(3,1/3)
 cookie=0x1050000000000, table=90, priority=200,tcp,tp_dst=80 actions=conjunction(4,3/3),conjunction(3,3/3)
 cookie=0x1050000000000, table=90, priority=200,ip,reg1=0x30 actions=conjunction(3,2/3)
 cookie=0x1050000000000, table=90, priority=200,ip,reg1=0x31 actions=conjunction(4,2/3)
 cookie=0x1050000000000, table=90, priority=200,ip actions=conjunction(4,1/3)
 cookie=0x1050000000000, table=90, priority=190,conj_id=3,ip actions=load:0x3->NXM_NX_REG6[],resubmit(,105)
 cookie=0x1050000000000, table=90, priority=190,conj_id=4,ip actions=load:0x4->NXM_NX_REG6[],resubmit(,105)
 cookie=0x1000000000000, table=90, priority=0 actions=resubmit(,100)
vmware@master:~$ 
</code></pre>

This table consists of the corresponding flow entries of the ingress rules that are configured in Kubernetes Network Policies "frontendpolicy" and "backendpolicy" respectively.

The current flow is actually the response of backend1 pod to frontend pod as part of the previous request from frontend pod (the flow which is explained in previous section 5); because of this reason the current flow is not NEW and it is part of an already ESTABLISHED flow.

Hence the current flow will match the first flow entry in this table ("ct_state=-new+est"). The action specified in this first flow entry is handing the flow over to Table 105 (actions=resubmit(,105)). So next stop is Table 105. 

## 6.10 ConntrackCommit Table #105

Table 105 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=105 --no-stats
 cookie=0x1000000000000, table=105, priority=200,ct_state=+new+trk,ip,reg0=0x1/0xffff actions=ct(commit,table=110,zone=65520,exec(load:0x20->NXM_NX_CT_MARK[]))
 cookie=0x1000000000000, table=105, priority=190,ct_state=+new+trk,ip actions=ct(commit,table=110,zone=65520)
 <b>cookie=0x1000000000000, table=105, priority=0 actions=resubmit(,110)</b>
vmware@master:~$ 
</code></pre>

This table is for committing all the new flows for tracking them. This action is specifically explained as "Commit the connection to the connection tracking module which will be stored beyond the lifetime of packet in the pipeline." [in OVS docs](https://docs.openvswitch.org/en/latest/tutorials/ovs-conntrack/).

The first flow entry checks whether if the flow is a new flow (+new) and if it is a tracked flow (+trk). The same flow entry also checks if the flow is coming from the gateway interface. This is represented by reg0=0x1/0xffff. Explanation of this is as following. The second part "0xffff" instructs OVS to verify the first 16 bits in reg0, meaning reg0[0..15] and then the first part "0x1" means the result of that check should be "1". So it basically checks reg0[0..15]. The Reg0[0..15] is set in Classifier Table 10 and it is set to "1" if the flow comes from antrea-gw0 interface. 

The second flow entry checks whether if the flow is a new flow (+new) and if it is a tracked flow (+trk). 

The current flow does **NOT** match the first nor the second flow entry in this table. Because the current flow is the response of backend1 pod, hence it is part of an already established flow and it matches the **last entry** in this table. The action in the last flow entry is specified as "resubmit(,110)" which basically is handing the flow over to the Table 110. So next stop is Table 110.

## 6.11 L2ForwardingOut Table #110

Table 110 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=110 --no-stats
 cookie=0x1000000000000, table=110, priority=200,ip,reg0=0x10000/0x10000 actions=output:NXM_NX_REG1[]
 cookie=0x1000000000000, table=110, priority=0 actions=drop
vmware@master:~$
</code></pre>

This table' s job is simple. First flow entry in this table first reads the value in register reg0[16]. If the value of this register is "1" in decimal, that means the destination MAC address is known to OVS and the flow should be able to get forwarded (otherwise it would get dropped). The same flow entry has an action defined as "actions=output:NXM_NX_REG1[]". What this action does is it reads the value in "NXM_NX_REG1" to determine the OF port this flow will be sent through and then sends the flow onwards to that port.

The reg0[16] was set to "1" back in L2ForwardingCalc Table #80. The value of REG1  was set to "0x2" (which is "2" in decimal) also back in L2ForwardingCalc Table #80. "2" is the OF Port ID of antrea-gw0 interface.  **Hence the OVS sends this flow onwards to the antrea-gw0 interface on the Worker 1 node.**

**Note :** The second flow entry in this table obviously drops all the other flows which do not have their "reg0[16]" register set.

The logic of "reg0=-0x10000/0x10000" in the flow entry is that the first 0x10000 is the desired value of this register. The second 0x10000 is the exact instructions on which specific bit(s) should be verified to match the desired value. Since "0x" is hexadecimal, below is a detailed explanation of the position of the bit that is verified. 

<pre><code>
23              16  15               8   7               0
0 0 0 0 | 0 0 0 1   0 0 0 0 |  0 0 0 0   0 0 0 0 | 0 0 0 0
</code></pre>

So bit 16 must be "1", and that is being verified in "reg0". The first four bits on the left hand side is not worth to mention hence the desired value and actual value are both shown as "0x10000". 

## 6.12 IPTables

The flow is now in Kernel IP stack of Worker 1 node to be processed by kube-proxy managed iptables NAT rules. Iptables NAT rules on Worker 1 node are shown below.

The current flow is the response of backend1 pod to the request of frontend pod' s to the backendsvc service. However the current flow has a source IP of 10.222.1.47 (backend1 pod IP) and destination IP of 10.222.1.48 (frontend pod IP). Since the current flow is part of an already etablished flow (explained in Section 4) which was processed by iptables NAT rules (back in Section 4.8), this time the source IP of this current flow is SNATed from the backend1 pod IP to the backendsvc IP (10.104.65.133). 

Highlighted entries are related to the the backendsvc service.

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

In the next section this flow, which has been SNATed by iptables, from backendsvc service to the frontend pod is explained.

# 7. Service to Frontend
[Back to table of contents](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#part-b)

In this section the response from backendsvc service to the frontend pod will be explained. 

In the previous section (6.12) kube-proxy managed iptables NAT rules on Worker 1 node applied SNAT to the flow from backend1 pod to the frontend pod. Hence the current flow has source IP of 10.104.65.133 (backendsvc service IP) and destination IP of 10.222.1.48 (frontend pod IP)

To verify how frontend pod receives the response from the backendsvc, a quick tcpdump on the frontend pod (while generating an http request from the same frontend pod using curl in a different shell) would reveal the source and destination IP/MAC of this communication, which is shown below.

<pre><code>
vmware@master:~$ kubectl exec -it frontend -- sh
/ # 
/ # tcpdump -en
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
<b>OUTPUT OMITTED</b>
13:34:36.631128 be:2c:bf:e4:ec:c5 > 4e:99:08:c1:53:be, ethertype IPv4 (0x0800), length 74: 10.222.1.48.41132 > 10.104.65.133.80: Flags [S], seq 3708988602, win 64860, options [mss 1410,sackOK,TS val 3357033759 ecr 0,nop,wscale 7], length 0
13:34:36.632662 <b>4e:99:08:c1:53:be > be:2c:bf:e4:ec:c5</b>, ethertype IPv4 (0x0800), length 74: <b>10.104.65.133.80 > 10.222.1.48.41132</b>: Flags [S.], seq 4193653914, ack 3708988603, win 64308, options [mss 1410,sackOK,TS val 2646923917 ecr 3357033759,nop,wscale 7], length 0
<b>OUTPUT OMITTED</b>
</code></pre>

**The highlighted line in the above output is the backendsvc service response to frontend pod. This section will investigate how this flow is delivered from the Kernel IP stack of Worker 1 Node to the frontend pod.**

So the current flow has the following values in the Ethernet and IP headers.

- Source IP = 10.104.65.133 (backendsvc service IP)
- Destination IP = 10.222.1.48 (frontend pod IP)
- Source MAC = 4e:99:08:c1:53:be (antrea-gw0 interface MAC on Worker 1)
- Destination MAC = be:2c:bf:e4:ec:c5 (frontend pod MAC)

This flow will come to OVS on OF port 2, which is the antrea-gw0 port. Shown below.

![](2020-09-23-14-56-24.png)

This flow will be matched against a flow entry in each OVS Table, processed top to bottom in each individual table, based on the priority value of the flow entry in the table.

## 7.1 Classifier Table #0

Table #0 on Worker 1 node is shown below.  

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=0 --no-stats
 <b>cookie=0x1000000000000, priority=200,in_port="antrea-gw0" actions=load:0x1->NXM_NX_REG0[0..15],resubmit(,10)</b>
 cookie=0x1000000000000, priority=200,in_port="antrea-tun0" actions=move:NXM_NX_TUN_METADATA0[28..31]->NXM_NX_REG9[28..31],load:0->NXM_NX_REG0[0..15],load:0x1->NXM_NX_REG0[19],resubmit(,30)
 cookie=0x1030000000000, priority=190,in_port="coredns--3e3abf" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)
 cookie=0x1030000000000, priority=190,in_port="antrea-o-830766" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)
 cookie=0x1030000000000, priority=190,in_port="backend1-bab86f" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)
 cookie=0x1030000000000, priority=190,in_port="frontend-a3ba2f" actions=load:0x2->NXM_NX_REG0[0..15],resubmit(,10)
 cookie=0x1000000000000, priority=0 actions=drop
vmware@master:~$
</code></pre>

This table is to classify the current flow by matching it on the ingress port and then setting the register NXM_NX_REG0[0..15] bits as following; "0" for tunnel, "1" for local gateway and "2" for local pod. 

The current flow came to OVS on antrea-gw0 port which is the local gateway. Hence it matches the **first** flow entry in the above output (which is highlighted). The first action in this flow entry is to set the register reg0[0..15] to "0x1" since the flow came from the local gateway. The second action in the same flow entry is to hand the flow over to Table 10 (resubmit(,10)). Hence next stop is Table 10.

## 7.2 Spoofguard Table #10

Table #10 on Worker 1 node is shown below. "nw_src" refers to the IP address and "dl_src" refers to the MAC address of the pod connected to the respective OF port.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=10 --no-stats
 <b>cookie=0x1000000000000, table=10, priority=200,ip,in_port="antrea-gw0" actions=resubmit(,30)</b>
 cookie=0x1000000000000, table=10, priority=200,arp,in_port="antrea-gw0",arp_spa=10.222.1.1,arp_sha=4e:99:08:c1:53:be actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,arp,in_port="coredns--3e3abf",arp_spa=10.222.1.2,arp_sha=f2:82:cc:96:da:bd actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,arp,in_port="antrea-o-830766",arp_spa=10.222.1.3,arp_sha=6e:9e:5a:3e:3f:e8 actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,arp,in_port="backend1-bab86f",arp_spa=10.222.1.47,arp_sha=f2:32:d8:07:e2:a6 actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,arp,in_port="frontend-a3ba2f",arp_spa=10.222.1.48,arp_sha=be:2c:bf:e4:ec:c5 actions=resubmit(,20)
 cookie=0x1030000000000, table=10, priority=200,ip,in_port="coredns--3e3abf",dl_src=f2:82:cc:96:da:bd,nw_src=10.222.1.2 actions=resubmit(,30)
 cookie=0x1030000000000, table=10, priority=200,ip,in_port="antrea-o-830766",dl_src=6e:9e:5a:3e:3f:e8,nw_src=10.222.1.3 actions=resubmit(,30)
 cookie=0x1030000000000, table=10, priority=200,ip,in_port="backend1-bab86f",dl_src=f2:32:d8:07:e2:a6,nw_src=10.222.1.47 actions=resubmit(,30)
 cookie=0x1030000000000, table=10, priority=200,ip,in_port="frontend-a3ba2f",dl_src=be:2c:bf:e4:ec:c5,nw_src=10.222.1.48 actions=resubmit(,30)
 cookie=0x1000000000000, table=10, priority=0 actions=drop
vmware@master:~$ 
</code></pre>

What this table does is verifying if the source IP and MAC of the current flow matches the IP and MAC assigned to the Pod by Antrea CNI plugin during initial Pod connectivity. It implements this check both for IP and ARP traffic.

Since the flow came from the antrea-gw0 interface, the flow matches the <b>first</b> flow entry in this table. The source IP of the current flow, which is coming from antrea-gw0 interface, is the backendsvc service IP, hence spoofguard does not do any checks in this instance. The only action specified in the first flow entry is handing the flow over to Table 30 (actions=goto_table:30). So next stop is Table 30.

**Note :** Spoofguard does not do any checks for IP packets on the antrea-gw0 port. However it still checks the ARP flows on that port. The second flow entry in the table is used for that purpose. It basically checks the ARP flows sent by the antrea-gw0 interface to the local pods running on Worker1 node.

## 7.3 ConntrackTable Table #30

Table #30 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=30 --no-stats
 cookie=0x1000000000000, table=30, priority=200,ip <b>actions=ct(table=31</b>,zone=65520)
vmware@master:~$ 
</code></pre>

Conntrack table' s job is to start tracking all the traffic. ("ct" action means connection tracking) Any flow that goes through this table will be in "tracked" (trk) state. In generic networking security terms this functionality makes the OVS a stateful connection aware component. Flow then gets handed over to the next table which is Table 31; as seen in the "actions" section of the flow entry. (actions=ct(table=31,)) So next stop is Table 31.

**Note :** Zone ID is explained [here](https://github.com/vmware-tanzu/antrea/blob/master/docs/ovs-pipeline.md#conntracktable-30) as "A ct_zone is simply used to isolate connection tracking rules. It is similar in spirit to the more generic Linux network namespaces, but ct_zone is specific to conntrack and has less overhead." 

## 7.4 ConnTrackState Table #31

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

The current flow is a tracked flow since it is the continuity of the previous flow which was from frontend pod to backendsvc service (explained in Section 4) Hence the current flow is not new and it also does not have the "ct_mark" set. ("ct_mark" was used for tracking service to backend pod communication back in Section 5 and 6)

Hence the flow matches the <b>last entry</b> in the flow table highlighted above. Notice the action in the same flow entry is handing the flow over to the next table which is table 40 (resubmit(,40). So next stop is Table 40.

## 7.5 DNAT Table #40

Table #40 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=40 --no-stats
 cookie=0x1040000000000, table=40, priority=200,ip,w_dst=10.96.0.0/12 actions=mod_dl_dst:4e:99:08:c1:53:be,load:0x2->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,105)
 cookie=0x1000000000000, table=40, priority=0 actions=resubmit(,50)
vmware@master:~$ 
</code></pre>

This table in essence checks whether if the flow is destined to a Kubernetes service so that it can redirect the traffic to the antrea-gw0 on the Worker 1 node. 

The table has only two flow entries. The first flow entry checks whether if the destination IP of the flow is part of the service CIDR range configured in the cluster (which is 10.96.0.0/12); if it does, then certain actions are taken on the flow to steer the flow to the antrea-gw0 interface on the node.  

The destination IP of the current flow is frontend pod IP (10.222.1.48) and it does not fall into the service CIDR range in the first flow entry in Table 40. Hence the current flow matches the second/last flow entry. That flow entry basically hands the flow over to Table 50 (actions=resubmit(,50)) . So next stop is Table 50.

**Note :** In the OVS Pipeline diagram [here](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#2-ovs-pipeline), there are tables 45,49 before Table50. However those tables are in use only when Antrea Network Policy feature of Antrea is used. In this Antrea environment, it is not used. 

## 7.6 EgressRule Table #50

Table 50 on Worker 1 node is shown below. 

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=50 --no-stats
 <b>cookie=0x1000000000000, table=50, priority=210,ct_state=-new+est,ip actions=resubmit(,70)</b>
 cookie=0x1050000000000, table=50, priority=200,ip actions=conjunction(2,2/3)
 cookie=0x1050000000000, table=50, priority=200,udp,tp_dst=53 actions=conjunction(2,3/3)
 cookie=0x1050000000000, table=50, priority=200,tcp,tp_dst=80 actions=conjunction(1,3/3)
 cookie=0x1050000000000, table=50, priority=200,ip,nw_src=10.222.1.48 actions=conjunction(1,1/3),conjunction(2,1/3)
 cookie=0x1050000000000, table=50, priority=200,ip,nw_src=10.222.1.47 actions=conjunction(5,1/2)
 cookie=0x1050000000000, table=50, priority=200,ip,nw_dst=10.222.2.34 actions=conjunction(1,2/3)
 cookie=0x1050000000000, table=50, priority=200,ip,nw_dst=10.222.1.47 actions=conjunction(1,2/3)
 cookie=0x1050000000000, table=50, priority=190,conj_id=2,ip actions=load:0x2->NXM_NX_REG5[],resubmit(,70)
 cookie=0x1050000000000, table=50, priority=190,conj_id=1,ip actions=load:0x1->NXM_NX_REG5[],resubmit(,70)
 cookie=0x1050000000000, table=50, priority=190,conj_id=5,ip actions=load:0x5->NXM_NX_REG5[],resubmit(,70)
 cookie=0x1000000000000, table=50, priority=0 actions=resubmit(,60)
vmware@master:~$ 
</code></pre>

The current flow is actually the response of backendsvc service to frontend pod as part of the previous flow from frontend pod (the flow which is explained back in section 4); because of this reason the current flow is not NEW and it is part of an already established flow.

Hence the current flow will match the first flow entry in this table ("ct_state=-new+est"). The action specified in this first flow entry is handing the flow over to Table 70. (actions=resubmit(,70)) So next stop is Table 70. 

## 7.7 L3Forwarding Table #70

The Table 70 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=70 --no-stats
 cookie=0x1000000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.1 actions=mod_dl_dst:4e:99:08:c1:53:be,resubmit(,80)
 cookie=0x1030000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.2 actions=mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:f2:82:cc:96:da:bd,dec_ttl,resubmit(,80)
 cookie=0x1030000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.3 actions=mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:6e:9e:5a:3e:3f:e8,dec_ttl,resubmit(,80)
 cookie=0x1030000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.47 actions=mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:f2:32:d8:07:e2:a6,dec_ttl,resubmit(,80)
 cookie=0x1030000000000, table=70, priority=200,ip,dl_dst=aa:bb:cc:dd:ee:ff,nw_dst=10.222.1.48 actions=mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:be:2c:bf:e4:ec:c5,dec_ttl,resubmit(,80)
 cookie=0x1020000000000, table=70, priority=200,ip,nw_dst=10.222.0.0/24 actions=dec_ttl,mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:aa:bb:cc:dd:ee:ff,load:0x1->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],load:0xa4f01c8->NXM_NX_TUN_IPV4_DST[],resubmit(,105)
 cookie=0x1020000000000, table=70, priority=200,ip,nw_dst=10.222.2.0/24 actions=dec_ttl,mod_dl_src:4e:99:08:c1:53:be,mod_dl_dst:aa:bb:cc:dd:ee:ff,load:0x1->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],load:0xa4f01ca->NXM_NX_TUN_IPV4_DST[],resubmit(,105)
 cookie=0x1000000000000, table=70, priority=0 actions=resubmit(,80)
vmware@master:~$ 
</code></pre>

Basically each flow entry in this flow table checks either the destination IP address or destination MAC address (in some entries both) to make a forwarding decision. 

The current flow' s source and destination MAC and IP address values are as following. Shown below again. 

- Source IP = 10.104.65.133 (backendsvc service IP)
- Destination IP = 10.222.1.48 (frontend pod IP)
- Source MAC = 4e:99:08:c1:53:be (antrea-gw0 interface MAC on Worker 1)
- Destination MAC = be:2c:bf:e4:ec:c5 (frontend pod MAC)

Based on the current flow' s source and destination MAC/IP values, the current flow matches the last flow entry in Table 70 (since the prior flow entries match against a different destination MAC). The action in the last flow entry is "resubmit(,80)" which basically hands the flow over to Table 80. Hence next stop is Table 80.

**Note :** The first five flow entries in this table are related to ARP processing (with "dl_dst=aa:bb:cc:dd:ee:ff") and will be explained in [Part D Section 12](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_d). The sixth and seventh flow entries in this table are for inter node flow patterns and it will be explained in Part C Section 9.

## 7.8 L2ForwardingCalc Table #80

Table 80 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=80 --no-stats
 cookie=0x1000000000000, table=80, priority=200,dl_dst=4e:99:08:c1:53:be actions=load:0x2->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)
 cookie=0x1030000000000, table=80, priority=200,dl_dst=f2:82:cc:96:da:bd actions=load:0x3->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)
 cookie=0x1030000000000, table=80, priority=200,dl_dst=6e:9e:5a:3e:3f:e8 actions=load:0x4->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)
 cookie=0x1030000000000, table=80, priority=200,dl_dst=f2:32:d8:07:e2:a6 actions=load:0x30->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)
 <b>cookie=0x1030000000000, table=80, priority=200,dl_dst=be:2c:bf:e4:ec:c5 actions=load:0x31->NXM_NX_REG1[],load:0x1->NXM_NX_REG0[16],resubmit(,90)</b>
 cookie=0x1000000000000, table=80, priority=0 actions=resubmit(,90)
vmware@master:~$ 
</code></pre>

What this table does is not that different than a typical IEEE 802.1d transparent bridge. This is basically the MAC address table of the OVS. Based on the destination MAC address of the flow OVS decides which OF Port the flow should be sent to. 

Each flow entry in this table sets two registers, both of which mentioned in earlier sections, will be explained here once again. 

- Reg1 is used to store OF port ID of the flow (the OVS port which the flow should be sent to). This register is set with the respective OF port ID based on the destination MAC address of the flow. This register which stores the OF port ID will be used later on in Table 110 (L2ForwardingOut Table).
- Reg0[16] is used and it is set to "1" to indicate that the given flow has a matching destination address in this table, which is known to OVS, and it should be forwarded. 

The current flow has a destination MAC address of be:2c:bf:e4:ec:c5 (the MAC address of frontend pod) hence it matches the **fifth** flow entry in the table. The actions specified in the fifth flow entry are as following : 

- set the reg1 register to "0x31".  0x31 in hexadecimal corresponds to [3 x (16 to the power of 1) + 1 x (16 to the power of 0)] = 49. And "49" is the OF port id of frontend pod on the OVS. (which can be verified in [Part A Section 3.4](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_a#34-identifying-ovs-port-ids-of-port-ids) Worker 1 output) 
- set the reg0[16] register to "1" (Hex : 0x1)  
- hand the flow over to Table 90 by "resubmit(,90)"

Hence next stop is Table 90.

Just to emphasize once more, as a result of the actions mentioned in above bullets, OVS now knows that the destination of fhis flow is egress OF port 49 and the destination MAC address is known. However there is still more processing that needs to be done, as Table 90 and onwards.

**Note :** In the OVS Pipeline diagram [here](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#2-ovs-pipeline), there are tables 85,89 before Table 90. However those tables are in use only when Antrea Network Policy feature of Antrea is used. In this Antrea environment, it is not used. 

## 7.9 IngressRule Table #90

The Table 90 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=90 --no-stats
 <b>cookie=0x1000000000000, table=90, priority=210,ct_state=-new+est,ip actions=resubmit(,105)</b>
 cookie=0x1000000000000, table=90, priority=210,ip,nw_src=10.222.1.1 actions=resubmit(,105)
 cookie=0x1050000000000, table=90, priority=200,ip,nw_src=10.222.1.48 actions=conjunction(3,1/3)
 cookie=0x1050000000000, table=90, priority=200,tcp,tp_dst=80 actions=conjunction(4,3/3),conjunction(3,3/3)
 cookie=0x1050000000000, table=90, priority=200,ip,reg1=0x30 actions=conjunction(3,2/3)
 cookie=0x1050000000000, table=90, priority=200,ip,reg1=0x31 actions=conjunction(4,2/3)
 cookie=0x1050000000000, table=90, priority=200,ip actions=conjunction(4,1/3)
 cookie=0x1050000000000, table=90, priority=190,conj_id=3,ip actions=load:0x3->NXM_NX_REG6[],resubmit(,105)
 cookie=0x1050000000000, table=90, priority=190,conj_id=4,ip actions=load:0x4->NXM_NX_REG6[],resubmit(,105)
 cookie=0x1000000000000, table=90, priority=0 actions=resubmit(,100)
vmware@master:~$ 
</code></pre>

This table consists of the corresponding flow entries of the ingress rules that are configured in Kubernetes Network Policies "frontendpolicy" and "backendpolicy" respectively.

The current flow is actually the response from the backendsvc service to frontend pod as part of the previous request from frontend pod (the flow which is explained in previous section 4); because of this reason the current flow is not NEW and it is part of an already established flow.

Hence the current flow will match the first flow entry in this table ("ct_state=-new+est"). The action specified in this first flow entry is handing the flow over to Table 105 (actions=resubmit(,105)). So next stop is Table 105. 

## 7.10 ConntrackCommit Table #105

Table 105 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=105 --no-stats
 cookie=0x1000000000000, table=105, priority=200,ct_state=+new+trk,ip,reg0=0x1/0xffff actions=ct(commit,table=110,zone=65520,exec(load:0x20->NXM_NX_CT_MARK[]))
 cookie=0x1000000000000, table=105, priority=190,ct_state=+new+trk,ip actions=ct(commit,table=110,zone=65520)
 <b>cookie=0x1000000000000, table=105, priority=0 actions=resubmit(,110)</b>
vmware@master:~$ 
</code></pre>

This table is for committing all the new flows for tracking them. This action is specifically explained as "Commit the connection to the connection tracking module which will be stored beyond the lifetime of packet in the pipeline." [in OVS docs](https://docs.openvswitch.org/en/latest/tutorials/ovs-conntrack/).

The first flow entry checks whether if the flow is a new flow (+new) and if it is a tracked flow (+trk). The same flow entry also checks if the flow is coming from the gateway interface. This is represented by reg0=0x1/0xffff. Explanation of this is as following. The second part "0xffff" instructs OVS to verify the first 16 bits in reg0, meaning reg0[0..15] and then the first part "0x1" means the result of that check should be "1". So it basically checks reg0[0..15]. The Reg0[0..15] is always set by Classifier Table 10 and it is set to "1" if the flow comes from antrea-gw0 interface. 

The second flow entry checks whether if the flow is a new flow (+new) and if it is a tracked flow (+trk).

The current flow does **not** match the first nor the second flow entry in this table. It is an already established flow hence the current flow will match the **last entry** in this table. The action in the last flow entry is specified as "resubmit(,110)" which basically hands the flow over to the Table 110. So next stop is Table 110.

## 7.11 L2ForwardingOut Table #110

Table 110 on Worker 1 node is shown below.

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-ofctl dump-flows br-int table=110 --no-stats
 cookie=0x1000000000000, table=110, priority=200,ip,reg0=0x10000/0x10000 actions=output:NXM_NX_REG1[]
 cookie=0x1000000000000, table=110, priority=0 actions=drop
vmware@master:~$
</code></pre>

This table' s job is simple. First flow entry in this table first reads the value in register reg0[16]. If the value of this register is "1" in decimal, that means the destination MAC address is known to OVS and the flow should be able to get forwarded (otherwise it would get dropped). The same flow entry has an action defined as "actions=output:NXM_NX_REG1[]". What this action does is it reads the value in "NXM_NX_REG1" to determine the OF port this flow will be sent through and then sends the flow onwards to that port.

The value of REG0[16] was set to "0x1" back in L2ForwardingCalc Table #80. The value of REG1 was set to "0x31" (which is "49" in decimal) also back in L2ForwardingCalc Table #80. "49" is the OF Port ID of frontend pod interface. Hence the OVS sends this flow onwards to the frontend pod. **At this stage frontend pod successfully receives the response from backendsvc.**

**Note :** The second flow entry in this table obviously drops all the other flows which do not have their "reg0[16]" register set.

The logic of "reg0=-0x10000/0x10000" in the flow entry is that the first 0x10000 is the desired value of this register. The second 0x10000 is the exact instructions on which specific bit(s) should be verified to match the desired value. Since "0x" is hexadecimal, below is a detailed explanation of the position of the bit that is verified. 

<pre><code>
23              16  15               8   7               0
0 0 0 0 | 0 0 0 1   0 0 0 0 |  0 0 0 0   0 0 0 0 | 0 0 0 0
</code></pre>

So bit 16 must be "1", and that is being verified in "reg0". The first four bits on the left hand side is not worth to mention hence the desired value and actual value are both shown as "0x10000".

[Back to table of contents](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#part-b)
