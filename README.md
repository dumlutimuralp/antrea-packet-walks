# [Antrea](https://github.com/vmware-tanzu/antrea) Packet Walks 

_**This disclaimer informs readers that the views, thoughts, and opinions expressed in this series of posts belong solely to the author, and not necessarily to the author’s employer, organization, committee or other group or individual.**_

This article explains the step by step packet walks for a given application flow in the Open vSwitch (OVS) dataplane implemented by Antrea CNI within a Kubernetes cluster. The high level packet walks and the OVS Pipeline are already explained [here](https://github.com/vmware-tanzu/antrea/blob/master/docs/design/architecture.md#pod-networking) and [here](https://github.com/vmware-tanzu/antrea/blob/master/docs/design/ovs-pipeline.md) at Project Antrea' s page on Github.

The purpose of this article is to walk the reader through an actual flow pattern between a frontend pod to a Kubernetes service and explain the forwarding decisions made from that specific flow' s point of view with some other details and more wording.

_**Special thanks to [Quan Tian](https://github.com/tnqn), [Wenying Dong](https://github.com/wenyingd), [Ran Gu](https://github.com/gran-vmv), [Antonin Bas](https://github.com/antoninbas) for responding to my queries about specific OVS registers and forwarding steps.**_

## [PART A - Architecture and Test Environment](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md)

- [1. Antrea Implementation in a Worker Node](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#1-antrea-implementation-in-a-worker-node)
- [2. OVS Pipeline](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#2-ovs-pipeline)
- [3. Test Environment](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#3-test-environment)

## [PART B - Same Node](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md)

- [4. Frontend to Service](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#4-frontend-to-service)
- [5. Service to Backend Pod](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#5-service-to-backend-pod)
- [6. Backend Pod to Service](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#6-backend-pod-to-service)
- [7. Service to Frontend](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#7-service-to-frontend)

## [PART C - Different Nodes](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_c/README.md)

- [8. Frontend to Service](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_c#8-frontend-pod-to-service)
- [9. Service to Backend Pod](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_c#9-service-to-backend-pod)
- [10. Backend Pod to Service](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_c#10-backend-pod-to-service)
- [11. Service to Frontend](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_c#11-service-to-frontend-pod)

## [PART D - ARP](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_d/README.md)

- [12. ArpResponder](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_d/README.md#12-arpresponder-table-20)

## [PART E - Different Nodes - Pod to Pod](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_e/README.md#part-e)

- [13. Worker1 (Phase 1)](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_e#13-worker1-phase-1)
- [14. Worker2 (Phase 1)](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_e/README.md#14-worker2-phase-1)
- [15. Worker2 (Phase 2)](https://github.com/dumlutimuralp/antrea-packet-walks/tree/master/part_e#15-worker2-phase-2)
- [16. Worker1 (Phase 2)](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_e/README.md#16-worker-1-phase-2)

## [PART F - Client to Nodeport](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_f/README.md)

- [17. Client to Frontend](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_f/README.md#17-client-to-frontend)
- [18. Frontend to Client](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_f/README.md#18-frontend-to-client)

