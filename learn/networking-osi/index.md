# Networking & the OSI Model: A Technician's Guide

Most courses make you memorize seven layers. This guide uses them as a **troubleshooting checklist**. When something is broken, you walk up the layers from the bottom and stop at the first one that fails. That tells you where the problem is.

---

## 1. The 30-Second Mental Model

Sending data is like posting a parcel. You put the letter in an envelope (Layer 4), write the street address on it (Layer 3), hand it to the local courier who knows which door to knock on (Layer 2), and the courier drives the road (Layer 1). Each layer only cares about its own job and wraps the data it receives from the layer above.

```text
Layer 7  Application   HTTP, DNS, TLS        "What are we saying?"
Layer 6  Presentation  Encoding, encryption  (in practice, part of 7)
Layer 5  Session       Connection state      (in practice, part of 7)
Layer 4  Transport     TCP / UDP, ports      "Which program, and reliably?"
Layer 3  Network       IP, routing           "Which network, which host?"
Layer 2  Data Link     Ethernet, MAC, ARP    "Which device on this LAN?"
Layer 1  Physical      Cables, Wi-Fi, light  "Is there a signal?"
```

The real internet uses the 4-layer **TCP/IP model**. It merges OSI layers 5-7 into one Application layer and layers 1-2 into one Link layer. OSI is still the common vocabulary, because "that's a Layer 2 problem" is faster to say than a paragraph.

**Rule of thumb:** work bottom-up. A DNS error at Layer 7 is a waste of time if the cable is unplugged.

---

## 2. How It Works in Practice

### Layer 1: Physical
Link lights, cables, transceivers, Wi-Fi signal. Nothing to configure; you check whether the link is up.

```bash
# Linux
ip link show              # look for "state UP" and LOWER_UP
ethtool eth0              # speed, duplex, "Link detected: yes"
```
```powershell
# Windows
Get-NetAdapter            # Status should be Up
```

### Layer 2: Data Link
Devices on the same LAN find each other by **MAC address**. **ARP** maps an IP address to a MAC address. Switches forward frames by MAC.

```bash
ip neigh show             # Linux ARP/neighbour table
arp -a                    # Windows / macOS / Linux
```

### Layer 3: Network
**IP addresses** identify hosts and networks. A **routing table** decides where each packet goes next. Packets for networks the host doesn't know about go to the **default gateway**.

```bash
ip addr show              # my IP address and subnet mask
ip route show             # routing table, look for "default via ..."
ping 192.168.1.1          # can I reach my gateway?
traceroute 8.8.8.8        # hop-by-hop path (Windows: tracert 8.8.8.8)
```

### Layer 4: Transport
**TCP** is reliable and ordered. It opens a connection with a three-way handshake:

```text
Client                      Server
  | ---- SYN ------------->  |
  | <--- SYN-ACK ----------  |
  | ---- ACK ------------->  |   connection established
```

**UDP** skips the handshake and the guarantees. It is faster and is used by DNS queries, video calls and games. A **port** identifies the program: 22 SSH, 53 DNS, 80 HTTP, 443 HTTPS.

```bash
ss -tulpn                 # listening TCP/UDP sockets and owning process
ss -tn state established  # active TCP connections
nc -vz example.com 443    # is the port reachable? (Windows: Test-NetConnection example.com -Port 443)
```

`netstat -an` shows similar information on older systems.

### Layer 7: Application
This is where users see problems. Opening `https://www.lukaslim.com` involves:

1. **DNS** turns the name into an IP address.
2. **TCP** handshake to port 443.
3. **TLS** handshake. The server presents a certificate and both sides agree on encryption keys.
4. **HTTP** request and response, with a status code.

```bash
dig www.lukaslim.com             # DNS answer (Windows: nslookup www.lukaslim.com)
dig +short www.lukaslim.com
curl -I https://www.lukaslim.com # HTTP status code and headers
curl -v https://www.lukaslim.com # shows DNS, TCP, TLS and HTTP steps
```

Common HTTP status classes: `2xx` success, `3xx` redirect, `4xx` client error (404 not found, 403 forbidden), `5xx` server error (502/503 bad gateway/unavailable).

---

## 3. What Happens When It Breaks

Run the checks in this order. **The first one that fails points to the layer.**

| Step | Command | If it fails | Layer |
|------|---------|-------------|-------|
| 1. Is the link up? | `ip link show` | Cable, port, adapter, Wi-Fi off | 1 |
| 2. Do I have an IP? | `ip addr show` | DHCP failure, or a `169.254.x.x` address | 2-3 |
| 3. Can I reach the gateway? | `ping <gateway>` | Bad VLAN, switch issue, wrong subnet | 2-3 |
| 4. Can I reach the internet by IP? | `ping 8.8.8.8` | Routing, firewall, ISP | 3 |
| 5. Does DNS work? | `dig example.com` | DNS server unreachable or wrong | 7 |
| 6. Is the port open? | `nc -vz example.com 443` | Firewall, service down | 4 |
| 7. Does the app answer? | `curl -I https://example.com` | Web server or certificate problem | 7 |

### Reading the symptoms

| Symptom | Likely cause |
|---------|--------------|
| Ping to the gateway fails | Layer 1-2: cable, switch port, VLAN |
| Ping by IP works, by name fails | Layer 7: DNS |
| Ping works, website won't load | Layer 4 or 7: port blocked, service down |
| `Connection refused` | Host is reachable but nothing is listening on that port |
| `Connection timed out` | Packets are being dropped, often by a firewall |
| `Destination host unreachable` | Layer 3: no route, or ARP failed |
| Certificate error in browser | Layer 7: expired, wrong name, or untrusted TLS certificate |
| Connected but slow | Congestion, packet loss (`ping` loss %, `mtr`), or an overloaded server |

**Refused vs. timed out is a useful distinction.** Refused means the host answered "no". Timed out means nobody answered at all, so look at firewalls and routing.

---

## 4. Hands-On Lab

You only need a terminal and an internet connection. Everything here is read-only.

**Goal:** trace what happens when you load a website, layer by layer.

1. **Layer 1-2:** List your interface and gateway.
   ```bash
   ip link show && ip route show
   ```
   Note the interface name and the gateway after `default via`.
2. **Layer 2:** Ping the gateway, then look at the ARP table.
   ```bash
   ping -c 3 <gateway-ip>
   ip neigh show
   ```
   Find the gateway's MAC address.
3. **Layer 3:** Trace the path to a public IP.
   ```bash
   traceroute 8.8.8.8
   ```
   Count the hops. Which one adds the most latency?
4. **Layer 7 (DNS):** Resolve the site.
   ```bash
   dig +short www.lukaslim.com
   ```
5. **Layer 4:** Check that port 443 is open.
   ```bash
   nc -vz www.lukaslim.com 443
   ```
6. **Layer 7 (HTTP/TLS):** Watch the whole sequence.
   ```bash
   curl -v https://www.lukaslim.com 2>&1 | head -40
   ```
   Find the lines for the TCP connect, the TLS handshake, the certificate, and the `HTTP/2 200` response.
7. **Break it on purpose.** Query a name that doesn't exist and a port that is closed, and note the different errors.
   ```bash
   dig doesnotexist.lukaslim.com
   nc -vz www.lukaslim.com 81
   ```

**Check yourself:** for each of your three failures above, can you name the layer without looking at the table?

---

## Further Reading
- [Diagnose a network problem on Linux](https://www.stephane-robert.info/en/docs/linux/networking/diagnostics/)
- [Network Troubleshooting: Ping, Traceroute & Debug Methods](https://www.networkershome.com/fundamentals/networking/network-troubleshooting-ping-traceroute-debug/)
- [OSI Model to TCP/IP Mapping](https://computingforgeeks.com/osi-tcp-ip-model-explained/)
- [Network Troubleshooting Workflow](https://inventivehq.com/blog/network-troubleshooting-workflow)
