# Chapter 8.1 The Trouble with Distributed Systems: Faults and Partial Failures

Single-Node vs. Distributed Systems When running software on a single computer, the system is designed to be deterministic: it either functions correctly or crashes completely (e.g., a kernel panic). This is a deliberate design choice to hide the messy physical reality of hardware; computers present an idealized model where operations are mathematically perfect.

In contrast, distributed systems must confront the physical world. They suffer from partial failures, where some components break unpredictably while others continue to function. Because network timing and message delivery are nondeterministic, it is often impossible to know if an operation succeeded or failed. This nondeterminism is the primary challenge in distributed systems

<br>

---

<br>

### Cloud Computing and Supercomputing

There is a spectrum of philosophies for building large-scale systems, ranging from High-Performance Computing (HPC) to Cloud Computing.

- Supercomputing (HPC): Typically used for computationally intensive tasks like weather forecasting. These systems treat a cluster like a single, massive machine. If a node fails, the standard strategy is to stop the entire workload and restart from the last checkpoint.
- Cloud Computing: Associated with multi-tenant datacenters and commodity hardware. These systems accept partial failure and must handle faults in software without stopping the entire service.

Requirements for Internet Services Internet services differ significantly from supercomputers because they must serve users with low latency and high availability. The "stop-and-restart" strategy of HPC is unacceptable for online services.

Key differences include:

- Availability: Online services cannot simply halt for repairs; they must process requests at any time.
- Hardware: Cloud services use commodity hardware, which is cheaper but has higher failure rates than the specialized hardware used in HPC.
- Network: Datacenters use IP and Ethernet (often in Clos topologies), whereas supercomputers use specialized topologies (like meshes) optimized for known communication patterns.
- Scale: In large systems, it is reasonable to assume something is always broken. Escalating every partial failure to a total crash would result in constant downtime.
- Operations: Fault tolerance enables maintenance tasks like rolling upgrades (restarting one node at a time) without interrupting the service.
- Geography: Distributed deployments often communicate over the public internet, which is slower and less reliable than the local networks assumed by supercomputers.

Building Reliable Systems from Unreliable Components It is possible to build a reliable system on top of an unreliable underlying base. This is a foundational idea in computing. For example, TCP (Transmission Control Protocol) provides a reliable transport layer by retransmitting missing packets and ordering them correctly, even though the underlying IP (Internet Protocol) layer may drop, delay, or duplicate packets.

While a higher-level system cannot be perfectly reliable—for instance, TCP cannot remove network delays—it can hide many tricky low-level faults, making the remaining errors easier to manage. The goal is to incorporate fault handling into the software design rather than hoping for hardware perfection.
