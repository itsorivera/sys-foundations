Production-grade diagram contrasting **Physical Local Disks**, **Networked Virtual Disks (Block Emulation)**, and **Cloud-Native Log-Structured Storage (e.g., Aurora / AlloyDB)**.

```mermaid
flowchart TD
    subgraph Scenario1 [Scenario 1: Local Physical Disk / Ephemeral Instance Store]
        direction TB
        subgraph HostA [Physical Server Host A]
            VM1[Virtual Machine / Application OS]
            PCIeBus[PCIe / NVMe Bus - Direct Hardware Wire]
            LocalNVMe[(Physical NVMe SSD<br/>Attached to Motherboard)]
            
            VM1 <-->|Direct I/O read/write<br/>Latency: ~10 to 50 µs| PCIeBus
            PCIeBus <--> LocalNVMe
        end
        Note1[<b>Characteristics:</b><br/>• Ultra-low latency<br/>• Tied to physical hardware<br/>• Ephemeral: Host crash = Data loss risk]
    end

    subgraph Scenario2 [Scenario 2: Virtual Disk Storage / Network Block Device - EBS / Persistent Disk]
        direction TB
        subgraph HostB [Compute Rack: Physical Server Host B]
            VM2[Virtual Machine / Legacy Database Engine]
            OSBlockDriver[OS Block Layer Emulation<br/>/dev/xvdf - 4 KiB Block Interface]
            HypervisorNIC[Hypervisor Virtual NIC / Host Adapter]
            
            VM2 -->|POSIX write/fsync call| OSBlockDriver
            OSBlockDriver -->|Translates block I/O into network frames| HypervisorNIC
        end

        subgraph DatacenterFabric [Datacenter Network Fabric]
            NetworkSwitch[Physical Switches, Spine-Leaf & Fiber Optics<br/>Protocol: NVMe-oF / Proprietary iSCSI]
        end

        subgraph StorageCluster [Dedicated Storage Fleet: Remote Server Rack]
            StorageNodeController[Storage Fleet Controller / Software Daemon]
            DiskR1[(Physical SSD Replica 1)]
            DiskR2[(Physical SSD Replica 2)]
            DiskR3[(Physical SSD Replica 3)]
            
            StorageNodeController --> DiskR1
            StorageNodeController --> DiskR2
            StorageNodeController --> DiskR3
        end

        HypervisorNIC <-->|Network round-trip call per I/O<br/>Latency: ~1 to 3 ms| NetworkSwitch
        NetworkSwitch <-->|TCP / RDMA packets| StorageNodeController
        
        Note2[<b>Characteristics:</b><br/>• Decoupled compute & storage<br/>• Easy attach/detach & snapshots<br/>• Network jitter sensitivity<br/>• Double-write & block emulation overhead]
    end

    subgraph Scenario3 [Scenario 3: Cloud-Native Redesign - 'The Log is the Database']
        direction TB
        subgraph ComputeNode [Compute Node: DB Engine Instance]
            DBEngine[Custom Cloud DB Engine<br/>e.g., Aurora / AlloyDB Head]
            WALWriter[WAL Generator<br/>Writes only redo-log diffs]
            DBEngine --> WALWriter
        end

        subgraph StorageLayer [Distributed Log-Structured Storage Nodes]
            LogRecv[Log Processing Daemons]
            LogQuorum[(6-way Quorum Storage Slices)]
            LogRecv --> LogQuorum
        end

        WALWriter ==>|Asynchronous network stream: Log diffs ONLY<br/>No full 4-8 KiB dirty page pushes| LogRecv
        Note3[<b>Characteristics:</b><br/>• Eliminates virtual block emulation<br/>• Minimal network payload<br/>• In-storage background page reconstruction]
    end

    Scenario1 ~~~ Scenario2
    Scenario2 ~~~ Scenario3
```

---

## Architectural Takeaway Matrix

| Architecture Pattern | Where the Bytes Physically Land | Communication Mechanism | Core Bottleneck / Risk | Primary Architectural Fit |
| :--- | :--- | :--- | :--- | :--- |
| **Local Physical Disk** (*Instance Store*) | Inside the same bare-metal chassis as your compute core. | Local motherboard PCIe/NVMe controller. | **Node Durability:** Host hardware failure drops the storage. | Caches, ScyllaDB/Cassandra with application-level replication, HFT scratchpads. |
| **Virtual Disk** (*EBS, Azure Managed Disk*) | In a separate storage rack across the data center. | Block emulation over high-speed Ethernet (NVMe-oF/iSCSI). | **Network Overhead:** Every file `fsync()` incurs a network round-trip. | Lift-and-shift legacy apps, general VMs, databases needing instant volume detachment. |
| **Cloud-Native Storage** (*Aurora, AlloyDB*) | Fleet of storage nodes applying redo logs in background. | Custom log-streaming RPCs bypassing block drivers. | **Cloud Vendor Lock-in:** Tied to proprietary engine primitives. | High-concurrency relational OLTP without the performance penalties of virtual block devices. |