# KRONOS-RT │ Deterministic Time Executive

"In critical systems, timing is not a metric—it is correctness."

## ⚙️ System Overview

KRONOS-RT is a high-performance system daemon and real-time executive written in Nim. It is designed to act as the deterministic "heartbeat" for critical computing nodes, ranging from embedded microcontrollers to edge gateways.

Its primary function is to enforce strict temporal constraints. While standard Operating Systems rely on best-effort scheduling, KRONOS-RT implements userspace real-time logic to guarantee that critical sensor loops and actuator commands execute within defined windows, regardless of system load.

## 📐 Core Capabilities

The daemon operates with kernel-level priorities (utilizing SCHED_FIFO on Linux targets) to deliver three critical functions:

### 1. Process Supervision (The Watchdog)

Function: Monitors the health of high-level, non-deterministic processes (e.g., AI agents, complex simulations).

Mechanism: Implements a "Dead Man's Switch" protocol. If the client process fails to signal the watchdog within a defined window (e.g., 50ms), KRONOS initiates a hard reset of the service or triggers a hardware safe-mode state to prevent undefined behavior.

### 2. Isochronous Execution (The Scheduler)

Function: Enforces precise execution slots for hardware I/O.

Mechanism: Acts as an abstraction layer that decouples hardware timing from the garbage collection cycles of higher-level languages. It ensures sensors are sampled and servos are driven with sub-millisecond jitter variance.

### 3. Distributed Synchronization

Function: Global time alignment across networked nodes.

Mechanism: Implements a lightweight Precision Time Protocol (PTP) implementation, allowing distributed nodes to negotiate a unified "t=0" for coordinated physical actions.

## 🏗️ Architecture & Stack

#### Why Nim?

We selected Nim over C++ or Rust for this specific layer due to its unique compilation properties:

- Zero-Overhead Interop: Compiles directly to optimized C, allowing seamless integration with legacy hardware drivers.

- Soft Real-Time GC: The ARC/ORC memory management strategies provide deterministic latency behavior without the complexity of manual memory management.

- Efficiency: Delivers bare-metal performance suitable for constrained environments.

---

#### Module Structure

- kronos-core: The main event loop and priority scheduler.

- kronos-ipc: Shared memory and ZeroMQ interfaces for low-latency Inter-Process Communication.

- kronos-hal: Hardware Abstraction Layer for GPIO control and Watchdog timers (supports Linux and ESP32 targets).

---

## 📦 Usage & Integration

KRONOS-RT is designed to run as a systemd service or a bare-metal task.

#### 1. Defining a Critical Task

Register a hardware task that must execute strictly every 10ms.
~~~
import kronos_api

proc read_sensor_array() =
  # Critical hardware IO
  let val = hardware_read(0x40)
  ipc_send("DATA_PIPE", val)

# Register task: ID, Interval(ms), Tolerance(ms), Callback
kronos.register_task("sensor_loop", 10, 1, read_sensor_array)
kronos.start()
~~~

#### 2. Client Handshake (Python Example)

High-level processes must report to KRONOS to maintain uptime.
~~~
import socket

def main_process_cycle():
    # ... Complex processing logic ...
    
    # Signal the watchdog to reset the kill-timer
    sock.sendto(b"HEARTBEAT_OK", ("/var/run/kronos.sock"))
~~~

---

## 📡 Upstream & Downstream Integration

- Upstream: Reports system health, jitter metrics, and timing violations to telemetry dashboards. Receives mode-switch commands from system agents.

- Downstream: Outputs precise signals to motor drivers and actuators via direct GPIO/Serial, ensuring smooth kinematic interpolation.

## 🔒 Security Protocol

Process Isolation: Runs with CAP_SYS_NICE and CAP_IPC_LOCK capabilities to prevent preemption and memory swapping.

Fail-Safe Logic: In the event of a total system crash or timing violation, KRONOS defaults to a pre-configured "Hardware Lock" state (e.g., stopping motors, saving logs to non-volatile memory) rather than allowing uncontrolled operation.

---

## ⚖️ License

BSD-3-Clause
