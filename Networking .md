# Technical Architecture Reference Book

## Preface

This book systematically delves into critical aspects of network protocols, communication layers , interaction points for data units and their real-world nuance. Designed for technical peers, it aims to be a practical guide for understanding complex systems and addressing real-world challenges.

## Table of Contents

1.  **Networking Fundamentals**
    1.  **IP Addressing and Routing**
        1.  How a Static IP Can Serve Multiple Machines
        2.  CDN vs. Static IP Load Balancing: A Comparison
        3.  Single Domain with Multiple IPs: DNS-Based Load Balancing
        4.  Anycast Routing vs. DNS-Based Routing: A Detailed Comparison
    2.  **MAC Addressing and Layer 2**
        1.  The Role of MAC Addresses in Networking
        2.  Does Layer 2 (Data Link Layer) Always Exist?
        3.  MAC Address Usage in Home Networks (Jio Fiber + Single Device Scenario)
        4.  Why MAC Is Used Initially (Inside LAN) and Not in NAT
    3.  **NAT (Network Address Translation) and PAT (Port Address Translation)**
        1.  Understanding NAT and PAT
        2.  Why is PAT Needed if MAC is Already Sent?
        3.  Scenario: What Happens Without a Router?
        4.  How PAT Differentiates Processes Behind the Same IP
            * Internal NAT Table Mechanism
2.  **Network Protocols and Communication Flow**
    1.  **Internet Request Flow**
    2.  **OSI Model Reference: Layered View**
    3.  **DNS & HTTPS Flow Summary (Diagram)**
        * TLS 1.3 Handshake Key Details (L6 Focus)
    4.  **CDN (Content Delivery Network)**
        * Cloudflare vs. Akamai
        * CDN and DNS: Complementary Technologies
    5.  **Understanding Port Usage: Source vs. Destination Ports**
3.  **System Communication Protocols**
    1.  **Kafka Client Connecting to a Kafka Cluster**
        * Key Aspects of Kafka's Protocol
    2.  **Client Connecting to an RDBMS**
        * JDBC/ODBC for Relational Databases
        * Key Differences from Kafka Protocol
        * Protocols Used by Popular RDBMS
    3.  **Summary: Kafka Protocol vs. RDBMS Protocol**
4.  **High Availability and Load Balancing**
    1.  **Floating IPs: Use Cases Beyond Load Balancing**
        * High Availability and Failover Mechanisms
        * Database Replication and Failover
        * Migrating Services Without Downtime
    2.  **Ensuring High Availability and Failover for Load Balancers**
        * Implementing Redundant Load Balancers
        * Leveraging Global Server Load Balancing (GSLB)
        * Configuring DNS Failover
        * Implementing Floating IPs for Load Balancers
        * Real-World Examples (AWS, GCP, Azure)
    3.  **Classic Load Balancer (ELB)**
    4.  **Application Load Balancer (ALB)**
    5.  **Network Load Balancer (NLB)**
    6.  **Summary: ELB, ALB, NLB Comparison Table**

## 1. Networking Fundamentals

### 1.1. IP Addressing and Routing

#### 1.1.1. How a Static IP Can Serve Multiple Machines

Yes, it is somewhat similar to how a CDN (Content Delivery Network) works, but the mechanism is slightly different. Even if google.com (or any website) has a static IP, that IP is not necessarily tied to a single physical machine. Instead, it can be handled by multiple machines using the following mechanisms:

* **Anycast Routing (Like CDN but for Services)**
    * Google and many large-scale services use Anycast IP routing.
    * The same IP address is advertised from multiple locations (data centers).
    * The closest or least congested node responds to the request.
    * This is similar to how a CDN serves static content, but instead of just caching files, it routes entire traffic intelligently.
* **Load Balancers (L3/L4 or L7)**
    * A static public IP may belong to a load balancer rather than a single machine.
    * The load balancer distributes incoming traffic among multiple backend servers based on availability, load, or geolocation.
    * Example: Google Cloud Load Balancer (GCLB) distributes traffic globally.
* **DNS Load Balancing (Different from Static IP)**
    * Some services rely on DNS-based load balancing, where multiple IPs are mapped to a single domain.
    * This is how CDNs like Cloudflare or Akamai work: the DNS gives different IPs based on location.

#### 1.1.2. CDN vs. Static IP Load Balancing: A Comparison

While CDN and static IP-based services share the idea of multiple nodes serving requests, a CDN works at the DNS level with multiple IPs, whereas a static IP with Anycast or Load Balancing can distribute traffic while keeping a single IP address.

| Feature         | CDN                                  | Static IP with Anycast/Load Balancer |
| :-------------- | :----------------------------------- | :----------------------------------- |
| **Purpose** | Serve static content efficiently     | Route traffic to multiple backend servers |
| **IP Address** | Can change dynamically               | Usually remains static               |
| **Routing** | DNS-based (different IPs per region) | Anycast or Load Balancer (same IP)   |
| **Use Case** | Caching for performance              | Ensuring availability of services    |

#### 1.1.3. Single Domain with Multiple IPs: DNS-Based Load Balancing

When "multiple IPs are tagged to a single domain," it means that a domain name (like google.com) can resolve to different IP addresses depending on various factors. This is different from a static IP setup. A domain name (like google.com) is mapped to one or more IP addresses using DNS (Domain Name System).

* **DNS-Based Load Balancing**
    * When you type google.com, your system queries a DNS server to resolve the domain to an IP address.
    * Instead of returning a single static IP, DNS can return different IPs based on factors like:
        * **Geolocation:** Users in India might get a different IP than users in the U.S.
        * **Load Distribution:** To balance traffic across data centers.
        * **Health Checks:** If one server is down, DNS can return a different IP.
    * Example using `nslookup` or `dig` (Try this on your terminal):
        ```bash
        nslookup google.com
        ```
        Example Output:
        ```
        Name: google.com
        Addresses: 142.250.180.14
                   142.250.180.206
                   142.250.180.238
        ```
        Notice how google.com resolves to multiple IPs? Your browser picks one.
* **Round-Robin DNS**
    * DNS servers rotate IP addresses for each request to spread traffic.
    * Example:
        * Request 1 → google.com → IP 142.250.180.14
        * Request 2 → google.com → IP 142.250.180.206
        * Request 3 → google.com → IP 142.250.180.238
    * This is different from a single static IP that is always the same.
* **CDN and Edge Servers**
    * CDNs like Cloudflare or Akamai work at the DNS level.
    * They return an IP of a nearby edge server instead of a central server.
    * This allows fast content delivery without a fixed/static IP.

#### 1.1.4. Anycast Routing vs. DNS-Based Routing: A Detailed Comparison

Anycast and DNS-based routing do not work the same way, although they both help in directing traffic to the nearest or best-performing server.

#### DNS-Based Routing (DNS Load Balancing)

✅ **How It Works:**

* When a user requests a domain (google.com), their system queries a DNS server to resolve the IP.
* Instead of returning a single IP, the DNS resolver can return different IPs based on factors like:
    * **Geolocation:** Users in India may get an Indian server's IP, while U.S. users get a U.S. server's IP.
    * **Load balancing:** Traffic is distributed across multiple data centers.
    * **Health checks:** If a server is down, DNS returns a different IP.
* The browser chooses an IP from the list and connects to it.

✅ **Example:**
`nslookup google.com`
Output might show:
```
Name: google.com
Addresses: 142.250.180.14
142.250.180.206
142.250.180.238
```
Each user might get different results based on location.

✅ **Use Cases:**

* CDNs (Cloudflare, Akamai) to route users to the closest edge server.
* Multi-region cloud applications (AWS Route 53, Google Cloud DNS).
* Websites that want to distribute traffic across multiple data centers.

⚠️  **Limitations:**

* DNS caching means users may still connect to an outdated server if an IP is removed.
* DNS resolution happens before a request is made, meaning it can't react dynamically to real-time congestion.
* The client (browser) picks an IP and connects to it, which may not always be optimal.

#### Anycast Routing (Network-Level Load Balancing)

✅ **How It Works:**

* A single IP address is advertised from multiple locations via BGP (Border Gateway Protocol).
* When a user sends a request to that IP, the network automatically routes it to the nearest or least congested server.
* Unlike DNS, this happens at the network layer (L3 - IP routing) and can dynamically adjust based on real-time traffic.

✅ **Example:**

* Google Public DNS (8.8.8.8)
    * This IP is the same worldwide but routes requests to different locations based on network proximity.
* Cloudflare's Anycast CDN
    * Even though 1.1.1.1 (Cloudflare's DNS resolver) is a single IP, it is served from multiple global locations.

✅ **Use Cases:**

* Global services with a single static IP (e.g., 8.8.8.8 for Google DNS).
* CDNs with Anycast IPs (Cloudflare, Fastly).
* DDoS mitigation – traffic is absorbed by multiple data centers.

⚠️  **Limitations:**

* Expensive & complex to set up (requires BGP control).
* Works best for stateless services (e.g., DNS, HTTP caching) rather than database-heavy applications.

##### Key Differences Summary
| **Feature**          | **Anycast Routing**                                                                                    | **DNS-Based Routing**                                                           |
| -------------------- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------- |
| **How it Works**     | Routes traffic to the *nearest node* at the network level using **BGP**                                | Resolves domain to different IPs based on **DNS policies** (geo, latency, etc.) |
| **IP Address**       | 🟢 Same IP globally (e.g., `8.8.8.8`)                                                                  | 🔵 Different IPs returned based on client location                              |
| **Decision Made By** | 🛰️ Network routers (BGP layer)                                                                        | 🌍 DNS resolvers applying logic                                                 |
| **Adaptability**     | ⚡ Real-time routing based on current **network congestion**<br>✅ Adjusts instantly if a node goes down | 🕒 Static routing until TTL expires<br>⚠️ Cannot adapt mid-session              |
| **Caching Issues**   | ✅ No DNS caching issues – users always hit the **nearest** and optimal node                            | ⚠️ DNS responses cached – can lead to **stale or suboptimal IPs**               |
| **Use Cases**        | 🌐 Global APIs, DNS services, DDoS mitigation                                                          | 🖥️ Websites, CDNs, multi-region applications                                   |
| **Examples**         | `8.8.8.8` (Google DNS), `1.1.1.1` (Cloudflare)                                                         | `google.com` resolving to multiple regional IPs                                 |

### 1.2. MAC Addressing and Layer 2

#### 1.2.1. The Role of MAC Addresses in Networking

| Concept         | Purpose                                                      |
| :-------------- | :----------------------------------------------------------- |
| **MAC Address** | Works at Layer 2 (Data Link). Used for identifying devices within a LAN (e.g., home network). |
| **IP Address** | Works at Layer 3 (Network). Used for identifying devices across different networks (e.g., the internet). |
| **NAT (Network Address Translation)** | Rewrites private IPs (192.168.x.x) to a single public IP (your router's IP). |
| **PAT (Port Address Translation)** | Helps multiple devices share one public IP by tracking requests using port numbers. |

#### 1.2.2. Does Layer 2 (Data Link Layer) Always Exist?

Yes, Layer 2 always exists in any network communication, but MAC addresses may not always be used explicitly depending on the underlying technology. If you don’t have a traditional LAN (Local Area Network) with Ethernet or Wi-Fi, the network still requires a Layer 2 mechanism to transfer data. The key difference is what replaces MAC addresses.

* **Case 1: Wired/Wireless LAN (Ethernet, Wi-Fi)**
    * Uses MAC addresses for device-to-device communication.
    * Every frame has a source MAC and destination MAC.
* **Case 2: Cellular Networks (4G/5G)**
    * No traditional MAC address at the user device level.
    * Uses other Layer 2 identifiers like:
        * LTE GUTI (Globally Unique Temporary Identifier)
        * 5G RNTI (Radio Network Temporary Identifier)
    * The mobile network still has a Data Link layer, but instead of MAC addresses, it uses these IDs.
* **Case 3: Satellite Communication**
    * Uses proprietary Layer 2 protocols instead of Ethernet.
    * MAC-like identifiers exist within satellite protocols.

**What If There’s No MAC Address?**
If a network doesn’t use MAC addresses, it still has a Layer 2 mechanism for:
* Framing (Encapsulating data in a structure that can be transmitted).
* Error detection (Checksums, CRC).
* Media access control (Who gets to send data).

For example:
* A fiber-optic backbone uses SONET/SDH (instead of MAC).
* A cellular network uses radio link identifiers.

**Conclusion: Layer 2 Always Exists**
 Layer 2 is fundamental to networking—it ensures devices can communicate over a physical medium. MAC addresses are only one form of Layer 2 addressing, but other network types (cellular, MPLS, satellite) have different Layer 2 identifiers.

#### 1.2.3. MAC Address Usage in Home Networks (Jio Fiber + Single Device Scenario)

Yes, MAC addresses will still be used but only within your local network (your device ↔ router).

* **Inside Your Home (Local Network)**

    1.  Your machine (e.g., PC) sends a request to the router (Jio Fiber ONT/Router).
        * Source MAC: Your PC’s MAC address.
        * Destination MAC: Jio Router’s MAC address.
        * Used? ✅ Yes, because Ethernet/Wi-Fi relies on MAC addresses.
    2.  Router processes the packet and performs NAT (Network Address Translation) to replace your private IP with its public IP.
        * The MAC address in this request only exists inside your LAN.
        * Once it leaves the router, the MAC is replaced.

* **Outside Your Home (Jio Network & Internet)**

    1.  Your router sends the request to Jio’s Fiber Network (ISP Infrastructure).
        * MAC is no longer relevant at this stage because ISPs route based on IPs, not MACs.
        * The router communicates with Jio's infrastructure using PPP, VLAN, or other L2 protocols.
    2.  The data travels over the internet (Layer 3 routing), and MAC addresses are not required beyond each hop.

**Conclusion**  
MAC addresses are used only within your home network (PC ↔ Router).  
After your router forwards the request to Jio's network, MAC is no longer relevant.  
The internet routes data based on IP, not MAC.



#### 1.2.4. Why MAC Is Used Initially (Inside LAN) and Not in NAT

1.  MAC (Layer 2) operates only within a single local network.
    * It is needed to route frames within your local network (PC ↔ Router).
2.  IP (Layer 3) is used across networks, including the internet.
    * Once the data leaves your router, it is routed using IP addresses, not MAC addresses.
3.  NAT is used to allow multiple devices in your LAN to share a single public IP, and the router uses IP and port mapping to differentiate them.

### 1.3. NAT (Network Address Translation) and PAT (Port Address Translation)

#### 1.3.1. Understanding NAT and PAT

Behind a router using PAT (Port Address Translation), multiple devices (like PC1, PC2) share one public IP. The router maps:

`Private IP + Private Port → Public IP + Public Port`

Example:

* `PC1: 192.168.1.2:1234 → 203.0.113.5:50001`
* `PC2: 192.168.1.3:2345 → 203.0.113.5:50002`

#### 1.3.2. Why is PAT Needed if MAC is Already Sent?

1.  **MAC Addresses Only Work in the Local Network**
    * When your PC sends data, it only includes MAC addresses of:
        * Source: Your PC (MAC\_A)
        * Destination: Your Router (MAC\_R)
    * Once the packet reaches the router, the MAC address changes!
    * The router replaces the source MAC with its own and forwards the request.
2.  **MAC Addresses Don’t Work Across the Internet**
    * Routers don’t route based on MAC addresses; they only use IP addresses.
    * Your ISP doesn’t know your PC’s MAC address, only your public IP.
3.  **PAT Solves the "Who Sent This?" Problem**
    * Since multiple devices share the same public IP, PAT tracks outgoing requests using port numbers.
    * Example:
        * `PC1 (192.168.1.2) → NAT: 203.0.113.5:50001`
        * `PC2 (192.168.1.3) → NAT: 203.0.113.5:50002`
    * When the response arrives, the router looks at the port number and sends it back to the correct device.

**Final Takeaways**
✅ MAC Addresses are used only inside your LAN (switches, local communication).
✅ IP Addresses are used to route traffic across the internet.
✅ NAT changes private IPs to a public IP (many devices → one IP).
✅ PAT allows multiple devices to use one public IP by tracking port numbers.

#### 1.3.3. Scenario: What Happens Without a Router?

If all devices are on the same network (LAN):

* MAC addresses are used for everything because Layer 2 (Ethernet) handles communication.
* **No NAT or PAT is needed**.

#### 1.3.4. How PAT Differentiates Processes Behind the Same IP

What if two processes on the same private IP (e.g., 192.168.1.2) talk to the internet from different ports?

✅ **How PAT Differentiates Processes Behind the Same IP:**
Let’s say:

* Process A on 192.168.1.2 uses port 1234
* Process B on 192.168.1.2 uses port 1235

When both connect to the internet, the router maps them to different public ports, like:

* `192.168.1.2:1234 → 203.0.113.5:50001`
* `192.168.1.2:1235 → 203.0.113.5:50002`

So even though both processes come from the same IP, they have different source ports, and that’s enough for the router to uniquely track them.

**Internally: The router keeps a NAT table like:**

| Private IP  | Private Port | Public Port | Destination IP/Port   |
| :---------- | :----------- | :---------- | :-------------------- |
| 192.168.1.2 | 1234         | 50001       | 142.250.195.100:443   |
| 192.168.1.2 | 1235         | 50002       | 151.101.65.69:80      |

On response, the router matches the public port (e.g., 50001) to the original sender (192.168.1.2:1234), and forwards it to that exact process.

**Summary:**

* Processes on the same device use different source ports.
* PAT maps (IP + Port) → (Public Port).
* This mapping ensures the router knows which process to forward the response to.
* The router doesn’t need to know the process ID (PID), just the source port, which is unique per connection per device.

```
+---------------------+           +------------------------+           +------------------------+
|   PC (192.168.1.2)  |           |     Home Router        |           |    Internet Server     |
|---------------------|           |  (NAT + PAT Enabled)   |           |  (e.g., Web/Google)     |
|                     |           |  Public IP: 203.0.113.5|           |                        |
| Process A           | -- Src: 192.168.1.2:1234 -->       | -- Src: 203.0.113.5:50001 --> | google.com:443 |
|                     |           |  (PAT maps to 50001)   |           |                        |
|---------------------|           |------------------------|           |------------------------|
|                     |           |                        |           |                        |
| Process B           | -- Src: 192.168.1.2:1235 -->       | -- Src: 203.0.113.5:50002 --> | example.com:80  |
|                     |           |  (PAT maps to 50002)   |           |                        |
+---------------------+           +------------------------+           +------------------------+
```

## 2. Network Protocols and Communication Flow

### 2.1. Internet Request Flow
```
┌──────────────┐            ┌────────────────┐            ┌───────────────┐            ┌──────────────────┐
│   User PC    │ ─────────> │  Home Router   │ ─────────> │   ISP Router  │ ─────────> │   Destination    │
│ 192.168.1.2  │            │ 192.168.1.1    │            │  (Public IP)  │            │  (e.g., google)  │
└─────┬────────┘            └──────┬─────────┘            └─────-─┬──────-┘            └────────┬─────────┘
      │                            │                              │                             │
      │ 1. DNS Request             │                              │                             │
      ├───────────────────────────>│                              │                             │
      │ (Who is google.com?)       │                              │                             │
      │                            │ 2. DNS Lookup                │                             │
      │                            ├─────────────────────────────>│                             │
      │                            │ Gets Public IP               │                             │
      │                            │ (e.g., 142.250.x.x)          │                             │
      ◄──────────────────────────-─┘                              │                             │
      │                                                           │                             │
      │ 3. HTTP Request            │                              │                             │
      ├───────────────────────────>│                              │                             │
      │ (Request to google.com)    │                              │                             │
      │ (src = 192.168.1.2)        │                              │                             │
      │                            │ 4. NAT Translation           │                             │
      │                            ├─────────────────────────────>│                             │
      │                            │ Src IP → Router's Public IP  │                             │
      │                            │                              │ 5. Request Forwarding       │
      │                            │                              ├────────────────────────────>│
      │                            │                              │ Sends to google.com         │
      │                            │                              │                             │
──────┴────────────────────────────┴──────────────────────────────┴─────────────────────────────┴────────────────────
```

### 2.2. OSI Model Reference: Layered View

The OSI model is a conceptual framework that standardizes how diverse hardware and software components in a network communication system divide responsibility and interact. It provides a layered approach where each layer performs specific functions and communicates with its adjacent layers through well-defined interfaces.

Within this model, each layer exchanges data using a specific format called the Protocol Data Unit (PDU). The PDU varies per layer — for instance, a segment at the Transport layer or a frame at the Data Link layer.

When a user initiates an action, such as browsing a website, the application layer (Layer 7) on the client generates the request. The data then flows downward through each OSI layer, where headers (and sometimes footers) are added by each layer in a process called encapsulation. These headers guide the data through the network, enabling routing, delivery, error handling, and security.

Once the data reaches the physical layer, it's transmitted over the medium to the destination. There, the reverse happens: each layer de-encapsulates its respective header, processing the information and passing the remaining payload upward until it reaches the destination application.

This layered encapsulation/de-encapsulation workflow ensures modularity, interoperability, and debugging ease across complex distributed systems.
```
┌──────────────┬──────────────┬───────────────────────────────────────-─────┬────────────────────────────────────────────────┬──────────────────────────────────────────────┐
│ OSI Layer    │ Data Unit    │ Encapsulates (Adds What)                    │ Purpose / Enables                              │ Real-world Analogy                          │
├──────────────┼──────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ 7. App       │ Data(text)   │ App content, protocol headers               │ User's intent (browse, email, stream)           │ Gmail, Netflix, Google Chrome               │
│ 6. Pres      │ Data         │ Format (JSON/XML), encryption, compression  │ Interoperability, security, compression         │ Translator, ZIP file, TLS/SSL               │
│ 5. Sess      │ Data         │ Session IDs, handshake tokens               │ Start/manage sessions (stateful communication)  │ WebSocket control, phone call setup         │
│ 4. Trans     │ Segment      │ TCP/UDP: Ports, Seq/Ack, flags              │ Reliable delivery, ordering, flow control       │ Numbered parcel via courier                 │
│ 3. Net       │ Packet       │ IP header: Src/Dst IP, TTL, protocol ID     │ Routing across networks (logical addressing)    │ Postal street address                       │
│ 2. Data Link │ Frame        │ MAC header, FCS, VLAN tags                  │ Local delivery, error detection                 │ Building/apartment + delivery receipt       │
│ 1. Physical  │ Bits         │ Binary signals over medium                  │ Physical movement (electric/light/radio waves)  │ Light/electric pulses on cable/wire         │
└──────────────┴──────────────┴─────────────────────────────────────────────┴────────────────────────────────────────────────┴──────────────────────────────────────────────┘
```
#### 2.2.1 Information flow across Layers


```
╭────────────────────────────────────────────╮           ╭──────────────────────────────────────────╮
│          Transmitting Device               │           │          Receiving Device                │
│          (Client Application)              │           │         (Server Application)             │
╰────────────┬───────────────--------────────╯           ╰──────────────────┬───────────────────────╯
             │                                                                     │
      ┌──────▼──────┐                                                ┌──────▲──────┐
      │ Application │       (Layer 7 - HTTPS)                        │ Application │
      └─────────────┘                                                └─────────────┘
             |                                                              ▲
      ┌──────▼──────┐                                                ┌──────▲──────┐
      │ Presentation│       (Layer 6 - TLS handshake + encryption)   │ Presentation│
      └─────────────┘                                                └─────────────┘
             |                                                              ▲
      ┌──────▼──────┐                                                ┌──────▲──────┐
      │ Session     │       (Layer 5 - implicit)                     │ Session     │
      └─────────────┘                                                └─────────────┘
             ▼                                                              ▲
┌──────────────────────────────────────────────┐           ┌────────────────────────────────────────────┐
│ Encrypted Payload (TLS Record Layer)         │ ────────▶ │ Decrypted Payload (by TLS Record Layer)    │
│ TLS record → "GET /index.html...Host:..."    │           │ "GET /index.html HTTP/1.1\r\nHost:..."     │
└──────────────────────────────────────────────┘           └────────────────────────────────────────────┘
             ▼                                                              ▲
┌──────────────────────────────┐                              ┌──────────────────────────────┐
│ TCP Header                   │       (Layer 4 - TCP)        │ TCP Header                   │
│   Src Port: 54321            │                              │   Dst Port: 54321            │
│   Dst Port: 443              │                              │   Src Port: 443              │
│   Flags: SYN                 │                              │   Flags: SYN-ACK             │
│   Seq: 12345678              │                              │   Ack: 12345679              │
└──────────────────────────────┘                              └──────────────────────────────┘
             ▼                                                              ▲
┌──────────────────────────────┐                              ┌──────────────────────────────┐
│ IP Header                    │       (Layer 3 - IP)         │ IP Header                    │
│   Src IP: 192.168.1.10       │                              │   Dst IP: 192.168.1.10       │
│   Dst IP: 142.250.183.110    │                              │   Src IP: 142.250.183.110    │
│   Protocol: 6 (TCP)          │                              │   Protocol: 6 (TCP)          │
└──────────────────────────────┘                              └──────────────────────────────┘
            ▼                                                               ▲
┌──────────────────────────────┐                              ┌──────────────────────────────┐
│ MAC Header                   │       (Layer 2a - MAC)       │ MAC Header                   │
│   Src MAC: 11:22:33:44:55:66 │                              │   Dst MAC: 11:22:33:44:55:66 │
│   Dst MAC: AA:BB:CC:DD:EE:FF │                              │   Src MAC: AA:BB:CC:DD:EE:FF │
│   EtherType: 0x0800 (IPv4)   │                              │   EtherType: 0x0800 (IPv4)   │
└──────────────────────────────┘                              └──────────────────────────────┘
            ▼                                                               ▲
┌──────────────────────────────┐                              ┌──────────────────────────────┐
│ FCS (Frame Check Sequence)   │       (Layer 2b - CRC)       │ FCS (Frame Check Sequence)   │
│   CRC: 0xABCD1234            │                              │   CRC: 0xABCD1234            │
└──────────────────────────────┘                              └──────────────────────────────┘
            ▼                                                               ▲
┌──────────────────────────────┐                              ┌──────────────────────────────┐
│ 010110101011101... (Bits)    │       (Layer 1 - Physical)   │ 010110101011101... (Bits)    │
└──────────────────────────────┘                              └────--------------------------┘
```

| Layer | Name             | Function                                      | Analogy (Mail System)                 | Key Protocols/Concepts               |
| :---- | :--------------- | :-------------------------------------------- | :------------------------------------ | :----------------------------------- |
| **7** | **Application** | User interaction, network services            | Writing the letter's content          | HTTP(S), DNS, FTP, SMTP, SSH         |
| **6** | **Presentation** | Data formatting, encryption/decryption        | Translating language, encrypting text | TLS/SSL, JPEG, MPEG, ASCII           |
| **5** | **Session** | Manages communication sessions                | Putting the letter in an envelope     | NetBIOS, Sockets (in some contexts)  |
| **4** | **Transport** | Reliable data transfer, flow control          | Postal service ensuring delivery      | TCP (reliable), UDP (fast, unreliable) |
| **3** | **Network** | Logical addressing, routing across networks   | Postal service routing parcels        | IP, ICMP, Routing protocols (OSPF, BGP) |
| **2** | **Data Link** | Physical addressing, error detection in LAN   | Mail carrier delivering to your street | MAC, ARP, Ethernet, Wi-Fi            |
| **1** | **Physical** | Physical transmission of bits                 | The actual road the mail truck drives | Cables, Fiber optics, Radio waves    |

### 2.3. DNS & HTTPS Flow Summary (Diagram)
```
| Client           DNS Resolver       NAT               CDN(edge nod)     Load Balancer           Backend
|    |                  |              |                 |                      |                    |
| 1. Enters URL         |              |                 |                      |                    |
| --------------------->|              |                 |                      |                    |
| 2. DNS Query          |              |                 |                      |                    |
| --------------------->|              |                 |                      |                    |
| 3. Receives IP Addr   |              |                 |                      |                    |
| <---------------------|              |                 |                      |                    |
|                       |              |                 |                      |                    |
| 4. Connect via Public IP            ------------------->                      |                    |
|                           5. NAT Translates IP        ------------------->                         |
| 6. TLS Handshake          ----------------------------->                                           |
|                       |           7. HTTP req inside TLS ---->                                           |
|                       |              |         8. TLS  (Optional CDN ↔ LB) -->                     |
|                       |              |                 |                      |   9. TLS (optional)|
|                       |              |                 |                      | -----------------> |
| 10. HTTPS Comm Established          --------------------------------------->                       |
| 11. Serve Static (if cached)        <-------------------- (if available)                           |
| 12. Else Route to Backend                              ----------------------------->              |
| 13. Backend Processes Request                                                                      |
| <--------------------------------------------------------------------------------------------------|
| 14. Response Returned               <----------------------------                                  |
| 15. Client Receives Response <---------------------------------------------------------------—-----|

```
### Simplified Flow Summary

- Client initiates the request by entering the URL.  
- DNS Resolver resolves the domain name into an IP address.  
- NAT translates the client’s private IP to a public IP.  
- The CDN handles the incoming HTTPS request and initiates a TLS handshake.  
- If content is cached, the CDN returns it; otherwise, the request is forwarded.  
- The Load Balancer routes the request to a backend server.  
- The Backend processes the dynamic request and sends a response.  
- The response flows back through the Load Balancer, CDN, NAT, and to the client.

**HTTPS Request Flow - Timeline Style**

Here's how the flow can be represented in a timeline format. The timeline showcases sequential steps in HTTP request processing, including DNS resolution, connection setup, TLS handshake, and response phases.

| Time (ms) | Action                                            | Description                                          |
| :-------- | :------------------------------------------------ | :--------------------------------------------------- |
| 0ms       | Client Initiates Request                          | User enters URL (e.g., example.com) into the browser |
| 1-20ms    | DNS Resolution Starts                             | Browser queries Local DNS Resolver                   |
| 20-100ms  | DNS Query to Root DNS Server                      | Root points to TLD server                            |
| 100-150ms | DNS Query to TLD DNS Server (.com)                | TLD directs to Authoritative DNS Server              |
| 150-200ms | DNS Query to Authoritative DNS Server             | Authoritative DNS returns IP (e.g., 192.0.2.1)       |
| 200-210ms | DNS Resolution Complete                           | Local Resolver caches the IP                         |
|           |                                                   |                                                      |
| 210-220ms | TCP Connection Starts (Client → CDN)              | Client initiates 3-way handshake with CDN            |
| 220-230ms | TCP Connection Established                        | TCP handshake complete                               |
|           |                                                   |                                                      |
| 230-300ms | TLS Handshake Starts                              | Client and CDN perform SSL/TLS negotiation           |
| 300-350ms | Secret Key Derived                                | Encrypted channel established for HTTPS              |
|           |                                                   |                                                      |
| 350ms     | HTTP Request Sent (Client → CDN)                  | Client sends HTTPS request for content               |
| 350-360ms | CDN Processes Request                             | CDN determines content type (static/dynamic)         |
| 360-370ms | CDN Returns Static Content (optional)             | Static files served directly (CSS, JS, images)       |
| 370-450ms | Dynamic Content Forwarded (CDN → Backend)         | CDN forwards request through Firewall and LB         |
| 450-500ms | Load Balancer Routes Request                      | LB directs request to appropriate microservice       |
| 500-600ms | Backend Processes Request (Document Service)      | Processes logic, fetches data, etc.                  |
| 600-650ms | Response Sent to Load Balancer                    | Backend returns data                                 |
| 650-700ms | Response Sent to CDN                              | LB → Firewall → CDN                                  |
| 700-720ms | CDN Delivers Response to Client                   | Encrypted response sent over established TLS         |
|           |                                                   |                                                      |
| 720ms+    | Client Receives and Renders Content               | Browser displays the webpage or resources            |


**TLS 1.3 Handshake Key Details (L6 Focus)**

* **ClientHello:**
    * List of supported cipher suites
    * TLS version
    * Key share (e.g., g^x mod p)
* **ServerHello:**
    * Selected cipher suite
    * Server's public key
    * ALPN (Application-Layer Protocol Negotiation)
* **Shared Secret Key Derived:**
    * Both sides derive same symmetric key using key exchange (e.g., ECDHE)
* **After handshake:**
    * All communication is encrypted
    * Application layer (L7) sends secure HTTP (HTTPS) requests
```
Browser                        DNS Resolver             NAT                    CDN / Edge Server
   |                               |                    |                             |
   | -- DNS Query ---------------->|                    |                             |
   | <- IP Address ----------------|                    |                             |
   |                               |                    |                             |
   | -- TCP SYN to CDN IP (104.x) ----------------------------------------------->    |
   |                        [NAT rewrites src IP]                                     |
   | <- TCP SYN-ACK <------------------------------------------------------------     |
   | -- TCP ACK --------------------------------------------------------------->      |
   |                               |                    |                             |
   | 🎯 TCP connection established with CDN IP (via NAT) — Layer 4 ready              |
   |                               |                    |                             |
   | -- ClientHello ----------------------------------------------------------->      |
   |                                Suggestions / Options:                            |
   |                                  • TLS version(s)                                |
   |                                  • Cipher suites                                 |
   |                                  • SNI = domain.com                              |
   |                                  • ALPN (http/1.1, h2)                           |
   |                                  • Key share (ECDHE group)                       |
   |                               |                    |                             |
   | <- ServerHello <----------------------------------------------------------       |
   |                                Selection / Agreement:                            |
   |                                  • TLS version                                   |
   |                                  • Cipher suite                                  |
   |                                  • Selected key share                            |
   |                                  • Chosen ALPN protocol (optional)               |
   | <- EncryptedExtensions <----------------------------------------------------     |
   | <- Certificate (X.509) <----------------------------------------------------     |
   | <- CertificateVerify <------------------------------------------------------     |
   | <- Finished (🔐 encrypted) <-------------------------------------------------    |
   | -- Finished (🔐 encrypted) ------------------------------------------------>     |
   |                               |                    |                             |
   | 🎯 TLS session established — Encrypted channel ready for HTTP (Layer 6 secure)   |
```


**Key Usage Post-Derivation**

The secret key derived during the TLS handshake is not shared with the presentation layer directly. Instead, it is used internally by the TLS layer to encrypt and decrypt data as it flows between the client and the server. Here's what happens after the key is derived:

* **Session Key Creation:**
    During the TLS handshake, both the client and the server derive the same session key (a shared secret) using asymmetric cryptography and information exchanged during the handshake process.
    This session key is only known to the client and server and is never transmitted across the network.
* **Encryption and Decryption:**
    This session key is used to encrypt data before it is sent over the network and decrypt data when it is received. It ensures confidentiality and integrity during data transmission.
* **Presentation Layer Interaction:**
    The presentation layer (e.g., HTTP) uses the secure connection established by the TLS layer for communication.
    When an HTTP request is sent (like GET /image), it is passed to the TLS layer, where it is encrypted with the session key before being transmitted.
    On the receiving side, the TLS layer decrypts the data using the session key and hands it to the HTTP server.

**Relationship Between Layers**

* **L7 (Application Layer):** Uses the secure connection to transmit application-specific data.
* **L6 (Presentation Layer):** This layer may handle data formats (e.g., SSL/TLS encryption/decryption can be seen as part of this layer in broader models, but it is often considered part of transport security like L4).
* **L4 (Transport Layer):** Ensures reliable transmission of encrypted packets (TCP).

### Where TLS Fits

**TLS (Transport Layer Security)** secures data transmission between two endpoints by encrypting data before it travels over the network. While it's often mapped to **Layer 6 (Presentation Layer)** in the OSI model, it actually spans multiple layers depending on how you look at it.

---

#### 🔹 In the OSI Model

- **Layer 6 – Presentation Layer**:  
  TLS performs **encryption, decryption**, and **data integrity checks**, which align with this layer’s job of transforming data for secure transmission. It also handles **cipher suite negotiation**, **certificate validation**, and **key exchange**.

- **Layer 4 – Transport Layer (indirectly)**:  
  TLS runs **on top of TCP**, using it for reliable delivery. It does **not handle** packet ordering or retransmission—it just **secures the data before handing it to TCP**.

---

#### 🔹 In the TCP/IP Model

Since the **TCP/IP model** is more practical and widely used, TLS is generally treated as part of the **application layer stack**. It’s used directly by application protocols like **HTTPS**, **SMTP**, and **IMAP**, and is usually **handled internally by high-level libraries**, so developers don’t manage the low-level handshake or encryption manually.

The exact classification varies, but TLS’s core purpose remains: **to secure data in transit before it hits the transport layer**.

**Quick Mapping of Key Components:**

| Functionality                | Layer | Component                      |
| :--------------------------- | :---- | :----------------------------- |
| URL entry + DNS resolution   | L7    | Browser, DNS protocol          |
| NAT translation              | L3    | Home/office router (NAT + PAT) |
| TCP connection setup         | L4    | TCP 3-way handshake            |
| TLS handshake & encryption   | L6    | TLS 1.3, certificate exchange  |
| HTTP(S) request              | L7    | GET / POST requests            |
| Static/Dynamic Content fetch | L7    | CDN or backend server          |

### 2.4. CDN (Content Delivery Network)

#### 2.4.1. Cloudflare vs. Akamai

Cloudflare provides both DNS and CDN services, making it a one-stop solution for domain name resolution and content delivery. Akamai, on the other hand, is primarily known for its CDN services, providing extensive content delivery and optimization capabilities but does not offer DNS services in the same way as Cloudflare.

#### 2.4.2. CDN and DNS: Complementary Technologies

Yes, CDN (Content Delivery Network) and DNS (Domain Name System) are different but complementary technologies:

* **DNS (Domain Name System):** Translates human-readable domain names (like example.com) into IP addresses so browsers can connect to the server hosting the website.
* **CDN (Content Delivery Network):** A system of distributed servers that delivers web content (like images, videos, and scripts) to users based on their geographic location, improving load times and reducing latency.

In essence, DNS helps users find a server, while a CDN helps deliver the content from the server efficiently.

### 2.5. Understanding Port Usage: Source vs. Destination Ports

When a client (e.g., your browser) initiates a connection to a server (e.g., facebook.com), the client uses a dynamic source port (e.g., 1025, 10001, etc.), and the destination port is 443 (because the API server uses HTTPS). When someone accesses facebook.com (acting as a server), the request goes to port 443, which is the default port for HTTPS traffic.

So yes, facebook.com can use any source port as a client while making outbound requests (e.g., to an API), but it must listen on port 443 when it's the server receiving incoming requests over HTTPS.

## 3. System Communication Protocols

### 3.1. Kafka Client Connecting to a Kafka Cluster

Kafka uses a custom binary protocol over **TCP**. The Kafka protocol is designed for handling high-throughput, distributed messaging and includes specific operations for producing, consuming, and managing Kafka topics.

#### Key aspects of Kafka's protocol:

* **TCP-based binary protocol:** Kafka clients communicate with brokers over a persistent TCP connection.
* **Request-Response model:** Kafka uses a request-response pattern, where the client sends a request (e.g., produce or consume message), and the broker sends back a response.
* **Metadata exchange:** Kafka clients first contact a bootstrap server (a known broker) to fetch metadata about the cluster, including broker locations and topic partition mappings.
* **Asynchronous communication:** Kafka protocol is highly asynchronous. Producers and consumers can batch data and communicate in an asynchronous manner for efficiency.
* **Support for multiple operations:** Kafka’s protocol supports operations like `Produce`, `Fetch`, `Offset Commit`, `Metadata`, `JoinGroup`, and more.

### 3.2. Client Connecting to an RDBMS

When connecting to a traditional RDBMS, the protocol typically follows the **SQL standard** using well-known connection protocols based on the database you are using.

* **JDBC/ODBC for relational databases:** For instance, PostgreSQL uses the **PostgreSQL wire protocol**, and MySQL uses its own **MySQL protocol**.
* **Request-Response model:** Like Kafka, RDBMS uses a request-response communication pattern, but the focus is on executing SQL queries, fetching result sets, and managing database transactions.

#### Key differences:

* **SQL-based commands:** RDBMS communication revolves around SQL operations like `SELECT`, `INSERT`, `UPDATE`, and `DELETE`.
* **Connection handling:** Typically, RDBMS connections are synchronous, with each query being executed sequentially, though some support batching and asynchronous query execution.
* **ACID transactions:** RDBMSs are designed to ensure strict transactional guarantees (ACID properties), unlike Kafka, which emphasizes high throughput and availability over strict transactional integrity (though Kafka does offer idempotency and exactly-once semantics).

#### Protocols used by popular RDBMS:

* **PostgreSQL:** PostgreSQL wire protocol (over TCP)
* **MySQL:** MySQL native protocol (over TCP)
* **SQL Server:** TDS (Tabular Data Stream) protocol (over TCP)
* **Oracle:** Oracle Net (formerly SQL\*Net) protocol (over TCP/IP)

### 3.3. Summary: Kafka Protocol vs. RDBMS Protocol

* **Kafka Protocol:** Custom binary protocol over TCP, optimized for distributed messaging, high-throughput operations.
* **RDBMS Protocol:** Database-specific SQL-based protocols, typically over TCP, designed for transactional operations and SQL query execution.

In both cases, the connection to a server is via a network protocol (TCP in both cases), but the actual communication models and the focus of the protocol are quite different due to the nature of the systems (distributed messaging vs. transactional database).

## 4. High Availability and Load Balancing

### 4.1. Floating IPs: Use Cases Beyond Load Balancing

Floating IPs are used in various scenarios beyond just load balancing. While load balancers are commonly used for distributing traffic and ensuring high availability, floating IPs have distinct use cases, particularly in cloud and high-availability environments. Here are some contexts where floating IPs are used:

* **1. High Availability and Failover Mechanisms:**
    * **Scenario:** In high-availability setups, where ensuring uptime is critical, floating IPs are key.
    * **Implementation:** A floating IP is initially associated with a primary server. If the primary server fails, the floating IP is automatically or manually reassigned to a standby (backup) server. This ensures that the service remains accessible under the same IP address, minimizing downtime.
    * **Example:** A database server cluster where a floating IP moves between the active and passive nodes during a failover.
* **2. Database Replication and Failover:**
    * **Scenario:** In database systems that use master-slave replication, floating IPs can simplify failover.
    * **Implementation:** The floating IP points to the current master database. If the master fails, a slave is promoted to master, and the floating IP is remapped to the new master, providing seamless connectivity for applications without updating their configuration.
    * **Example:** PostgreSQL or MySQL replication where the application always connects to the floating IP.
* **3. Migrating Services Without Downtime:**
    * **Scenario:** When upgrading or migrating a service to new infrastructure, floating IPs can facilitate a smooth transition.
    * **Implementation:** The floating IP is initially associated with the old server. Once the new server is ready, the floating IP is moved to the new server, instantly redirecting traffic without requiring DNS changes or application restarts.
    * **Example:** Moving a web application from one EC2 instance to another in AWS by reassigning an Elastic IP (which acts as a floating IP).

### 4.2. Ensuring High Availability and Failover for Load Balancers

Ensuring high availability and failover for load balancers is critical to maintain service continuity and prevent single points of failure. Here are the key strategies:

* **1. Implementing Redundant Load Balancers:**
    * **Concept:** Deploying multiple load balancer instances instead of a single one.
    * **Implementation:** This can involve an active-passive setup (one primary, one standby) or an active-active setup (both handle traffic simultaneously). If one load balancer fails, traffic is automatically diverted to the healthy instance(s).
    * **Example:** Setting up two AWS Application Load Balancers (ALBs) in different availability zones.
* **2. Leveraging Global Server Load Balancing (GSLB):**
    * **Concept:** Distributing traffic across multiple geographically dispersed load balancers or data centers.
    * **Implementation:** GSLB solutions (often integrated with DNS) direct users to the closest or healthiest data center/load balancer based on factors like latency, geography, or health checks. If an entire region or data center goes down, traffic is routed to another active location.
    * **Example:** Using AWS Route 53 with latency-based routing or Google Cloud Load Balancing’s global capabilities.
* **3. Configuring DNS Failover:**
    * **Concept:** Using DNS to redirect traffic away from unhealthy load balancers or endpoints.
    * **Implementation:** Use a DNS provider that supports health checks and failover. Configure health checks for your primary load balancer's endpoint. Set up failover DNS records to point to a secondary load balancer or an alternate endpoint. If the primary fails, the DNS record is updated to point to the backup.
* **4. Implementing Floating IPs for Load Balancers:**
    * **Concept:** A single IP address that can be moved between different load balancer instances.
    * **Implementation:** Assign a floating IP to your primary load balancer. In the event of a failure, manually or automatically reassign the floating IP to a backup load balancer. This provides a static entry point for clients, regardless of which load balancer is active.

**Real-World Example**

* **1. Amazon Web Services (AWS):**
    * **Example:** AWS Elastic Load Balancer (ELB) automatically distributes incoming application traffic across multiple targets, suchs as EC2 instances. AWS Route 53 can be used for DNS failover, where health checks are performed on the ELB, and traffic is rerouted to a backup ELB in case of failure.
* **2. Google Cloud Platform (GCP):**
    * **Example:** Google Cloud Load Balancing provides global load balancing with automatic failover capabilities. Traffic Director can be used for advanced traffic management and failover scenarios.
* **3. Microsoft Azure:**
    * **Example:** Azure Load Balancer can be set up with multiple instances for redundancy. Azure Traffic Manager can be used for DNS-based load balancing and failover across different regions.

**Summary**
Ensuring high availability and failover for load balancers involves using redundant load balancers, global server load balancing, DNS failover, and floating IPs. These strategies help maintain service continuity and minimize disruption in case of a load balancer failure.

### 4.3. Classic Load Balancer (ELB)

This is the oldest type of load balancer offered by AWS and is now considered legacy for most new applications.

✅ **When to Use:**

* **Legacy applications** that don't need advanced features.
* **Simple HTTP/HTTPS or TCP** load balancing.
* **Basic health checks**.

❌ **Gotcha:** ELB only supports one application per instance, making it less efficient for microservices.

🚀 **Example:**
A traditional **web application** with a single server that needs basic traffic distribution.

💡 **Used with:** Older monolithic applications, simple websites.

### 4.4. Application Load Balancer (ALB)

ALB operates at the Application Layer (Layer 7) and is ideal for modern, complex applications.

✅ **When to Use:**

* **Microservices architectures** or containerized applications (e.g., Docker, Kubernetes).
* Need for **HTTP/HTTPS load balancing**.
* **Path-based routing** (e.g., `/users` to one service, `/products` to another).
* **Host-based routing** (e.g., `api.example.com` to one service, `www.example.com` to another).
* **WebSocket support**.
* **Advanced request routing** based on headers, query strings, or methods.
* **Serverless applications** (e.g., AWS Lambda, AWS Fargate).

🚀 **Example:**
A **microservices-based e-commerce platform** where different services handle users, products, and orders.

💡 **Used with:** RESTful APIs, modern web apps, serverless functions.

### 4.5. Network Load Balancer (NLB)

NLB operates at the Transport Layer (Layer 4) and is designed for extreme performance and low latency.

✅ **When to Use:**

* **Ultra-low latency** required (milliseconds-level).
* Need to handle **millions of requests per second**.
* Load balancing for **TCP, UDP, and TLS-based** applications.
* **Static IP support** needed (ALB/ELB don’t provide static IPs).

🚀 **Example:**
A **high-frequency trading platform** that needs **low latency and fast failover** when a server crashes.

💡 **Used with:** Financial apps, gaming, IoT, real-time streaming.

### 4.6. Summary: ELB, ALB, NLB Comparison Table

| **Feature** | **ELB (Classic LB)** | **ALB (Application LB)** | **NLB (Network LB)** |
| :------------------ | :------------------- | :----------------------- | :--------------------------- |
| **Best for** | Legacy apps, simple LB | Modern apps, microservices | High-performance, real-time apps |
| **Routing** | Basic HTTP, TCP      | Path & host-based        | TCP/UDP, low-latency         |
| **WebSocket Support** | ❌ No                | ✅ Yes                   | ✅ Yes                       |
| **Latency** | Medium               | Medium                   | **Ultra-low** (sub-millisecond) |
| **Protocols** | HTTP, HTTPS, TCP     | HTTP, HTTPS              | TCP, UDP, TLS                |
| **Static IPs** | ❌ No                | ❌ No                    | ✅ Yes                       |

