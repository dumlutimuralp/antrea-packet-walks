# [Antrea](https://github.com/vmware-tanzu/antrea) Packet Walks 

_**This disclaimer informs readers that the views, thoughts, and opinions expressed in this series of posts belong solely to the author, and not necessarily to the author’s employer, organization, committee or other group or individual.**_

This article explains the step by step packet walks for a given application flow in the Open vSwitch (OVS) dataplane implemented by Antrea CNI within a Kubernetes cluster. The high level packet walks and the OVS Pipeline are already explained [here](https://github.com/vmware-tanzu/antrea/blob/master/docs/architecture.md#pod-networking) and [here](https://github.com/vmware-tanzu/antrea/blob/master/docs/ovs-pipeline.md) at Project Antrea' s page on Github.

The purpose of this article is to walk the reader through an actual flow pattern between a frontend pod to a Kubernetes service and look at things from that specific flow' s point of view with some other details and more wording.

Special thanks to [Quan Tian](https://github.com/tnqn), [Wenying Dong](https://github.com/wenyingd), [Ran Gu](https://github.com/gran-vmv) for responding to my queries about OVS registers and forwarding logic. 

## [PART A - Architecture and Test Environment](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md)

- [Antrea Implementation in a Worker Node](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#1-antrea-implementation-in-a-worker-node)
- [OVS Pipeline](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#2-ovs-pipeline)
- [Test Environment](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#3-test-environment)

## [PART B - Flow within the same node](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md)

- [Phase 1 Frontend to Service](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#4-phase-1---frontend-to-service)
- [Phase 2 Service to Backend Pod](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#5-phase-2---service-to-backend-pod)
- [Phase 3 Backend Pod to Service](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#6-phase-3---backend-pod-to-service)
- [Phase 4 Service to Frontend](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#7-phase-4---service-to-frontend)

## [PART C - Flow across nodes](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_c/README.md)

- [Phase 1 Frontend to Service](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_c#8-phase-1---frontend-pod-to-service)
- [Phase 2 Service to Backend Pod](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_c#9-phase-2---service-to-backend-pod)
- [Phase 3 Backend Pod to Service](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_c#10-phase-3---backend-pod-to-service)
- [Phase 4 Service to Frontend](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_c#11-phase-4---service-to-frontend-pod)

## [PART D - ARP Process and Additional Info](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_d/README.md)
