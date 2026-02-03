# Milvus Primary-Standby Disaster Recovery and Failover User Guide

## Overview

Starting from v2.6, Milvus supports real-time incremental data replication across clusters through its built-in CDC (Change Data Capture) mechanism, enabling a **single-primary, multi-standby** disaster recovery architecture:

- **Primary Cluster**: The sole write endpoint, handling all read and write traffic.
- **Standby Cluster**: A hot standby node that continuously receives incremental data from the primary cluster and provides read-only query capabilities.

When the primary cluster fails, Milvus offers two failover strategies:

| Strategy | Use Case | Consistency Guarantee | Failover Speed |
|---|---|---|---|
| **Switchover (Planned)** | Primary partially degraded but still responsive | RPO=0, zero data loss | Depends on remaining data sync |
| **Force Failover** | Primary entirely unavailable (e.g., region-wide outage) | Availability-first; may lose a small amount of unsynced data | Seconds |

This document covers the usage of both **Switchover (Planned Failover)** and **Force Failover**.

## Prerequisites

- Milvus version >= v2.6.11
- Primary and standby clusters deployed, with a CDC replication relationship established (see [Appendix A: Setting Up CDC Replication](#appendix-a-setting-up-cdc-replication))
- Network connectivity between primary and standby clusters (required during CDC replication)

## Core Concepts

### CDC (Change Data Capture)

Milvus CDC is a distributed data replication component introduced in v2.6. It captures and forwards data change logs from the source cluster, enabling efficient, low-latency incremental data replication between primary and standby clusters.

Architecture:

- **Source Cluster**: The data source (primary cluster), supports read and write operations.
- **CDC Node**: Deployed alongside the source cluster. Once the replication relationship is established, it continuously consumes data change logs and replicates them to the target cluster.
- **Target Cluster**: The destination (standby cluster). Receives data changes from CDC Nodes and writes them to its own storage, completing incremental replication.

### Force Failover

Force Failover is the operation of forcibly promoting a standby cluster to become the new primary when the primary cluster is **entirely unavailable**:

- **Availability-first**: Does not wait for data to be fully synchronized; immediately promotes the standby to primary, restoring write capability within seconds.
- **Potential data loss**: Since it does not wait for synchronization to complete, any data written to the original primary but not yet replicated to the standby will be lost. The amount of data loss depends on the CDC replication lag at the time of failure.

### Switchover (Planned Failover)

Switchover is a planned role swap performed when the primary cluster is **partially degraded but still responsive**. The system ensures all data is fully synchronized before switching roles, guaranteeing RPO=0 strong consistency.

## Usage Guide

### Step 1: Establish Primary-Standby Replication

Before performing any failover operation, you must first establish a CDC replication relationship between the primary and standby clusters.

#### 1. Declare Cluster Information

```python
from pymilvus import MilvusClient

# Cluster information
cluster_a_addr = "http://<primary-host>:19530"
cluster_b_addr = "http://<standby-host>:19530"
cluster_a_token = "root:Milvus"
cluster_b_token = "root:Milvus"
cluster_a_id = "cluster-a"
cluster_b_id = "cluster-b"

# PChannel configuration (adjust based on your cluster setup)
pchannel_num = 16
cluster_a_pchannels = [f"{cluster_a_id}-rootcoord-dml_{i}" for i in range(pchannel_num)]
cluster_b_pchannels = [f"{cluster_b_id}-rootcoord-dml_{i}" for i in range(pchannel_num)]
```

#### 2. Define the Replication Topology

```python
config = {
    "clusters": [
        {
            "cluster_id": cluster_a_id,
            "connection_param": {
                "uri": cluster_a_addr,
                "token": cluster_a_token
            },
            "pchannels": cluster_a_pchannels
        },
        {
            "cluster_id": cluster_b_id,
            "connection_param": {
                "uri": cluster_b_addr,
                "token": cluster_b_token
            },
            "pchannels": cluster_b_pchannels
        }
    ],
    "cross_cluster_topology": [
        {
            "source_cluster_id": cluster_a_id,
            "target_cluster_id": cluster_b_id
        }
    ]
}
```

#### 3. Apply the Replication Configuration

Call `update_replicate_configuration` on both the primary and standby clusters:

```python
# Configure the primary cluster
client_a = MilvusClient(uri=cluster_a_addr, token=cluster_a_token)
client_a.update_replicate_configuration(**config)
client_a.close()

# Configure the standby cluster
client_b = MilvusClient(uri=cluster_b_addr, token=cluster_b_token)
client_b.update_replicate_configuration(**config)
client_b.close()
```

Once configured, incremental data from the primary cluster will be replicated to the standby cluster in real time via CDC.

---

### Step 2: Force Failover

When the primary cluster (A) suffers an unrecoverable failure (e.g., region-wide network outage, datacenter failure), follow these steps to forcibly promote the standby cluster (B) to the new primary.

#### Use Cases

- The primary cluster is entirely unavailable and cannot respond to any requests
- The expected recovery time is unacceptable and write capability must be restored immediately

#### Procedure

**Step 1: Confirm the Primary Is Unavailable**

Verify that the primary cluster A is indeed in an unrecoverable state and cannot be restored within an acceptable timeframe.

> **Warning**: Force Failover is an irreversible operation. Once executed, the original primary will be removed from the replication topology, and **any data not yet replicated to the standby will be permanently lost**. If the primary is only partially degraded and can still respond to requests, consider using [Switchover (Planned Failover)](#step-3-switchover-planned-failover) instead to achieve zero data loss.

**Step 2: Send the Failover Request to Standby Cluster B**

Construct a new replication topology configuration with `force_promote=True` to forcibly promote B to the primary:

```python
failover_config = {
    "clusters": [
        {
            "cluster_id": cluster_b_id,
            "connection_param": {
                "uri": cluster_b_addr,
                "token": cluster_b_token
            },
            "pchannels": cluster_b_pchannels
        }
    ],
    "cross_cluster_topology": [],  # Clear topology; B becomes a standalone primary
    "force_promote": True
}

client_b = MilvusClient(uri=cluster_b_addr, token=cluster_b_token)
client_b.update_replicate_configuration(**failover_config)
client_b.close()
```

**Step 3: Verify Failover Completion**

After the request is submitted, standby cluster B will automatically complete the role transition and enable write access (completes within seconds). Verify that B can accept writes:

```python
client_b = MilvusClient(uri=cluster_b_addr, token=cluster_b_token)
# Attempt a write operation to confirm B is now the primary
client_b.insert(collection_name="test_collection", data=[{"id": 1, "vector": [0.1]*128}])
client_b.close()
```

**Step 4: Redirect Application Traffic**

Update your application's connection endpoint from the original primary cluster A to the new primary cluster B. Once complete, the original primary cluster A can be decommissioned.

#### Failover Sequence Summary

```
 Primary A (Failed)                  Standby B
     |                                  |
     X Unavailable                      |
     |                          Receives failover request
     |                          Completes role transition (seconds)
     |                          Enables writes ← Traffic redirected to B
     |                                  |
     |                          (B is now the primary)
```

#### Data Loss Details

Force Failover prioritizes availability and does not wait for primary-standby data to be fully synchronized. Therefore, **data that was written to the original primary but not yet replicated to the standby via CDC will be lost**.

The amount of data loss depends on the CDC replication lag at the time of failure (i.e., RPO > 0). To minimize potential data loss:

- Continuously monitor CDC replication lag and keep it as low as possible
- Design idempotent write logic at the application layer so that writes can be safely retried after failover

---

### Step 3: Switchover (Planned Failover)

When the primary cluster (A) is partially degraded but still responsive, use Switchover to perform a zero-data-loss planned failover.

#### Use Cases

- The primary cluster has partial node failures (e.g., QueryNode or DataNode crashes) but can still respond to requests
- Planned maintenance switchover

> **Note**: If the primary cluster is entirely unavailable, use [Force Failover](#step-2-force-failover) instead.

#### Procedure

**Step 1: Send the Switchover Request to Primary Cluster A**

Construct a new replication topology (B→A) and send it to A:

```python
switchover_config = {
    "clusters": [
        {
            "cluster_id": cluster_a_id,
            "connection_param": {
                "uri": cluster_a_addr,
                "token": cluster_a_token
            },
            "pchannels": cluster_a_pchannels
        },
        {
            "cluster_id": cluster_b_id,
            "connection_param": {
                "uri": cluster_b_addr,
                "token": cluster_b_token
            },
            "pchannels": cluster_b_pchannels
        }
    ],
    "cross_cluster_topology": [
        {
            "source_cluster_id": cluster_b_id,
            "target_cluster_id": cluster_a_id
        }
    ]
}

# Send switchover request to A; A will demote to standby and reject new writes
client_a = MilvusClient(uri=cluster_a_addr, token=cluster_a_token)
client_a.update_replicate_configuration(**switchover_config)
client_a.close()
```

> If the request fails, it is safe to retry.

**Step 2: Send the Switchover Request to Standby Cluster B**

```python
# Send the same topology configuration to B
client_b = MilvusClient(uri=cluster_b_addr, token=cluster_b_token)
client_b.update_replicate_configuration(**switchover_config)
client_b.close()
```

B will wait for all remaining data from A to finish synchronizing, then automatically promote itself to the primary and enable writes.

**Step 3: Post-Switchover Actions**

Depending on the state of the original primary cluster A:

- **A is recoverable**: The switchover is complete. A becomes B's standby (topology: B→A), and the system resumes normal operation.
- **A is unrecoverable**: Remove the replication relationship and decommission cluster A.

#### Switchover Sequence Summary

```
 Primary A                              Standby B
     |                                      |
  Receives switchover request               |
  Demotes to standby, rejects writes        |
     |-------- CDC continues sync ------->  |
     |-------- Remaining data synced --->  Receives switchover request
     |                                  Confirms sync complete
     |                                  Promotes to primary, enables writes
     |                                      |
     |         (RPO=0, zero data loss)      |
```

---

## Best Practices

### 1. Failure Assessment and Strategy Selection

```
Primary Cluster Failure
  ├── Can the primary still respond to requests?
  │     ├── Yes → Use Switchover (planned, RPO=0)
  │     └── No  → Is the primary entirely unavailable?
  │                ├── Yes → Use Force Failover (may lose data)
  │                └── No  → Wait for recovery or investigate
  └── Failure Type
        ├── Partial node failure  → Prefer Switchover
        ├── Region-wide outage    → Force Failover
        └── Datacenter failure    → Force Failover
```

### 2. Standard Post-Failover Procedure

1. **Verify the new primary**: Confirm that the new primary cluster B can handle reads and writes normally
2. **Redirect application traffic**: Update the application's connection endpoint to the new primary cluster B
3. **Decommission the original primary**: The original primary cluster is no longer needed

### 3. Minimizing Data Loss During Force Failover

The amount of data lost during a Force Failover equals the CDC replication lag at the moment of failure. The following measures can effectively reduce this risk:

- **Monitor CDC replication lag**: Keep the lag within seconds; lower lag means less potential data loss during failover
- **Implement idempotent writes at the application layer**: Design idempotent write logic so that unacknowledged writes can be safely retried after failover
- **Prefer Switchover**: Only use Force Failover when the primary cluster is completely unavailable

---

## FAQ

### Q1: Does Force Failover cause data loss?

Yes. Force Failover prioritizes availability. **Data that has not yet been replicated to the standby will be lost.** The amount of data loss depends on the CDC replication lag at the time of failure. If your application has strict zero-data-loss requirements, ensure the primary cluster is still responsive and use Switchover instead.

### Q2: How long does a Failover take?

It is expected to complete within **seconds**.

### Q3: What happens to the original primary after Failover?

The original primary is removed from the replication topology and can be decommissioned.

### Q4: How can I minimize data loss during Failover?

Continuously monitor CDC replication lag and keep it within seconds.

### Q5: Does CDC replication affect the primary cluster's read/write performance?

No significant impact. CDC runs asynchronously and does not consume resources on the primary data path.

### Q6: Will data be lost if the CDC service crashes?

No. CDC supports resumable replication, ensuring events are neither lost nor duplicated.

---

## API Reference

### update_replicate_configuration

Updates the cluster's replication topology configuration. Used to establish, modify, or remove replication relationships, as well as to execute Switchover and Force Failover.

```python
client.update_replicate_configuration(
    clusters=[...],                  # List of clusters
    cross_cluster_topology=[...],    # Cross-cluster replication topology
    force_promote=False              # Whether to force failover (default: False)
)
```

| Parameter | Type | Description |
|---|---|---|
| `clusters` | list | List of participating clusters, each containing cluster_id, connection_param, and pchannels |
| `cross_cluster_topology` | list | Replication topology specifying source and target clusters |
| `force_promote` | bool | Set to `True` to execute Force Failover |

---

## Appendix A: Setting Up CDC Replication

Follow these steps to establish a complete CDC replication relationship:

### 1. Deploy the Source Cluster (with CDC Enabled)

When deploying via Milvus Operator, enable CDC in the spec:

```yaml
apiVersion: milvus.io/v1beta1
kind: Milvus
metadata:
  name: primary-cluster
  namespace: milvus
spec:
  mode: cluster
  components:
    image: milvusdb/milvus:v2.6.5
    cdc:
      replicas: 1
      config: {}
```

### 2. Deploy the Target Cluster

Deploy the target (standby) cluster in the same manner and ensure network connectivity.

### 3. Establish the Replication Relationship

Refer to the [Establish Primary-Standby Replication](#step-1-establish-primary-standby-replication) section in this document.

---

## Appendix B: Glossary

| Term | Description |
|---|---|
| **CDC** | Change Data Capture, a mechanism for capturing and replicating data changes |
| **PChannel** | Physical Channel |
| **Switchover** | Planned primary-standby failover, consistency-first (RPO=0) |
| **Force Failover** | Forced primary-standby failover, availability-first (potential data loss) |
| **RPO** | Recovery Point Objective, a measure of acceptable data loss |
| **RTO** | Recovery Time Objective, a measure of acceptable service downtime |
