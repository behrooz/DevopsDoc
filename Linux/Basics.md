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
