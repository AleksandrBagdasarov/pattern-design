# Leader-Follower

**Category:** Concurrency · **Also known as:** Leader/Followers (POSA2) · **Related:** Thread Pool, Producer-Consumer, Event Loop, Half-Sync/Half-Async, Leader Election

## Intent
A pool of threads takes turns: one thread, the leader, waits on a shared event source; when an event arrives it promotes a follower to be the next leader and then processes the event itself, so there is no hand-off queue between the thread that receives an event and the one that handles it.

Do not confuse this with **Leader Election** (one node in a cluster acting as coordinator) or with database leader/follower replication. Those are distributed-systems concepts; this is an in-process threading pattern.

## Problem
- In a classic "acceptor thread + queue + worker pool" server (Half-Sync/Half-Async), every event pays for a queue insertion, a lock, a context switch and a cache miss as it moves from the I/O thread to a worker.
- At very high event rates with short handlers (market data, telecom signaling, a custom protocol server), that hand-off cost dominates.
- Multiple threads blocking on the same socket set can cause a thundering herd: all wake, one wins.

Scenario: a low-latency quote server in C++ handles 500k small UDP messages/second; profiling shows most time spent in the queue between the receiver and workers.

## Solution and structure
```
            ┌─────────── followers wait on a lock/condition ───────────┐
            ▼                                                           │
 [leader] ──waits on event source (epoll/select/accept)──► event!       │
     │ 1. promote a follower to leader ──────────────────────────────────┘
     │ 2. handle the event itself (now a "processing" thread)
     └ 3. when done, rejoin the follower set
```
- **Handle set / event source**: sockets, a selector, a completion port.
- **Leader**: the only thread waiting on the source at a time (avoids thundering herd).
- **Followers**: threads queued for leadership.
- **Processing threads**: former leaders handling events concurrently.
- **Promotion**: a lock + condition; the leader releases leadership before processing.

Requirements: events must be safely handleable on whichever thread picked them up, and the handle must be deactivated while being processed so two threads do not read the same connection.

## When to use
- Rarely in application code. It is a server-runtime technique for very high event rates with short handlers, written in languages with real threads and direct control over the demultiplexer (C, C++, Java, Rust).
- When profiling shows the receiver→worker queue is the bottleneck.

## When NOT to use
- **Python.** The GIL (default build) and asyncio make this irrelevant for app code: use one asyncio loop per process and scale with processes (Gunicorn/Uvicorn workers). Even on free-threaded 3.13t/3.14t, use a framework rather than hand-rolling a demultiplexer.
- **Node / browser JS.** Single-threaded event loop per thread; there is no shared selector between threads. Scale with `cluster` / multiple processes or a load balancer.
- **Long or blocking handlers.** The pool runs out of followers and nobody waits on the source; Half-Sync/Half-Async with a queue and backpressure is safer.
- When ordering per connection matters and the framework does not deactivate handles: you get interleaved processing.
- Almost always: use the server framework (Netty, nginx, Envoy, Tokio) whose model is already tuned.

## Example — Python
Shown for understanding only: the idiomatic Python equivalent is "N processes, each with its own asyncio loop, sharing a listening socket", where the kernel picks which process accepts (pre-fork, as in Gunicorn). The sketch below is the literal pattern with threads and a shared selector.
```python
import selectors, socket, threading

class LeaderFollowerServer:
    def __init__(self, sock: socket.socket, threads: int = 4) -> None:
        self._sel = selectors.DefaultSelector()
        self._sel.register(sock, selectors.EVENT_READ)
        self._sock = sock
        self._leader_lock = threading.Lock()       # holder = current leader; waiters = followers
        self._threads = [threading.Thread(target=self._loop, daemon=True) for _ in range(threads)]

    def start(self) -> None:
        for t in self._threads:
            t.start()

    def _loop(self) -> None:
        while True:
            with self._leader_lock:                # become leader (followers block here)
                events = self._sel.select(timeout=1.0)
                conn = None
                if events:
                    try:
                        conn, _ = self._sock.accept()
                    except BlockingIOError:
                        pass
            # leaving the 'with' promotes the next follower to leader
            if conn is not None:
                self._handle(conn)                 # processing happens outside leadership

    def _handle(self, conn: socket.socket) -> None:
        with conn:
            conn.setblocking(True)
            request = conn.recv(4096)
            conn.sendall(b"HTTP/1.1 200 OK\r\nContent-Length: 2\r\n\r\nok" if request else b"")

if __name__ == "__main__":
    s = socket.create_server(("127.0.0.1", 8080))
    s.setblocking(False)
    LeaderFollowerServer(s).start()
    threading.Event().wait()
```
Under the GIL this gives concurrency for blocking I/O in `_handle` but no CPU parallelism. Prefer `uvicorn --workers N` in practice.

## Example — TypeScript
There is no shared demultiplexer across JS threads, so the pattern does not apply directly. The Node analog is several processes sharing one listening socket, letting the OS or the primary distribute connections:
```ts
import cluster from 'node:cluster';
import http from 'node:http';
import { availableParallelism } from 'node:os';

if (cluster.isPrimary) {
  // Default scheduling (except on Windows) is round-robin by the primary;
  // set cluster.schedulingPolicy = cluster.SCHED_NONE to let the OS choose
  // which worker accepts, which is closer to "whoever is waiting takes it".
  for (let i = 0; i < availableParallelism(); i++) cluster.fork();
  cluster.on('exit', (worker) => {
    console.warn(`worker ${worker.process.pid} died, restarting`);
    cluster.fork();
  });
} else {
  http
    .createServer((req, res) => {
      res.writeHead(200, { 'content-type': 'text/plain' });
      res.end(`handled by ${process.pid}\n`);
    })
    .listen(8080);
}
```
In containers, prefer one process per container and let the orchestrator/load balancer scale replicas instead of `cluster`.

## In the wild
- ACE / TAO (the CORBA ORB where POSA2 documented it); some Java NIO servers.
- Kernel support that serves the same goal: Linux `EPOLLEXCLUSIVE` (wake only one waiter) and `SO_REUSEPORT` (kernel-level load balancing across listeners).
- Pre-fork servers with a shared accept socket: Gunicorn sync workers, Apache prefork MPM, Node `cluster` with `SCHED_NONE`.
- Most modern runtimes (Netty, Tokio, nginx) instead use one event loop per thread with connections pinned to loops, which avoids both hand-off and leader promotion.

## Trade-offs
| Pros | Cons |
|---|---|
| No queue hand-off between receive and process | Complex to implement and debug |
| Good cache locality: the receiving thread processes | Hard to guarantee per-connection ordering |
| Avoids thundering herd on the event source | Long handlers starve the event source |
| Bounded thread count | Little or no benefit in Python and JS |

## Pitfalls
- **Processing while still leader**: blocks everyone. Promote before handling.
- **Not deactivating a handle** during processing: two threads read the same connection. Use one-shot registration (`EPOLLONESHOT`) and re-arm after handling.
- **Assuming it helps under the GIL.** It only overlaps blocking I/O; benchmark against asyncio.
- **Mixing up the name** with Leader Election or replication when discussing designs; say which you mean.

## Refactoring path
- **Introduce it:** only after a profile of a threaded, high-rate server shows the receiver→worker queue as the bottleneck, and after trying per-thread event loops and `SO_REUSEPORT`. Replace the acceptor + queue with a shared selector guarded by a leader lock; handle events after promotion.
- **Remove it / simplify:** move to a framework with per-thread event loops (Netty, Tokio) or, in Python/Node, to multiple single-threaded processes behind the OS or a load balancer.

## Related patterns
- [Thread Pool](../concurrency/thread-pool.md): leader-follower is a pool variant without a separate task queue.
- [Producer-Consumer](../concurrency/producer-consumer.md): the Half-Sync/Half-Async alternative with an explicit queue.
- [Event Loop](../emergent/event-loop.md): the single-threaded demultiplexer that Python and Node use instead.
- [Leader Election](../cloud/leader-election.md): unrelated distributed pattern with a similar name.
