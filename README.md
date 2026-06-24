# Reliable Data Transfer over UDP — Error & Flow Control
**CS 371: Introduction to Computer Networking | University of Kentucky | Spring 2026**

> A two-part implementation of reliable data transfer (RDT) mechanisms over UDP in C — first observing packet loss under load with a pipelined protocol, then eliminating it with sequence numbers, ACKs, and Go-Back-N retransmission.

---

## Overview

UDP provides no delivery guarantees — no ordering, no retransmission, no flow control. This project builds those guarantees from scratch in two stages:

**Task 1** converts the PA1 TCP echo server to UDP and extends it to a pipelined protocol, deliberately saturating the server to observe and measure packet loss under high fan-in load.

**Task 2** implements a full Go-Back-N sliding window protocol over UDP — adding sequence numbers, per-packet ACKs, timeout-based retransmission, and a fixed-size wire format — achieving zero effective packet loss regardless of client thread count.

Both tasks run as a single binary supporting server and client modes, evaluated on NSF CloudLab bare-metal Ubuntu 22.04 x86 servers.

---

## Project Structure

```
├── task1.c      # UDP pipelined protocol with packet loss measurement
├── task2.c      # Go-Back-N over UDP: SN, ACK, retransmission, sliding window
└── README.md
```

---

## Architecture

### Task 1 — UDP Pipelined Protocol (Observe Loss)

```
Client Thread 1  ──┐
Client Thread 2  ──┤──► UDP Server (single socket, epoll, non-blocking)
Client Thread N  ──┘         │
                              └──► echo back (may drop under load)

Per thread metrics:
  tx_cnt   = packets sent
  rx_cnt   = packets received
  lost     = tx_cnt - rx_cnt  (timer-detected + tx-rx difference)
```

- Each client thread uses a **non-blocking UDP socket** and **epoll** for async I/O
- A sliding pipeline window (`PIPELINE_WINDOW = 32`) allows multiple in-flight packets
- Per-packet send timestamps enable **RTT measurement** and **timeout-based loss detection** (2s threshold)
- As `num_client_threads` increases, the server saturates and packet loss becomes observable

### Task 2 — Go-Back-N (Eliminate Loss)

```
┌──────────────────────────────────────────┐
│            packet_frame_t (32 bytes)     │
│  kind (4B) | seq_num (4B) | ack_num (4B) │
│  payload_len (4B) | payload (16B)        │
└──────────────────────────────────────────┘

Client sliding window:
  base ──────────────────────► next_seq
  [acked] [sent] [sent] [sent] [unsent...]
     └── timeout triggers retransmit from base

Server per-client state:
  expected_seq → in-order delivery enforcement
  last_acked_seq → cumulative ACK tracking
```

- Fixed 32-byte wire format with `kind` (DATA/ACK), `seq_num`, `ack_num`, `payload_len`
- Server maintains **per-client state** (keyed by `sockaddr_in`) for ordered ACK tracking
- Client retransmits all unACKed packets in the window on timeout (Go-Back-N semantics)
- Verified correct when `tx_cnt == rx_cnt` across all threads

---

## Build

Requires GCC and pthreads. Tested on Ubuntu 22.04 x86.

```bash
# Task 1
gcc -o pa2_task1 task1.c -pthread

# Task 2
gcc -o pa2_task2 task2.c -pthread
```

---

## Usage

Both binaries share the same argument pattern:

```bash
./<binary> <server|client> <server_ip> <server_port> <num_client_threads> <num_requests>
```

**Task 1 — observe packet loss:**
```bash
# Terminal 1: start server
./pa2_task1 server 127.0.0.1 9000 0 0

# Terminal 2: run client (increase threads to trigger loss)
./pa2_task1 client 127.0.0.1 9000 4 1000
./pa2_task1 client 127.0.0.1 9000 16 1000   # loss visible here
```

**Task 2 — zero packet loss:**
```bash
# Terminal 1: start server
./pa2_task2 server 127.0.0.1 9000 0 0

# Terminal 2: run client (loss eliminated regardless of thread count)
./pa2_task2 client 127.0.0.1 9000 8 5000
```

---

## Sample Output

**Task 1 (high load — loss visible):**
```
Average RTT (successful): 318 us
Aggregated Request Rate: 142857.14 messages/s
Total TX packets: 16000
Total RX packets: 14203
Total lost packets (timer-detected): 892
Total lost packets (TX-RX): 1797
```

**Task 2 (Go-Back-N — zero loss):**
```
Average RTT (successful): 401 us
Aggregated Request Rate: 119047.62 messages/s
Total TX packets: 40000
Total RX packets: 40000
Total lost packets: 0
tx_cnt == rx_cnt: PASS
```

---

## Key Concepts Demonstrated

| Concept | Task 1 | Task 2 |
|---|---|---|
| UDP sockets (`SOCK_DGRAM`) | ✓ | ✓ |
| Non-blocking I/O (`O_NONBLOCK`) | ✓ | ✓ |
| Linux epoll (async event handling) | ✓ | ✓ |
| Pipelined protocol (in-flight window) | ✓ | ✓ |
| Per-packet timer / loss detection | ✓ | ✓ |
| Sequence numbers | — | ✓ |
| Cumulative ACKs | — | ✓ |
| Go-Back-N retransmission | — | ✓ |
| Fixed wire format (`packet_frame_t`) | — | ✓ |
| Per-client server state tracking | — | ✓ |
| POSIX threads (`pthread`) | ✓ | ✓ |

---

## Group Members
- Katie Bell
- Ian Rowe
- Kaleb Gordon

---

## Environment
- OS: Ubuntu 22.04 (x86)
- Evaluated on: NSF CloudLab bare-metal servers
- Compiler: GCC with `-pthread`

---

## References
- [Linux epoll man page](https://man7.org/linux/man-pages/man7/epoll.7.html)
- Practical TCP/IP Sockets in C, 2nd Edition
