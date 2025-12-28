# System-Design-Notes
These revision notes are structured to provide a deep, technical dive into the networking concepts required for system design, following the provided text.

---

# 📘 Comprehensive Revision Notes: Networking for System Design

## 1. The Networking Foundation (The Layered Model)
Networking is built on the **OSI Model**, which uses layers as abstractions. This allows application developers to focus on data rather than physical voltages or hardware routing.

### The Three Essential Layers for System Design:
1.  **Network Layer (Layer 3):** Handles **IP (Internet Protocol)**. Responsible for routing, addressing, and breaking data into packets. It provides "best-effort" delivery.
2.  **Transport Layer (Layer 4):** Handles end-to-end communication (TCP, UDP, QUIC). Provides reliability, ordering, and flow control.
3.  **Application Layer (Layer 7):** The abstraction layer for web apps (HTTP, DNS, WebSockets, gRPC).

---

## 2. Anatomy of a Web Request
A simple request for `hellointerview.com` involves multiple coordinated steps:

1.  **DNS Resolution:** Human-readable domain $\rightarrow$ IP Address.
2.  **TCP Three-Way Handshake (Establishment):**
    *   **SYN:** Client requests connection.
    *   **SYN-ACK:** Server acknowledges and requests.
    *   **ACK:** Client confirms.
3.  **HTTP Request/Response:** The actual data exchange (GET/POST).
4.  **TCP Four-Way Handshake (Teardown):**
    *   **FIN $\rightarrow$ ACK** (Client side close).
    *   **FIN $\rightarrow$ ACK** (Server side close).

---

## 3. Transport Layer Protocols (L4)

### **TCP (Transmission Control Protocol)**
*   **Nature:** Connection-oriented "Stream."
*   **Guarantees:** Reliable delivery, data integrity, and strict ordering.
*   **Controls:** 
    *   *Flow Control:* Prevents overwhelming the receiver.
    *   *Congestion Control:* Adapts to network traffic to prevent collapse.
*   **Best For:** Most web applications, databases, and file transfers.

### **UDP (User Datagram Protocol)**
*   **Nature:** Connectionless "Machine gun" (Spray and pray).
*   **Pros:** Lower latency, no handshake overhead.
*   **Cons:** No delivery or ordering guarantees; packets can be lost or duplicated.
*   **Best For:** Live streaming, VoIP, Gaming, DNS lookups, and high-volume telemetry.

### **QUIC**
*   A modern protocol aiming to provide TCP’s reliability with better performance (used in HTTP/3). Mentioning this in interviews shows advanced knowledge.

---

## 4. Application Layer Protocols (L7)

### **HTTP/HTTPS**
*   **Stateless:** Each request is independent.
*   **HTTPS:** Adds SSL/TLS encryption. **Note:** Encrypts data in transit but does not validate request body content; servers must still perform validation.
*   **Methods:** 
    *   `GET`: Idempotent, no body.
    *   `POST`: Create resource.
    *   `PUT`/`PATCH`: Update resource.
    *   `DELETE`: Idempotent removal.
*   **Status Codes:** 2xx (Success), 3xx (Redirection), 4xx (Client Error - e.g., 429 Too Many Requests), 5xx (Server Error).

### **API Paradigms**
1.  **REST:** Simple, uses JSON, standard for most interviews. Maps Core Entities to resources.
2.  **GraphQL:** Solves **Over-fetching** (too much data) and **Under-fetching** (too many requests). Best for complex frontends needing flexible queries.
3.  **gRPC:** High-performance, uses **Protocol Buffers** (binary format) and HTTP/2. 
    *   *Pros:* Faster serialization (up to 10x), strong typing.
    *   *Use Case:* Internal microservice communication.

### **Real-Time Push Communication**
1.  **SSE (Server-Sent Events):** A "hack" on HTTP where the server streams chunks in one response. Unidirectional. Good for dashboards/notifications.
2.  **WebSockets:** Persistent, bidirectional TCP connection. Starts with an HTTP "Upgrade." Essential for chats and real-time gaming.
3.  **WebRTC:** Peer-to-Peer (P2P). Uses UDP. 
    *   Requires **Signaling Server** (to find peers) and **STUN/TURN** (to bypass firewalls/NAT).
    *   *Use Case:* Video/Audio conferencing.

---

## 5. Load Balancing (LB)

### **Client-Side LB**
*   **How:** Client queries a service registry (like Redis Cluster) or uses DNS rotation to pick a server.
*   **Pros:** Fast, no extra network hop.
*   **Cons:** Harder to update many clients quickly.

### **Dedicated (Server-Side) LB**
1.  **Layer 4 LB:** Routes based on IP/Port. Minimal packet inspection. **Best for WebSockets** as it maintains the TCP session.
2.  **Layer 7 LB:** "Smart" routing based on URL, cookies, or headers. More CPU intensive but offers more flexibility (e.g., API Gateway functionality).

### **Algorithms:**
*   **Round Robin:** Sequential.
*   **Least Connections:** Best for long-lived sessions (SSE/WebSockets).
*   **IP Hash:** Ensures a specific client always hits the same server (session persistence).

---

## 6. Latency, Regionalization & Resiliency

### **The Physics of Latency**
*   Light in fiber moves at ~200,000 km/s.
*   Round trip NYC $\rightarrow$ London: $\approx$ 56ms theoretical minimum.
*   **Solution:** **CDNs (Content Delivery Networks)**. Caches static assets at "Edge Locations" near the user.

### **Regional Partitioning**
*   Keep data close to the user.
*   *Example:* Uber partitions data by city (NYC vs. Miami) so queries are handled by local regional databases and servers.

### **Resiliency Patterns**
1.  **Retries + Exponential Backoff:** If a request fails, wait, then retry with increasing delays.
2.  **Jitter:** Add randomness to backoff to prevent the **"Thundering Herd"** (synchronized retries crashing a system).
3.  **Idempotency Keys:** A unique ID for a request to ensure it’s processed only once (critical for payments).
4.  **Circuit Breaker:** If a service fails repeatedly, stop calling it for a timeout period. Prevents **cascading failures** and allows the system to self-heal.

---

## 💡 Interview Tip: "Premature Optimization"
Don't jump to complex protocols (like gRPC or WebSockets) unless the requirements demand them. Default to **REST over HTTP** and **TCP** unless you can justify the trade-offs of performance vs. complexity.
