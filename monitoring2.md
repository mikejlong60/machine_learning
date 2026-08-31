### Monitoring System Proposal

## Requirements
- Monitoring should feed information to the UI shown in Chris's mockups
- Monitoring system should work with minimal configuration on:
    - Raw Kubernetes
    - Istio inside Kubernetes
    - Greymatter
    - Kumu
    - Bare metal EC2, Azure, OSX
    - Plain Envoy environments outside Kubernetes(docker, ...)
    - Plain docker environments
- Monitoring should be orthogonal to environment(i.e the monitored environment has no knowledge of its being monitored.)
- System should be able to monitor TCP applications, HTTP applications, and UDP applications
- For each of those three environments there will be a specialized collection mechanism
- There will also be a mechanism for parsing custom data structures like Swift.

## Design Considerations

# Cannot send all captured data to the an AI model. 

# You have to put the AI model at the end, on findings rather than data.

# Aggregate in the Linux kernel. The dominant cost is copying events to userspace.
    The use of BPF maps as per-CPU histograms and counters mean we export event summaries, not events from Kernel(eBPF) to user space.
    Reserve full event capture for sampled flows or when something is already known to be interesting. You want detail only when something is wrong,
    Put the AI model at the end, on findings rather than data:


# What to build first

    Build an eBPF program that does something small and prove that that same eBPF binary produces the same output when
    attached to AAC in docker compose and in Greymatter. That proves our orthogonality thesis and we should move forward with other parts.
    That something could be sched:sched_process_exec.  Run it and confirm you get the same cgroup ID for the same container whether it was 
    started by docker compose, or by kubelet-with-a-sidecar-injected in Greymatter. 

    The model can wait — it's the last stage, and it's the easiest to add once the data underneath is trustworthy.

# System Diagram
![](./mesh_agnostic_monitoring_system_architecture.png)

# What crosses each boundary

      boundary	        shape	                                              rate
      kernel → agent	ring buffer events + map reads	                      ~10k/s after in-kernel aggregation
      agent → cluster	serialised sketches, graph edges, local findings	  ~1/s per node
      cluster → agent	detail requests	on finding only
      cluster → model	findings + context	                                  ~10/min

# Node tier
      Kernel: The Kernel program is an eBPF program that runs on the Linux kernel. It is written 
      in restricted C.  It can never block.  It is triggered by an event, a function call, 
      or a packet, or some other things. It can't have unbounded loops and there are restictions on stack size
      of 512 bytes.  These restrictions are enforced by a program verifier that
      won't compile it unless it passes verification.  The end product is a "byte-code" file
      that is kernel-agnostic.  We don't have to recompile the eBPF program for different versions of
      of Linux.  It's like Java in this way.  There is a map of data called a BPF ring-buffer map 
      that programs outside the Kernel read.  For the purposes of this paper, that program is called an "Agent"

## Aggregation within the Node tier
      Aggregation occurs in both the Kernel and the Agent.  For example, the Kernel would aggregate 10 million requests
      into 64 different buckets.  Recall that in invocation of the Kernal program cannot use more then 512
      bytes of data.  But the same is not true of ring-buffer.  So for this example you would have 10 million
      invocations of the Kernel program. The BPF ring-buffer can hold megabytes of data.

      Agent - Rust:
         The Agent is a program that sits in user space and reads the BPF ring-buffer,
         This BPF ring buffer is an efficient way to transmit large amounts of data from eBPF(kernel) programs to 
         the user space where the Agent lives. The responsibilities of this agent are 

         aggregation - data obtianed from the ring-buffer maps
         enriches — cgroup ID or netns to pod, via a local k8s watch; SPIFFE ID from the peer cert
         sketches — A sketch is a small, fixed-size data structure that gives you an approximate answer to a question 
            about a large stream without storing the stream. t-digest is a way to measure latency given a sketch. Count-min 
            can estimate frequency given a sketch. A HyperLogLog measures cardinality given a sketch. 
            And you can merge these metrics from a node to the cluster level.

# Cluster tier

      Merge — combine per-node sketches into cluster-wide percentiles.
      Detection — anything needing a global view or a longer baseline. Seasonal comparisons, cross-service correlation, cardinality anomalies suggesting scanning.
      Different kinds of data stores — examples include a time-stamp-database for metrics, a findings store, and a service graph.
      Metadata — one k8s watch shared by everything, mapping workload identity. This is deliberately the only place that knows about Kubernetes, which keeps the 
      agent focused on kernel work.

# Analysis tier

      The Analysis tier consumes findings and summaries them.  It should not consume raw data.  Potential responsibilites include:  
      explaining a finding in English, ranking findings, removing duplicate findings.  It should also contain some sort of query 
      language against the various data stores. It does not consume many resources, a small language model at ten findings a minute needs 
      no GPU.
