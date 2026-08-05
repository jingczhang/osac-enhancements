# Table of Contents

- [Multi-Tenancy Model](#multi-tenancy-model)
- [First-Class vs Internal Resources](#first-class-vs-internal-resources)
- [Deployment Example: A Tenant of Two Hardware Groups](#deployment-example-a-tenant-of-two-hardware-groups)
- [Implementation Details](#implementation-details)
  - [Add VPC](#add-vpc)
  - [Change VirtualNetwork](#change-virtualnetwork)
  - [Add ServerCluster](#add-servercluster)

# Multi-Tenancy Model

CaaS, BMaaS, and VMaaS share one assignment model. A tenant is given nodes —
bare metal or virtual — and those nodes are grouped by hardware class (GPU
servers, storage servers, CPU workers, and so on).

- Nodes in the **same group** share one east-west isolation domain. RDMA and
  collective traffic stay inside the group (Ethernet VNI, InfiniBand PKey,
  NVLink partition).
- Nodes in **different groups of the same tenant** share one north-south
  isolation domain: a supernet. Each group is carved a unique child subnet,
  so groups reach each other at L3 (storage, ingress) without joining each
  other's east-west fabric.

OSAC ServerCluster is that group. It applies the same E-W isolation across
Ethernet, InfiniBand, and NVLink. The tenant VPC holds the N-S supernet;
additional ServerClusters in that VPC receive the next child prefix.

Netris supports this directly — when creating a Netris Server Cluster, the
following data-plane primitives are created automatically:

- **VPC** — or references an existing one (tenant VRF; holds the N-S supernet)
- **V-Nets** (per-cluster VXLAN segments — E-W, N-S, mgmt — as defined in the template)
- **IP subnets** (child prefixes carved from the N-S supernet for each cluster)
- **Switch port assignments**
- **IB PKeys** (if `netris-ufm` in template)
- **NVLink partitions** (if `netris-nvlink` in template)

# First-Class vs Internal Resources

In Netris, the Ethernet V-Net is a first-class API object — users can create
V-Nets, add/remove switch ports, and delete them independently. IB PKeys and
NVLink partitions are **not** first-class — they have no standalone API. They
exist only as internal side effects of Server Cluster lifecycle operations.

| | Ethernet V-Net | IB PKey | NVLink Partition |
|---|---|---|---|
| First-class Netris API | Yes | No | No |
| User can create directly | Yes | No | No |
| User can attach/detach servers | Yes | No | No |
| Managed through | [V-Net API](https://www.netris.io/docs/en/latest/vnet.html), Labels, or Server Cluster | Server Cluster only | Server Cluster only |
| Backend | Netris switches directly | [Netris-UFM plugin](https://www.netris.io/docs/en/latest/netris-ufm-integration.html) → NVIDIA UFM | [Netris-NMX plugin](https://www.netris.io/docs/en/latest/netris-nvlink-integration.html) → NVIDIA NMX-C |

This is a deliberate design: Server Cluster guarantees that Ethernet, IB, and
NVLink partitions stay in sync. Exposing individual PKey/partition APIs would
let users create inconsistent states across fabrics.

OSAC ServerCluster offers the same uniform isolation for all fabric managers.
For fabric managers that lack this holistic capability, the osac-operator can
build equivalent orchestration:
- Switch API calls for Ethernet V-Nets/VLANs
- UFM API calls for IB PKeys
- NVLink Fabric Manager API calls for GPU partitions

Hence, Phase 1 can focus on Netris integration — which delivers multi-fabric
(Ethernet, IB, NVLink) via a single API. Phase 2 builds equivalent
orchestration for other fabric managers, informed by Phase 1 patterns.

# Deployment Example: A Tenant of Two Hardware Groups

Single tenant with two hardware groups. Each group has its own east-west
isolation; both share the tenant N-S supernet.

```text
┌─────────────────────────────────────────────────────────────────────────┐
│  VPC: tenant-a                                                          │
│  N-S supernet: 10.200.0.0/16                                            │
│                                                                         │
│  ServerCluster: gpu-a (nvlink: true)                                    │
│    servers: gpu-01..gpu-32                                              │
│    VirtualNetwork: gpu-ew (type: L3)     E-W isolation for this group   │
│    VirtualNetwork: gpu-ns (type: L2)     child 10.200.0.0/24            │
│    NVLink Partition: nvl-gpu-a                                          │
│                                                                         │
│  ServerCluster: storage-a                                               │
│    servers: stor-01..stor-16                                            │
│    VirtualNetwork: storage-ew (type: L2) E-W isolation for this group   │
│    VirtualNetwork: storage-ns (type: L2) child 10.200.1.0/24            │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  VPC: management (provider-owned)                                       │
│                                                                         │
│  VirtualNetwork: mgmt (type: L2)                                        │
│    eth11, IPMI/BMC (per-V-Net VPC mapping)                              │
│    persists across tenant reassignments                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

In this deployment:
- `gpu-a` and `storage-a` are hardware groups, each with its own E-W domain.
  GPU RDMA stays on `gpu-ew`; storage replication stays on `storage-ew`.
- Both groups sit in VPC `tenant-a` and share the N-S supernet
  `10.200.0.0/16`. Netris carves a unique child subnet per ServerCluster
  (`10.200.0.0/24`, `10.200.1.0/24`). GPU nodes reach storage at L3 over N-S
  without joining the storage E-W fabric.
- Setting `nvlink: true` on `gpu-a` creates partition `nvl-gpu-a` for that
  group only — same boundary as the group's Ethernet E-W isolation.
- Same-VPC V-Nets route to each other with no peering. Separate VPCs would
  need peering even for this tenant's GPU-to-storage path.
- The management VPC is provider-owned. The per-V-Net VPC mapping attaches
  each server's BMC interface to the management VPC — the provider never loses
  out-of-band access, even when servers are reassigned between tenants.

# Implementation Details

Creating OSAC networking resources (VPC, VirtualNetwork) does not trigger
fabric manager actions — there is nothing for the backend to do without
switch ports. The ServerCluster is what provisions the fabric by attaching
servers and switch ports to these resources via NetworkAttachment semantics
(same concept as BareMetalNetworkAttachment).

This aligns with Netris V-Net design: [to build a V-Net you need to supply
a list of switch ports](https://www.netris.io/docs/en/latest/vnet.html).

## Add VPC

ServerCluster requires a VPC — a routing domain that groups multiple
VirtualNetworks under a single isolation boundary. VPC is a new OSAC
networking resource that sits above VirtualNetwork in the hierarchy:

```text
NetworkClass (per deployment, provider-only)

VPC (tenant-managed, routing/isolation domain)
  └── VirtualNetwork (segment within VPC — E-W, N-S, mgmt, etc.)
       └── Subnet (IP range within VN)
```

VirtualNetworks within the same VPC can route to each other — no peering
needed.

### VPC Proto (fulfillment-service)

```protobuf
message VPCSpec {
  string name = 1;
  string network_class = 2;  // NetworkClass reference, required, immutable
}

message VPCStatus {
  string state = 1;          // Pending | Ready | Failed
  string backend_id = 2;     // Fabric manager's VPC ID (e.g. Netris VPC ID)
}
```

## Change VirtualNetwork

VirtualNetwork gains `vpc` and `type` fields. All VirtualNetworks must belong
to a VPC and inherit the NetworkClass from it. The `type` field determines
the forwarding mode:

| Type | Forwarding | Use Case |
|------|-----------|----------|
| `L3` | Routed (IP fabric, ECMP) | RoCE east-west, GPU-to-GPU RDMA |
| `L2` | Switched (bridged VXLAN) | North-south, storage access, management |

```protobuf
message VirtualNetworkSpec {
  string vpc = 1;           // VPC ID, required, immutable (inherits NetworkClass from VPC)
  string type = 2;          // "L2" or "L3", required, immutable
  string ipv4_cidr = 3;     // optional, immutable
  string ipv6_cidr = 4;     // optional, immutable
}
```

L3 VirtualNetworks provide IP routing with ECMP across the fabric — required
for RoCE/RDMA workloads where lossless, routed transport between servers is
essential. L2 VirtualNetworks provide a flat bridged segment — suitable for
north-south traffic, storage access, and management where simple L2
reachability is sufficient.

## Add ServerCluster

### Proto (fulfillment-service)

```protobuf
message ServerClusterNetworkAttachment {
  string virtual_network = 1;  // VirtualNetwork ID, required
  string subnet = 2;           // Subnet ID within the VN, optional (auto-allocated if omitted)
}

message ServerClusterSpec {
  string name = 1;
  string vpc = 2;                                        // VPC ID, required
  repeated ServerClusterNetworkAttachment network_attachments = 3;
  repeated string servers = 4;                           // Explicit membership; names must exist in fabric manager inventory
  bool nvlink = 5;                                       // Enable NVLink partition for this cluster
}

message ServerClusterStatus {
  string state = 1;                                      // Pending | Provisioning | Ready | Failed
  string backend_id = 2;                                 // Fabric manager's Server Cluster ID (e.g. Netris SC ID)
  string nvlink_partition = 3;                           // NVLink partition ID (populated when nvlink=true)
  repeated string servers = 4;                           // Confirmed members from fabric manager inventory
  repeated ServerClusterNetworkAttachmentStatus network_attachment_statuses = 5;
}

message ServerClusterNetworkAttachmentStatus {
  string virtual_network = 1;
  string subnet = 2;
  string state = 3;             // Pending | Ready | Failed
  string backend_vnet_id = 4;  // Fabric manager's V-Net ID for this segment
}
```

### Operator CRD

```go
type ServerCluster struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec              ServerClusterSpec   `json:"spec"`
    Status            ServerClusterStatus `json:"status,omitempty"`
}

type ServerClusterSpec struct {
    VPC                string                            `json:"vpc"`
    NetworkAttachments []ServerClusterNetworkAttachment  `json:"networkAttachments"`
    Servers            []string                          `json:"servers"`
    NVLink             bool                              `json:"nvlink,omitempty"`
}

type ServerClusterNetworkAttachment struct {
    VirtualNetwork string `json:"virtualNetwork"`
    Subnet         string `json:"subnet,omitempty"`
}

type ServerClusterStatus struct {
    ObservedGeneration        int64                                    `json:"observedGeneration,omitempty"`
    State                     string                                   `json:"state,omitempty"`
    BackendID                 string                                   `json:"backendID,omitempty"`
    NVLinkPartition           string                                   `json:"nvlinkPartition,omitempty"`
    Servers                   []string                                 `json:"servers,omitempty"`
    NetworkAttachmentStatuses []ServerClusterNetworkAttachmentStatus   `json:"networkAttachmentStatuses,omitempty"`
    Conditions                []metav1.Condition                       `json:"conditions,omitempty"`
}

type ServerClusterNetworkAttachmentStatus struct {
    VirtualNetwork string `json:"virtualNetwork"`
    Subnet         string `json:"subnet"`
    State          string `json:"state"`
    BackendVNetID  string `json:"backendVNetID,omitempty"`
}
```

### Example: Two ServerClusters in one VPC

Same layout as the deployment example: `gpu-a` and `storage-a` in VPC
`tenant-a`. Each cluster has its own E-W VirtualNetwork. N-S children
`10.200.0.0/24` and `10.200.1.0/24` are carved from supernet `10.200.0.0/16`.

#### Prerequisites (VirtualNetwork + Subnet via osac CLI)

```bash
# GPU E-W (L3 routed RoCE/Spectrum-X)
osac create virtualnetwork --vpc tenant-a --type L3 --cidr 10.100.0.0/16 \
  --name gpu-ew
osac create subnet --virtual-network gpu-ew --cidr 10.100.1.0/24 \
  --name gpu-ew-subnet

# GPU N-S child of tenant supernet 10.200.0.0/16
osac create virtualnetwork --vpc tenant-a --type L2 --cidr 10.200.0.0/24 \
  --name gpu-ns
osac create subnet --virtual-network gpu-ns --cidr 10.200.0.0/24 \
  --name gpu-ns-subnet

# Storage E-W (L2)
osac create virtualnetwork --vpc tenant-a --type L2 --cidr 10.110.0.0/24 \
  --name storage-ew
osac create subnet --virtual-network storage-ew --cidr 10.110.0.0/24 \
  --name storage-ew-subnet

# Storage N-S child of the same supernet
osac create virtualnetwork --vpc tenant-a --type L2 --cidr 10.200.1.0/24 \
  --name storage-ns
osac create subnet --virtual-network storage-ns --cidr 10.200.1.0/24 \
  --name storage-ns-subnet
```

#### CLI

```bash
osac create servercluster --vpc tenant-a \
  --servers gpu-01,gpu-02,...,gpu-32 \
  --network-attachment virtualNetwork=gpu-ew,subnet=gpu-ew-subnet \
  --network-attachment virtualNetwork=gpu-ns,subnet=gpu-ns-subnet \
  --nvlink \
  --name gpu-a

osac create servercluster --vpc tenant-a \
  --servers stor-01,stor-02,...,stor-16 \
  --network-attachment virtualNetwork=storage-ew,subnet=storage-ew-subnet \
  --network-attachment virtualNetwork=storage-ns,subnet=storage-ns-subnet \
  --name storage-a
```

#### ServerCluster CRs

```yaml
apiVersion: osac.openshift.io/v1alpha1
kind: ServerCluster
metadata:
  name: gpu-a
  namespace: osac
spec:
  vpc: tenant-a
  servers:
    - gpu-01
    - gpu-02
    # ...
    - gpu-32
  networkAttachments:
    - virtualNetwork: gpu-ew
      subnet: gpu-ew-subnet
    - virtualNetwork: gpu-ns
      subnet: gpu-ns-subnet
  nvlink: true
---
apiVersion: osac.openshift.io/v1alpha1
kind: ServerCluster
metadata:
  name: storage-a
  namespace: osac
spec:
  vpc: tenant-a
  servers:
    - stor-01
    - stor-02
    # ...
    - stor-16
  networkAttachments:
    - virtualNetwork: storage-ew
      subnet: storage-ew-subnet
    - virtualNetwork: storage-ns
      subnet: storage-ns-subnet
```

### Expected Status (after reconciliation)

The osac-operator populates each ServerCluster status by calling fabric
manager GET APIs if they are not returned from the create response. The two
CRs reconcile independently against the same VPC.

```yaml
apiVersion: osac.openshift.io/v1alpha1
kind: ServerCluster
metadata:
  name: gpu-a
  namespace: osac
spec:
  vpc: tenant-a
  servers:
    - gpu-01
    - gpu-02
    # ...
    - gpu-32
  networkAttachments:
    - virtualNetwork: gpu-ew
      subnet: gpu-ew-subnet
    - virtualNetwork: gpu-ns
      subnet: gpu-ns-subnet
  nvlink: true
status:
  observedGeneration: 1
  state: Ready
  backendID: "netris-sc-4a7b2c"
  servers:
    - gpu-01
    - gpu-02
    # ...
    - gpu-32
  nvlinkPartition: nvl-gpu-a
  networkAttachmentStatuses:
    - virtualNetwork: gpu-ew
      subnet: gpu-ew-subnet
      state: Ready
      backendVNetID: "netris-vnet-ew-9f3d"
    - virtualNetwork: gpu-ns
      subnet: gpu-ns-subnet
      state: Ready
      backendVNetID: "netris-vnet-ns-1e8a"
  conditions:
    - type: ServersResolved
      status: "True"
      reason: InventoryConfirmed
      message: "32 servers found in fabric manager inventory"
    - type: NetworkAttachmentsReady
      status: "True"
      reason: AllAttached
      message: "All network attachments configured for 32 servers"
    - type: NVLinkReady
      status: "True"
      reason: PartitionCreated
      message: "NVLink partition nvl-gpu-a created for 32 servers"
---
apiVersion: osac.openshift.io/v1alpha1
kind: ServerCluster
metadata:
  name: storage-a
  namespace: osac
spec:
  vpc: tenant-a
  servers:
    - stor-01
    - stor-02
    # ...
    - stor-16
  networkAttachments:
    - virtualNetwork: storage-ew
      subnet: storage-ew-subnet
    - virtualNetwork: storage-ns
      subnet: storage-ns-subnet
status:
  observedGeneration: 1
  state: Ready
  backendID: "netris-sc-8c1d4e"
  servers:
    - stor-01
    - stor-02
    # ...
    - stor-16
  networkAttachmentStatuses:
    - virtualNetwork: storage-ew
      subnet: storage-ew-subnet
      state: Ready
      backendVNetID: "netris-vnet-ew-2b6c"
    - virtualNetwork: storage-ns
      subnet: storage-ns-subnet
      state: Ready
      backendVNetID: "netris-vnet-ns-7a4f"
  conditions:
    - type: ServersResolved
      status: "True"
      reason: InventoryConfirmed
      message: "16 servers found in fabric manager inventory"
    - type: NetworkAttachmentsReady
      status: "True"
      reason: AllAttached
      message: "All network attachments configured for 16 servers"
```

### Reconciliation Flow

The osac-operator owns each ServerCluster CR independently. `gpu-a` and
`storage-a` are two CRs that share VPC `tenant-a`; they do not share E-W
isolation or NVLink. Reconciliation matches desired spec against the fabric
manager's actual state, populating status from GET operations on the backend.

**Create / Update:**

1. **Validate servers** — look up each name in `spec.servers` in the fabric
   manager inventory. Populate `status.servers` with the confirmed list. Set
   condition `ServersResolved`. Missing names fail the CR with
   `ServersResolved=False`.
2. **Validate VPC** — verify VPC exists and is owned by the tenant.
3. **Create Server Cluster** — dispatch to fabric manager (e.g., Netris POST
   server-cluster with template, VPC, server list). Store returned ID in
   `status.backendID`.
4. **Poll backend status** — GET the Server Cluster from the fabric manager
   until it reports Active. For Netris, this also activates all V-Nets defined
   in the template.
5. **Populate network attachment statuses** — GET each V-Net associated with the Server
   Cluster. For each, store `backendVNetID` and `state` (Active → Ready).
   Set condition `NetworkAttachmentsReady`.
6. **If `nvlink: true`** — GET the NVLink partition from the fabric manager
   (Netris-NMX plugin → NMX-C). Store partition ID in `status.nvlinkPartition`.
   Set condition `NVLinkReady`.
7. **Set `observedGeneration`** — mark reconciliation complete.

**What happens with Netris (Phase 1):**

| Operator action | Netris API call | Response used for |
|-----------------|-----------------|-------------------|
| Create Server Cluster | `POST /api/v2/server-clusters` | `status.backendID` |
| Poll until Active | `GET /api/v2/server-clusters/{id}` | `status.state` |
| Get V-Net details | `GET /api/v2/vnets?serverCluster={id}` | `status.networkAttachmentStatuses[].backendVNetID`, `.state` |
| Get NVLink partition | Netris-NMX API | `status.nvlinkPartition` |
| Resize (servers changed) | `PUT /api/v2/server-clusters/{id}` | Re-poll status |
| Delete | `DELETE /api/v2/server-clusters/{id}` | Clear status |

**Delete:**

Remove NVLink partition, delete Server Cluster (Netris removes all V-Net port
assignments and partitions), clear status.

### What the Fabric Manager Creates

For the example above, the fabric manager (Netris) creates:

| Cluster | Fabric | Primitive | Scope |
|---------|--------|-----------|-------|
| `gpu-a` | Ethernet (L3) | V-Net `gpu-ew` routed VXLAN, ECMP, ports gpu-01..gpu-32 eth1-eth8 | GPU E-W |
| `gpu-a` | Ethernet (L2) | V-Net `gpu-ns` bridged VXLAN, ports gpu-01..gpu-32 eth9-eth10 | N-S `10.200.0.0/24` |
| `gpu-a` | NVLink | Partition `nvl-gpu-a`, GPU UIDs of gpu-01..gpu-32 | GPU-to-GPU |
| `storage-a` | Ethernet (L2) | V-Net `storage-ew` bridged VXLAN, ports stor-01..stor-16 | Storage E-W |
| `storage-a` | Ethernet (L2) | V-Net `storage-ns` bridged VXLAN, ports stor-01..stor-16 eth9-eth10 | N-S `10.200.1.0/24` |

### Server Validation

- `vpc`: must exist, must be owned by the requesting tenant.
- `networkAttachments`: at least one entry required. Each `virtualNetwork` must exist within
  the VPC. If `subnet` is specified, it must belong to the referenced
  VirtualNetwork and be Ready.
- `servers`: non-empty. Each name must exist in the fabric manager inventory
  and must not already be a dedicated member of another ServerCluster. If any
  name is missing or double-booked, the CR enters Failed state with condition
  `ServersResolved=False`.
- `nvlink`: if true, all servers must have NVLink-capable GPUs
  (validated against fabric manager inventory). The NVLink partition is meaningful
  for servers that have NVLink-capable GPUs.
- Immutability: `vpc` is immutable after creation. `servers`, `networkAttachments`, and
  `nvlink` are mutable (add/remove members, add/remove network attachments,
  enable NVLink on an existing cluster).
