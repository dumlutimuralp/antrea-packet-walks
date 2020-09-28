# Packet Walks with [Antrea CNI](https://github.com/vmware-tanzu/antrea)

This article explains the step by step packet walks for a simple application flow in the Open vSwitch (OVS) dataplane implemented by Antrea CNI within a Kubernetes cluster. The high level packet walks are already explained [here](https://github.com/vmware-tanzu/antrea/blob/master/docs/architecture.md#pod-networking), at Antrea' s project repo on Github.

## [Part A](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md)

1. [Antrea Implementation in a Worker Node](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#1-antrea-implementation-in-a-worker-node)
2. [OVS Pipeline](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#2-ovs-pipeline)
3. [Test Environment](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_a/README.md#3-test-environment)

## [Part B](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md)

1. [Scenario 1 Phase 1 Frontend to Service](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#4-scenario-1---phase-1---frontend-to-service)
2. [Scenario 1 Phase 2 Service to Backend Pod](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#5-scenario-1---phase-2---service-to-backend-pod)
3. [Scenario 1 Phase 3 Backend Pod to Service](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#6-scenario-1---phase-3---backend-pod-to-service)
4. [Scenario 1 Phase 4 Service to Frontend](https://github.com/dumlutimuralp/antrea-packet-walks/blob/master/part_b/README.md#7-scenario-1---phase-4---service-to-frontend)
