# Linux Fundamentals

## Topics

1. Processes
2. Linux Namespaces
3. Control Groups (cgroups)
4. Filesystems
5. Networking Stack
6. systemd

---

# Processes

## What is a Process?

A process is a running instance of a program.

### Example

```bash
sleep 100
ps aux | grep sleep
```

Output:

```text
PID   COMMAND
1234  sleep
```

* **PID** = Process ID

Every process has:

* PID
* Parent PID (PPID)
* User
* Open files
* Memory allocations
* Network sockets

Processes are organized in a parent-child relationship.

### Example

```bash
sleep 1000
pstree
```

Or:

```bash
ps -ef --forest
```

---

## What Happens If the Parent Process Dies?

The child process becomes an **orphan process**.

The Linux kernel reassigns orphaned processes to **PID 1**, a process known as the **init process**.

---

## Why Is PID 1 Special?

PID 1 is the first userspace process started by the Linux kernel.

Its responsibilities include:

* Starting and managing system services
* Adopting orphaned processes
* Reaping zombie processes

On most modern Linux distributions, PID 1 is **systemd**.

---

## PID 1 Inside Containers

Example:

```bash
docker run ubuntu sleep 1000
```

Inside the container:

```text
PID  COMMAND
1    sleep
```

The `sleep` process becomes PID 1 inside the container.

---

## Why Do Some Containers Use `tini`?

PID 1 has special signal-handling behavior.

Without a proper init process:

* SIGTERM handling may fail
* Zombie processes may accumulate
* Child processes may not be cleaned up correctly

To solve this, many containers use:

* `tini`
* `dumb-init`

These lightweight init systems act as PID 1 and properly manage child processes.

---

# Zombie Processes

## What Is a Zombie Process?

A zombie process is a process that has finished execution but still has an entry in the process table because its parent process has not yet collected its exit status.

You can identify zombie processes with:

```bash
ps aux | grep Z
```

or

```bash
ps -el | grep Z
```

---

## Why Do Zombie Processes Exist?

When a process exits, the kernel keeps a small amount of information:

* PID
* Exit code
* Resource usage statistics

The parent process may need this information.

Therefore, the kernel cannot completely remove the process until the parent calls:

```c
wait()
```

or

```c
waitpid()
```

to collect the child's exit status.

---

## How Does PID 1 Prevent Zombie Processes?

Example:

```text
systemd (PID 1)
│
└── nginx (PID 500)
    │
    └── worker (PID 501)
```

If `nginx` dies unexpectedly:

```text
systemd (PID 1)
│
└── worker (PID 501)
```

The kernel reparents the orphaned worker process to PID 1.

This process is called **adoption**.

When the worker exits, PID 1 calls:

```c
waitpid(...)
```

and collects the exit status.

As a result, zombie processes do not accumulate.

---

## Why Is PID 1 Important Inside Containers?

Consider the following Dockerfile:

```dockerfile
CMD ["myapp"]
```

In this case:

```text
PID 1 = myapp
```

If `myapp` spawns child processes but never calls `wait()` or `waitpid()`, zombie processes may accumulate.

Example:

```text
myapp (PID 1)
│
├── child1
├── child2
├── child3
└── child4
```

If any child exits and `myapp` does not reap it, the child remains as a zombie.

This is why many containers use:

* `tini`
* `dumb-init`

Example:

```dockerfile
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["myapp"]
```

`tini` becomes PID 1, forwards signals correctly, and reaps zombie processes.

# SIGTERM

## What Is SIGTERM?

SIGTERM is a Unix signal used to request that a process terminate gracefully.

* Signal name: `SIGTERM`
* Signal number: `15`

You can send it using:

```bash
kill -TERM <PID>
```

or simply:

```bash
kill <PID>
```

By default, `kill` sends SIGTERM.

---

## What Happens When a Process Receives SIGTERM?

The kernel delivers the signal to the process.

The process can then:

* Handle the signal
* Ignore the signal (in some cases)
* Exit immediately

This gives the application an opportunity to:

* Close database connections
* Save its state
* Finish in-flight requests
* Clean up temporary files
* Release resources

before exiting.

---

## SIGTERM vs SIGKILL

### Graceful Termination

```bash
kill -15 1234
```

The process receives SIGTERM and has a chance to clean up before exiting.

### Forced Termination

```bash
kill -9 1234
```

The kernel sends SIGKILL and immediately terminates the process.

Characteristics of SIGKILL:

* Cannot be caught
* Cannot be ignored
* Cannot be handled
* No cleanup is possible

---

# How Docker Uses SIGTERM

When you run:

```bash
docker stop <container>
```

Docker performs the following steps:

1. Sends SIGTERM to PID 1 inside the container.
2. Waits for the grace period (10 seconds by default).
3. If the process is still running, sends SIGKILL.

---

## Scenario 1: Application Handles SIGTERM

```text
docker stop
     |
     v
  SIGTERM
     |
     v
Application performs cleanup
     |
     v
    Exit
```

Examples of cleanup:

* Closing database connections
* Finishing requests
* Flushing logs
* Saving state

---

## Scenario 2: Application Ignores SIGTERM

```text
docker stop
     |
     v
  SIGTERM
     |
     v
Application keeps running
     |
     v
10-second timeout expires
     |
     v
  SIGKILL
     |
     v
Process terminated immediately
```

---

# Linux Namespaces

This is where containers begin.

## What Makes a Container a Container?

Containers are primarily built from:

* Linux Namespaces
* cgroups (Control Groups)

Namespaces provide isolation.

cgroups provide resource control.

---

## What Is a Namespace?

A namespace isolates a set of system resources so that processes see only their own isolated view.

| Namespace | Isolates                          |
| --------- | --------------------------------- |
| PID       | Processes                         |
| NET       | Network interfaces, routes, ports |
| MNT       | Filesystem mount points           |
| IPC       | Shared memory and IPC objects     |
| UTS       | Hostname and domain name          |
| USER      | User and group IDs                |

---

## Why Doesn't a Pod See Processes From Other Pods?

Because each pod typically has its own PID namespace.

For example:

```text
Pod A
 ├── PID 1
 ├── PID 10
 └── PID 20

Pod B
 ├── PID 1
 ├── PID 5
 └── PID 15
```

Processes inside Pod A cannot see the processes running inside Pod B.

Each namespace provides its own isolated process tree.

Note: Kubernetes can be configured to share the PID namespace between containers in the same pod using:

```yaml
shareProcessNamespace: true
```

but this is not the default behavior.

## Network Namespaces

Every container gets its own isolated network stack. This includes:

- its own network interface
- its own routing table
- its own firewall rules (iptables)
- its own sockets

This is why two containers can both listen on port 80 without conflict — they are in different network namespaces and never see each other's ports.

---

## How a packet leaves a pod

Every pod has a virtual network interface called `eth0`. When a pod sends traffic outbound, the packet travels through a chain of virtual networking layers before reaching its destination.

```
Application
     │
     ▼
eth0 (virtual NIC inside the pod)
     │
     ▼
veth pair (virtual pipe — one end inside the pod, one end on the host)
     │
     ▼
Bridge / CNI plugin (e.g. Calico, Cilium — connects pods across the node)
     │
     ▼
Host networking (node's main network stack)
     │
     ▼
Destination
```

### Key components

| Component | What it does |
|---|---|
| `eth0` | The pod's own virtual NIC, created when the pod starts |
| veth pair | A virtual cable — packets written to one end appear on the other |
| Bridge / CNI | A virtual switch on the host that connects all pod veth ends |
| Host networking | The node's real network stack, where NAT and routing happen |

> The CNI (Container Network Interface) plugin — such as Calico, Cilium, or Flannel — is responsible for creating the veth pairs, configuring the bridge, and setting up routing rules when a pod is scheduled.

---

---

# NAT, IP Masquerading, and conntrack

## When a packet arrives at the host network

When a packet enters the host server's network namespace, Linux checks the destination:

- If the destination is a **pod on the same node**, the packet is forwarded directly to that pod via the virtual switch (Linux bridge or veth pair, depending on the CNI plugin).
- If the destination is **outside the node** (e.g. the internet), the packet goes to the routing table for forwarding.

---

## IP Masquerading (SNAT)

Pod IP addresses are internal (private) and not routable on the public internet. When a pod sends outbound traffic, the host performs **IP Masquerading** — a form of Source NAT (SNAT) — which replaces the pod's internal source IP with the node's own public IP.

This allows the remote destination to know where to send the reply.

> Example: Pod A at `10.0.0.5` sends a packet to Google. The node rewrites the source to `<node-IP>` before forwarding it.

---

## conntrack (Connection Tracking)

`conntrack` is the Linux kernel's connection tracking table. It acts as a stateful record of every active NAT translation.

When the host performs masquerading, it simultaneously writes a conntrack entry:

| Field | Value |
|---|---|
| Original source | `10.0.0.5` (Pod A) |
| Rewritten source | `<node-IP>` |
| Destination | `8.8.8.8` (Google) |

> Note: iptables (or nftables) rules are what *trigger* the masquerade on the first packet. conntrack then handles all subsequent packets in that flow automatically — no iptables lookup needed per-packet.

---

## The return path

1. Google sends a reply to the node's IP.
2. The kernel looks up the conntrack table.
3. It finds the matching entry: *"this reply belongs to Pod A."*
4. It performs reverse NAT (DNAT) — rewrites the destination from `<node-IP>` back to `10.0.0.5`.
5. The packet is forwarded to Pod A.

```
Pod A (10.0.0.5)
     │  outbound
     ▼
iptables (SNAT: src → node-IP)  ──writes──▶  conntrack table
     │
     ▼
Routing table ──────────────────────────────▶  Internet (Google)
                                                      │
                                               reply arrives
                                                      │
                                             conntrack lookup
                                                      │
                                          reverse DNAT (dst → 10.0.0.5)
                                                      │
                                                      ▼
                                               Pod A receives reply
```

---

## Summary

| Concept | Role |
|---|---|
| Network namespace | Isolates each container's network stack |
| eth0 | Virtual NIC inside the pod |
| veth pair | Virtual pipe connecting pod to host bridge |
| Bridge / CNI | Virtual switch connecting all pods on the node |
| Virtual switch | Forwards packets between pods on the same node |
| Routing table | Forwards packets destined outside the node |
| IP Masquerading | Replaces pod IP with node IP for outbound traffic |
| iptables | Applies the masquerade rule on the first packet |
| conntrack | Records the NAT mapping; handles reverse translation on return |