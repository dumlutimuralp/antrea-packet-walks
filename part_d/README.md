

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

## OFAPPCTL OUTPUT (Packet Tracing Tool)

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-appctl ofproto/trace --ct-next trk br-int in_port=48,tcp,nw_src=10.222.1.47,dl_src=f2:32:d8:07:e2:a6,ct_mark=0x20
Flow: ct_mark=0x20,tcp,in_port=48,vlan_tci=0x0000,dl_src=f2:32:d8:07:e2:a6,dl_dst=00:00:00:00:00:00,nw_src=10.222.1.47,nw_dst=0.0.0.0,nw_tos=0,nw_ecn=0,nw_ttl=0,tp_src=0,tp_dst=0,tcp_flags=0

bridge("br-int")
----------------
 0. in_port=48, priority 190, cookie 0x1030000000000
    load:0x2->NXM_NX_REG0[0..15]
    goto_table:10
10. ip,in_port=48,dl_src=f2:32:d8:07:e2:a6,nw_src=10.222.1.47, priority 200, cookie 0x1030000000000
    goto_table:30
30. ip, priority 200, cookie 0x1000000000000
    ct(table=31,zone=65520)
    drop
     -> A clone of the packet is forked to recirculate. The forked pipeline will be resumed at table 31.
     -> Sets the packet to an untracked state, and clears all the conntrack fields.

Final flow: tcp,reg0=0x2,in_port=48,vlan_tci=0x0000,dl_src=f2:32:d8:07:e2:a6,dl_dst=00:00:00:00:00:00,nw_src=10.222.1.47,nw_dst=0.0.0.0,nw_tos=0,nw_ecn=0,nw_ttl=0,tp_src=0,tp_dst=0,tcp_flags=0
Megaflow: recirc_id=0,eth,ip,in_port=48,dl_src=f2:32:d8:07:e2:a6,nw_src=10.222.1.47,nw_frag=no
Datapath actions: ct(zone=65520),recirc(0x22aa8)

===============================================================================
recirc(0x22aa8) - resume conntrack with ct_state=new|trk
===============================================================================

Flow: recirc_id=0x22aa8,ct_state=new|trk,ct_zone=65520,ct_mark=0x20,eth,tcp,reg0=0x2,in_port=48,vlan_tci=0x0000,dl_src=f2:32:d8:07:e2:a6,dl_dst=00:00:00:00:00:00,nw_src=10.222.1.47,nw_dst=0.0.0.0,nw_tos=0,nw_ecn=0,nw_ttl=0,tp_src=0,tp_dst=0,tcp_flags=0

bridge("br-int")
----------------
    thaw
        Resuming from table 31
31. priority 0, cookie 0x1000000000000
    goto_table:40
40. priority 0, cookie 0x1000000000000
    goto_table:50
50. priority 0, cookie 0x1000000000000
    goto_table:60
60. ip,nw_src=10.222.1.47, priority 200, cookie 0x1000000000000
    drop

Final flow: unchanged
Megaflow: recirc_id=0x22aa8,ct_state=+new-est-inv+trk,ct_mark=0x20,eth,tcp,in_port=48,nw_src=10.222.1.47,nw_dst=0.0.0.0/5,nw_frag=no,tp_dst=0x0/0xffe0
Datapath actions: drop
</code></pre>

## OFDPCTL OUTPUT

<pre><code>
vmware@master:~$ kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-dpctl show
system@ovs-system:
  lookups: hit:464363067 missed:674464 lost:3
  flows: 11
  masks: hit:3701419315 total:7 hit/pkt:7.96
  port 0: ovs-system (internal)
  port 1: genev_sys_6081 (geneve: packet_type=ptap)
  port 2: antrea-gw0 (internal)
  port 3: coredns--3e3abf
  port 4: antrea-o-830766
  port 5: backend1-bab86f
  port 6: frontend-a3ba2f
vmware@master:~$ 
vmware@master:~$ <b>kubectl exec -n kube-system -it antrea-agent-f76q2 -c antrea-ovs -- ovs-dpctl dump-flows br-int table=31</b>
recirc_id(0x3),in_port(2),ct_state(-new+est+trk),ct_mark(0x20),eth(dst=f2:82:cc:96:da:bd),eth_type(0x0800),ipv4(dst=10.222.1.0/255.255.255.0,frag=no), packets:2071324, bytes:193208628, used:1.712s, flags:FP., actions:3
recirc_id(0x15),in_port(4),ct_state(-new+est-inv+trk),ct_mark(0),eth(src=6e:9e:5a:3e:3f:e8,dst=4e:99:08:c1:53:be),eth_type(0x0800),ipv4(dst=10.222.0.0/255.255.255.0,tos=0/0x3,ttl=64,frag=no), packets:0, bytes:0, used:never, actions:set(tunnel(tun_id=0x0,dst=10.79.1.200,ttl=64,tp_dst=6081,flags(df|csum|key))),set(eth(src=4e:99:08:c1:53:be,dst=aa:bb:cc:dd:ee:ff)),set(ipv4(ttl=63)),1
recirc_id(0),in_port(4),eth(src=6e:9e:5a:3e:3f:e8,dst=4e:99:08:c1:53:be),eth_type(0x0806),arp(sip=10.222.1.3,tip=10.222.1.1,op=2/0xff,sha=6e:9e:5a:3e:3f:e8), packets:0, bytes:0, used:never, actions:2
recirc_id(0x1),in_port(3),ct_state(-new+est+trk),ct_mark(0x20),eth(dst=4e:99:08:c1:53:be),eth_type(0x0800),ipv4(dst=10.222.1.0/255.255.255.0,frag=no), packets:2269328, bytes:219471332, used:1.712s, flags:SFP., actions:2
<b>recirc_id(0x22d69),in_port(5),ct_state(-new+est+trk),ct_mark(0x20),eth(dst=be:2c:bf:e4:ec:c5),eth_type(0x0800),ipv4(dst=10.222.1.0/255.255.255.0,frag=no), packets:4, bytes:566, used:6.268s, flags:FP., actions:set(eth(dst=4e:99:08:c1:53:be)),2</b>
<b>recirc_id(0x3),in_port(2),ct_state(-new+est-inv+trk),ct_mark(0),eth(dst=be:2c:bf:e4:ec:c5),eth_type(0x0800),ipv4(dst=10.222.1.0/255.255.255.0,frag=no), packets:5, bytes:640, used:6.268s, flags:SFP., actions:6</b>
recirc_id(0),in_port(5),eth(src=f2:32:d8:07:e2:a6,dst=be:2c:bf:e4:ec:c5),eth_type(0x0806),arp(sip=10.222.1.47,tip=10.222.1.48,op=1/0xff,sha=f2:32:d8:07:e2:a6), packets:0, bytes:0, used:never, actions:6
recirc_id(0),in_port(2),eth(src=4e:99:08:c1:53:be,dst=be:2c:bf:e4:ec:c5),eth_type(0x0806),arp(sip=10.222.1.1,tip=10.222.1.48,op=1/0xff,sha=4e:99:08:c1:53:be), packets:0, bytes:0, used:never, actions:6
recirc_id(0x22d68),in_port(2),eth(),eth_type(0x0800),ipv4(frag=no), packets:0, bytes:0, used:never, actions:5
recirc_id(0),in_port(6),eth(src=be:2c:bf:e4:ec:c5,dst=4e:99:08:c1:53:be),eth_type(0x0806),arp(sip=10.222.1.48,tip=10.222.1.1,op=2/0xff,sha=be:2c:bf:e4:ec:c5), packets:0, bytes:0, used:never, actions:2
<b>recirc_id(0),in_port(5),eth(src=f2:32:d8:07:e2:a6),eth_type(0x0800),ipv4(src=10.222.1.47,frag=no), packets:4, bytes:566, used:6.268s, flags:FP., actions:ct(zone=65520),recirc(0x22d69)</b>
recirc_id(0x3),in_port(2),ct_state(-new+est-inv+trk),ct_mark(0),eth(dst=f2:82:cc:96:da:bd),eth_type(0x0800),ipv4(dst=10.222.1.0/255.255.255.0,frag=no), packets:2691820, bytes:1608305445, used:0.728s, flags:P., actions:3
recirc_id(0),in_port(6),eth(src=be:2c:bf:e4:ec:c5),eth_type(0x0800),ipv4(src=10.222.1.48,frag=no), packets:8, bytes:640, used:6.268s, flags:SFP., actions:ct(zone=65520),recirc(0x22d65)
recirc_id(0x22d66),in_port(6),eth(),eth_type(0x0800),ipv4(frag=no), packets:2, bytes:170, used:6.268s, flags:S, actions:2
recirc_id(0),in_port(2),eth(src=4e:99:08:c1:53:be,dst=6e:9e:5a:3e:3f:e8),eth_type(0x0806),arp(sip=10.222.1.1,tip=10.222.1.3,op=1/0xff,sha=4e:99:08:c1:53:be), packets:0, bytes:0, used:never, actions:4
recirc_id(0x3),in_port(2),ct_state(+new-est-inv+trk),ct_mark(0),eth(dst=f2:82:cc:96:da:bd),eth_type(0x0800),ipv4(src=10.222.1.48,dst=10.222.1.0/255.255.255.224,proto=17,frag=no),udp(dst=53), packets:0, bytes:0, used:never, actions:ct(commit,zone=65520,mark=0x20/0xffffffff),recirc(0x22d67)
recirc_id(0x4),in_port(2),eth(),eth_type(0x0800),ipv4(frag=no), packets:756362, bytes:63276445, used:1.712s, flags:S, actions:3
recirc_id(0),in_port(5),eth(src=f2:32:d8:07:e2:a6,dst=4e:99:08:c1:53:be),eth_type(0x0806),arp(sip=10.222.1.47,tip=10.222.1.1,op=2/0xff,sha=f2:32:d8:07:e2:a6), packets:0, bytes:0, used:never, actions:2
recirc_id(0x3),in_port(2),ct_state(-new+est+trk),ct_mark(0x20),eth(dst=f2:32:d8:07:e2:a6),eth_type(0x0800),ipv4(dst=10.222.1.0/255.255.255.0,frag=no), packets:4, bytes:264, used:6.268s, flags:F., actions:5
recirc_id(0x15),in_port(4),ct_state(-new+est-inv+trk),ct_mark(0),eth(dst=4e:99:08:c1:53:be),eth_type(0x0800),ipv4(dst=10.64.0.0/255.224.0.0,frag=no), packets:5180238, bytes:416521562, used:0.172s, flags:FPR., actions:2
recirc_id(0x1),in_port(3),ct_state(-new-inv+trk),ct_mark(0),eth(dst=4e:99:08:c1:53:be),eth_type(0x0800),ipv4(dst=10.96.0.0/255.240.0.0,frag=no), packets:2656594, bytes:189213104, used:0.684s, flags:P., actions:2
recirc_id(0),in_port(6),eth(src=be:2c:bf:e4:ec:c5,dst=f2:32:d8:07:e2:a6),eth_type(0x0806),arp(sip=10.222.1.48,tip=10.222.1.47,op=2/0xff,sha=be:2c:bf:e4:ec:c5), packets:0, bytes:0, used:never, actions:5
recirc_id(0x22d6a),tunnel(tun_id=0x0,src=10.79.1.200,dst=10.79.1.201,flags(-df+csum+key)),in_port(1),ct_state(-new+est-inv+trk),ct_mark(0),eth(src=ea:68:8e:2f:34:80,dst=aa:bb:cc:dd:ee:ff),eth_type(0x0800),ipv4(dst=10.222.1.3,ttl=61,frag=no), packets:0, bytes:0, used:never, actions:set(eth(src=4e:99:08:c1:53:be,dst=6e:9e:5a:3e:3f:e8)),set(ipv4(ttl=60)),4
recirc_id(0),in_port(6),eth(src=be:2c:bf:e4:ec:c5,dst=4e:99:08:c1:53:be),eth_type(0x0806),arp(sip=10.222.1.48,tip=10.222.1.1,op=1/0xff,sha=be:2c:bf:e4:ec:c5), packets:0, bytes:0, used:never, actions:2
recirc_id(0),in_port(3),eth(src=f2:82:cc:96:da:bd),eth_type(0x0800),ipv4(src=10.222.1.2,frag=no), packets:5377777, bytes:479872655, used:0.684s, flags:SFPR., actions:ct(zone=65520),recirc(0x1)
recirc_id(0),in_port(3),eth(src=f2:82:cc:96:da:bd,dst=be:2c:bf:e4:ec:c5),eth_type(0x0806),arp(sip=10.222.1.2,tip=10.222.1.48,op=1/0xff,sha=f2:82:cc:96:da:bd), packets:0, bytes:0, used:never, actions:6
recirc_id(0x22d67),in_port(2),eth(),eth_type(0x0800),ipv4(frag=no), packets:1, bytes:96, used:6.272s, actions:3
recirc_id(0),tunnel(tun_id=0x0,src=10.79.1.200,dst=10.79.1.201,flags(-df+csum+key)),in_port(1),eth(),eth_type(0x0800),ipv4(frag=no), packets:0, bytes:0, used:never, actions:ct(zone=65520),recirc(0x22d6a)
recirc_id(0),in_port(2),eth(src=4e:99:08:c1:53:be,dst=f2:32:d8:07:e2:a6),eth_type(0x0806),arp(sip=10.222.1.1,tip=10.222.1.47,op=1/0xff,sha=4e:99:08:c1:53:be), packets:0, bytes:0, used:never, actions:5
recirc_id(0x3),in_port(2),ct_state(+new-est-inv+trk),ct_mark(0x20),eth(dst=f2:82:cc:96:da:bd),eth_type(0x0800),ipv4(src=10.222.1.48,dst=10.222.1.0/255.255.255.224,proto=17,frag=no),udp(dst=53), packets:0, bytes:0, used:never, actions:ct(commit,zone=65520,mark=0x20/0xffffffff),recirc(0x22d67)
recirc_id(0x3),in_port(2),ct_state(+new-est-inv+trk),ct_mark(0),eth(dst=f2:32:d8:07:e2:a6),eth_type(0x0800),ipv4(src=10.222.1.48,dst=10.222.1.47,proto=6,frag=no),tcp(dst=80), packets:0, bytes:0, used:never, actions:ct(commit,zone=65520,mark=0x20/0xffffffff),recirc(0x22d68)
recirc_id(0),in_port(2),eth(src=4e:99:08:c1:53:be,dst=be:2c:bf:e4:ec:c5),eth_type(0x0806),arp(sip=10.222.1.1,tip=10.222.1.48,op=2/0xff,sha=4e:99:08:c1:53:be), packets:0, bytes:0, used:never, actions:6
recirc_id(0x3),in_port(2),ct_state(+new-est-inv+trk),ct_mark(0),eth(dst=f2:82:cc:96:da:bd),eth_type(0x0800),ipv4(src=10.222.1.1,dst=10.222.1.0/255.255.255.224,proto=6,frag=no),tcp(dst=4096/0xf000), packets:157150, bytes:11629100, used:1.713s, flags:S, actions:ct(commit,zone=65520,mark=0x20/0xffffffff),recirc(0x4)
recirc_id(0x1),in_port(3),ct_state(-new+est+trk),ct_mark(0x20),eth(dst=be:2c:bf:e4:ec:c5),eth_type(0x0800),ipv4(dst=10.222.1.0/255.255.255.0,frag=no), packets:1, bytes:189, used:6.273s, actions:set(eth(dst=4e:99:08:c1:53:be)),2
<b>recirc_id(0),in_port(2),eth(),eth_type(0x0800),ipv4(frag=no), packets:45536480, bytes:16033110808, used:0.173s, flags:SFPR., actions:ct(zone=65520),recirc(0x3)</b>
recirc_id(0x22d65),in_port(6),ct_state(-new-inv+trk),ct_mark(0),eth(dst=4e:99:08:c1:53:be),eth_type(0x0800),ipv4(dst=10.96.0.0/255.240.0.0,frag=no), packets:4, bytes:264, used:6.269s, flags:F., actions:2
recirc_id(0x3),in_port(2),ct_state(-new+est-inv+trk),ct_mark(0),eth(dst=6e:9e:5a:3e:3f:e8),eth_type(0x0800),ipv4(dst=10.222.1.0/255.255.255.0,frag=no), packets:4949870, bytes:3684225114, used:0.173s, flags:SFP., actions:4
recirc_id(0x22d65),in_port(6),ct_state(+new-inv+trk),ct_mark(0),eth(dst=4e:99:08:c1:53:be),eth_type(0x0800),ipv4(dst=10.96.0.0/255.240.0.0,frag=no), packets:2, bytes:170, used:6.269s, flags:S, actions:ct(commit,zone=65520),recirc(0x22d66)
recirc_id(0),in_port(6),eth(src=be:2c:bf:e4:ec:c5,dst=f2:82:cc:96:da:bd),eth_type(0x0806),arp(sip=10.222.1.48,tip=10.222.1.2,op=2/0xff,sha=be:2c:bf:e4:ec:c5), packets:0, bytes:0, used:never, actions:3
recirc_id(0),in_port(4),eth(src=6e:9e:5a:3e:3f:e8),eth_type(0x0800),ipv4(src=10.222.1.3,frag=no), packets:5641878, bytes:1294437785, used:0.173s, flags:SFPR.EC, actions:ct(zone=65520),recirc(0x15)
vmware@master:~$ 
</code></pre>

If destination is never used when matching the packet in the pipeline, it won’d be part of the datapath flow

If the datapath has all metadata fields of a packet, it cannot be hit by other packets







