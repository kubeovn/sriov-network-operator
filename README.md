# sriov-network-operator

The Sriov Network Operator is designed to help the user to provision and configure SR-IOV nic in kubernetes cluster.

## Motivation

SR-IOV network is an optional feature of an kubernetes cluster. To make it work, it requires different components to be provisioned and configured accordingly. It makes sense to have one operator to coordinate those relevant components in one place, instead of having them managed by different operators. And also, to hide the complexity, we should provide an elegant user interface to simplify the process of enabling SR-IOV.

## Features

- Initialize the supported SR-IOV NIC types on selected nodes.
- Generate net-att-def CRs for SR-IOV CNI plugin
- Supports operation in a virtualized Kubernetes deployment
  - Discovers VFs attached to the Virtual Machine (VM)
  - Does not require attached of associated PFs
  - VFs can be associated to SriovNetworks by selecting the appropriate PciAddress as the RootDevice in the SriovNetworkNodePolicy

## Quick Start

For kube-ovn  offload  more detail on installing this operator, refer to the [quick-start](doc/quickstart-offload.md) guide.

## API

The SR-IOV network operator introduces following new CRDs:

- SriovNetwork

- SriovNetworkNodeState

- SriovNetworkNodePolicy

### SriovNetworkNodeState

The custom resource to represent the SR-IOV interface states of each host, which should only be managed by the operator itself.

- The ‘spec’ of this CR represents the desired configuration which should be applied to the interfaces 
- The ‘status’ contains current states of those PFs (baremetal only), and the states of the VFs. It helps user to discover SR-IOV network hardware on node, or attached VFs in the case of a virtual deployment.

The spec is rendered by sriov-policy-controller, and consumed by sriov-config-daemon. Sriov-config-daemon is responsible for updating the ‘status’ field to reflect the latest status, this information can be used as input to create SriovNetworkNodePolicy CR.

An example of SriovNetworkNodeState CR:

```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodeState
metadata:
  name: node1
  namespace: kube-system
spec:
  interfaces:
  - eSwitchMode: switchdev
    name: ens41f0np0
    numVfs: 3
    pciAddress: 0000:5f:00.0
    vfGroups:
    - policyName: policy
      vfRange: 0-2
  - eSwitchMode: switchdev
    name: ens41f1np1
    numVfs: 3
    pciAddress: 0000:5f:00.1
    vfGroups:
    - policyName: policy
      vfRange: 0-2
- Vfs:
    - deviceID: 1018
      driver: mlx5_core
      pciAddress: 0000:5f:00.2
      vendor: "15b3"
    - deviceID: 1018
      driver: mlx5_core
      pciAddress: 0000:5f:00.3
      vendor: "15b3"
    - deviceID: 1018
      driver: mlx5_core
      pciAddress: 0000:5f:00.4
      vendor: "15b3"
    deviceID: "1017"
    driver: mlx5_core
    linkSeed: 25000Mb/s
    linkType: ETH
    mac: 08:c0:eb:f4:85:ab
    mtu: 1500
    name: ens41f0np0
    numVfs: 3
    pciAddress: 0000:5f:00.0
    totalvfs: 3
    vendor: "15b3"
    
    - deviceID: 1018
      driver: mlx5_core
      pciAddress: 0000:5f:00.5
      vendor: "15b3"
    - deviceID: 1018
      driver: mlx5_core
      pciAddress: 0000:5f:00.6
      vendor: "15b3"
    - deviceID: 1018
      driver: mlx5_core
      pciAddress: 0000:5f:00.7
      vendor: "15b3"
    deviceID: "1017"
    driver: mlx5_core
    linkSeed: 25000Mb/s
    linkType: ETH
    mac: 08:c0:eb:f4:85:bb
    mtu: 1500
    name: ens41f1np1
    numVfs: 3
    pciAddress: 0000:5f:00.1
    totalvfs: 3
    vendor: "15b3"
```


From this example, in status field, the user can find out there are 2 SRIOV capable NICs on node 'node1'; in spec field, user can learn what the expected configure is generated from the combination of SriovNetworkNodePolicy CRs.  In the virtual deployment case, a single VF will be associated with each device.

### SriovNetworkNodePolicy

This CRD is the key of SR-IOV network operator. This custom resource should be managed by cluster admin, to instruct the operator to:
Render the spec of SriovNetworkNodeState CR for selected node, to configure the SR-IOV interfaces.  In virtual deployment, the VF interface is read-only.

An example of SriovNetworkNodePolicy CR:

```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: policy
  namespace: kube-system
spec:
  nodeSelector:
    feature.node.kubernetes.io/network-sriov.capable: "true"
  eSwitchMode: switchdev
  numVfs: 3
  nicSelector:
    pfNames:
      - ens41f0np0
      - ens41f1np1
```

In this example, user selected the nic ens41f0np0 and ens41f1np1, on nodes labeled with 'network-sriov.capable' equals 'true'. Then for those PFs, load 3 VFs each   to those virtual functions.  

In a virtual deployment: 
- The mtu of the PF is set by the underlying virtualization platform and cannot be changed by the sriov-network-operator.
- The numVfs parameter has no effect as there is always 1 VF
- The deviceType field depends upon whether the underlying device/driver is [native-bifurcating or non-bifurcating](https://doc.dpdk.org/guides/howto/flow_bifurcation.html) For example, the supported Mellanox devices support native-bifurcating drivers and therefore deviceType should be netdevice (default).  The support Intel devices are non-bifurcating and should be set to vfio-pci.

#### Multiple policies

When multiple SriovNetworkNodeConfigPolicy CRs are present, the `priority` field
(0 is the highest priority) is used to resolve any conflicts. Conflicts occur
only when same PF is referenced by multiple policies. The final desired
configuration is saved in `SriovNetworkNodeState.spec.interfaces`.

Policies processing order is based on priority (lowest first), followed by `name`
field (starting from `a`). Policies with same **priority** or **non-overlapping
VF groups** (when #-notation is used in pfName field) are merged, otherwise only
the highest priority policy is applied. In case of same-priority policies and
overlapping VF groups, only the last processed policy is applied.

## Components and design

This operator is split into 2 components:

- controller
- sriov-config-daemon

The controller is responsible for:

1. Read the SriovNetworkNodePolicy CRs and SriovNetwork CRs as input.
2. Render the spec of SriovNetworkNodeState CR for each node.

The sriov-config-daemon is responsible for:

1. Discover the SRIOV NICs on each node, then sync the status of SriovNetworkNodeState CR.
2. Take the spec of SriovNetworkNodeState CR as input to configure those NICs.

