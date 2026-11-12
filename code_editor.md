# Multiplayer live code editor

Building a multiplayer live code editor from scratch in Go requires mastering network concurrency, distributed data synchronization, and Linux container security.
building this from the ground up without relying on third-party real-time services, learning roadmap breaks down into five core engineering topics in Golang.

1. **Advanced Concurrency & WebSockets**
Your Go server will act as the central nervous system, routing keystrokes and terminal outputs between users with zero lag.

* **Goroutines, Channels, and select**: Master Go's concurrency primitives to build a Hub-and-Spoke (Pub/Sub) pattern. Each coding session ("room") needs its own isolated hub that broadcasts messages to connected users without blocking other rooms.
* **Synchronization Primitives (sync package)**: Learn when to use sync.RWMutex for thread-safe in-memory maps (like tracking active users and rooms) and sync.Once or atomic counters for clean state management.
* **Binary WebSockets (coder/websocket)**: Go beyond basic JSON text messages. Learn to send and receive raw binary frames ([]byte), which is required for high-speed CRDT syncing and real-time terminal streaming.
* **Context Cancellation & Graceful Shutdowns**: Master the context package so that when a user closes their browser tab, all associated goroutines, file locks, and running compilers immediately terminate to prevent memory leaks.

2. **Conflict Resolution Algorithms (CRDTs or OT)**
To build the "from scratch" engine that merges simultaneous edits without overwriting anyone's code, you must understand distributed text algorithms.

* **CRDTs** (Conflict-free Replicated Data Types): Study sequence CRDTs like RGA (Replicated Growable Array) or YATA (the algorithm behind Yjs). Even if you use a library on the frontend, understanding how state vectors and binary updates work lets your Go server persist document snapshots efficiently.
* **Operational Transformation** (OT): Alternatively, study OT (used by Google Docs). In an OT model, your Go server acts as the central authority that receives character insert/delete operations, transforms their index positions against concurrent edits, and broadcasts the corrected order.
* **Binary Serialization**: Learn encoding/binary or Protocol Buffers (gRPC/Protobuf) in Go to pack cursor movements and text edits into tiny byte arrays instead of bulky JSON strings.

3. **Virtual File Systems & Storage Persistence**
Multiple users editing a project means managing a live folder structure (creating files, renaming folders, saving state) without corrupting the disk.

* **In-Memory File Systems (afero or io/fs)**: Learn Go's io/fs interface to hold active project files in RAM while users are typing, flushing changes to disk or an object store (like S3 or MinIO) only every few seconds (debouncing).
* **Embedded Databases (bbolt or BadgerDB)**: Master an embedded key-value store in Go to save append-only edit logs (CRDT update histories) locally at ultra-high speeds before archiving them.

4. **Process Execution, PTYs & Sandboxing**
Compiling and running untrusted code from 7 different languages (Go, Rust, Python, Node, Ruby, .NET, Java) is the hardest and most dangerous part of the backend.

* **Pseudo-Terminals (creack/pty)**: Standard os/exec commands do not support interactive terminal features like colors, progress bars, or user input (stdin). Master how to attach a Go WebSocket directly to a Linux PTY so users get a real, interactive terminal in their browser.
* **Docker SDK for Go (docker/client)**: Learn how to programmatically spin up containers, mount project folders, stream stdout/stderr pipes, and enforce hard timeouts using Go contexts.
* **Linux Namespaces & Cgroups (Optional Deep Dive)**: If you want to build your sandbox from scratch without Docker overhead, study Go's syscall package (SysProcAttr) to isolate processes using Linux namespaces, chroot, and cgroups for CPU/RAM limits.

5. **Reverse Proxying & Instant App Publishing**
To let developers "publish their app instantly" with a live URL, your Go server needs to route incoming web traffic to their running container.

* **Dynamic Reverse Proxy** (net/http/httputil): Master Go's built-in httputil.ReverseProxy. You will use this to inspect incoming subdomains (e.g., ://yourdomain.com), look up which local port or container IP belongs to that project, and forward the HTTP/WebSocket traffic seamlessly.
* **Dynamic TLS / SSL**: Learn how to use Go's golang.org/x/crypto/acme/autocert to automatically issue free Let's Encrypt SSL certificates on the fly when a user publishes a new app URL.

## Recommended Study Order

1. Week 1–2: Build a multi-room binary WebSocket chat server in Go using coder/websocket and goroutine hubs.
2. Week 3–4: Implement a basic text sync engine (either a simple OT server in Go or a Go relay for Yjs binary updates) and save snapshots to bbolt.
3. Week 5–6: Use creack/pty and the Go Docker SDK to execute a go run or cargo run command inside a container and stream the live terminal output to your frontend.
4. Week 7: Use httputil.ReverseProxy to expose the running container's web port to a custom subdomain.
