# AWS Storage Options: Local Storage, EBS, and EFS

## Key Points

- **Local Storage** is ephemeral, physically attached to the EC2 instance, and data is lost when the instance stops or terminates.

- **EBS (Elastic Block Store)** is network-attached block storage, presented as a local disk to EC2 instances via the hypervisor (often using NVMe on Nitro instances). It offers high performance, persistence, and is suitable for single-instance workloads.

  Amazon EBS (Elastic Block Store) volumes **could only be attached to a single EC2 instance at a time**. However, AWS introduced a feature called **EBS Multi-Attach** for certain EBS volume types, which allows a single EBS volume to be attached to multiple EC2 instances simultaneously.

  **EBS Multi-Attach** is supported only for

  - io1 and
  - io2 Provisioned IOPS SSD volumes

  Note #1: **io1/io2 volumes supports the necessary locking and coordination mechanisms for safe multi-instance access**, while other volume types do not.

  Note #2: All attached instances must be in the same AZ

- **EFS (Elastic File System)** is network-attached file storage, accessible via NFS by multiple EC2 instances simultaneously. It is ideal for shared, scalable file storage.

## EBS Technical Details

- EBS uses a proprietary AWS protocol (similar to iSCSI) over the AWS internal network.
- The hypervisor (Nitro for modern instances) exposes EBS volumes as block devices, making them appear as regular disks to the EC2 OS.
- EBS volumes can achieve high IOPS and low latency due to dedicated network bandwidth and NVMe interfaces.

## Comparison Table


| Feature         | Local Storage         | EBS (Elastic Block Store)         | EFS (Elastic File System)         |
|-----------------|----------------------|-----------------------------------|-----------------------------------|
| Type            | Instance-attached     | Network block storage             | Network file storage (NFS)        |
| Protocol        | Direct SATA/NVMe      | AWS proprietary (block, NVMe/iSCSI-like) | NFS                              |
| Persistence     | Ephemeral             | Persistent                        | Persistent                        |
| Access          | Single instance       | Single instance (multi-attach limited) | Multiple instances               |
| Scalability     | Fixed per instance    | Resize manually                   | Auto-scales with usage            |
| Performance     | Very high, low latency| High IOPS, low latency            | Scales with clients, higher latency|
| Use Case | Temporary data, cache | Databases, app data, root volume | Shared file storage, web servers |