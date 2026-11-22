---
layout: post
title: "Linux Network Troubleshooting Fundamentals for DevOps Engineers"
date: 2026-11-22 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [linux, networking, troubleshooting, devops, tcpdump, ss, ip]
description: "Practical Linux network troubleshooting commands and patterns every DevOps engineer should have in their toolkit, with safe examples for diagnosing connectivity, latency, and configuration issues."
---

# Linux Network Troubleshooting Fundamentals for DevOps Engineers

A DevOps engineer's most common infrastructure problems — failed deployments, unreachable services, or latency spikes — almost always have a network root cause. This article compiles the essential Linux networking commands and inspection patterns for diagnosing and resolving these issues quickly.

## 1. Immediate connectivity checks

Before reaching for packet capture tools, verify the basic path:

```bash
# Is the target host reachable?
ping -c 4 example.com

# DNS resolution path
nslookup example.com
dig example.com +short

# Route packet takes to reach the destination
traceroute example.com

# Alternative to traceroute (shows TCP-level path)
traceroute -T -p 80 example.com
```

All three commands are read-only and safe on any production host.

## 2. Socket and port inspection with `ss`

The `ss` utility replaces `netstat` and queries the kernel directly — it's faster and more reliable.

```bash
# List all listening TCP ports
ss -tln

# Show established connections to a specific port
ss -tlnp 'sport = :8080'

# Display connections in TIME_WAIT state (helps diagnose port exhaustion)
ss -ntw 'state FINITIMEWAIT'

# View processes owning sockets (requires root or netns)
ss -tlnp
```

The `-p` flag reveals the PID and program name associated with each socket — invaluable when a port is unexpectedly held.

## 3. Interface and device diagnostics

```bash
# Show interface statistics (bytes, packets, errors, drops)
ip -s link

# Check interface IP addresses and MTU
ip addr show dev eth0
ip -details link show dev eth0

# Interface queue length and overrun metrics
ethtool -S eth0

# Check for interface collisions or carrier losses
ethtool eth0 | grep -i "link\|speed\|duplex"
```

High error counts or dropped packets on an interface often point to cabling, duplex mismatch, or virtualization overhead.

## 4. Packet capture with `tcpdump`

`tcpdump` captures live traffic. All examples below are read-only — no packets are modified or dropped.

```bash
# Capture TCP traffic to/from port 8080, write to file
tcpdump -i eth0 -nn 'port 8080' -w /tmp/tcpdump-8080.pcap

# Capture only HTTP GET/POST requests (requires tcpdump with -A flag)
tcpdump -i eth0 -A -s 0 'port 80 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420' -w /tmp/http-get.pcap

# Limit capture size and duration (safe for production)
tcpdump -i eth0 -nn 'port 22' -w /tmp/sshd-traffic.pcap -c 1000

# Exclude your own machine's traffic (useful on multi-homed hosts)
tcpdump -i eth0 -nn 'not host $(hostname -I | awk "{print \$1}")' -w /tmp/external-traffic.pcap
```

Captured `.pcap` files can be analyzed with `tcpdump -r` or imported into Wireshark for deeper inspection.

## 5. DNS debugging

```bash
# Query specific DNS record types
dig @8.8.8.8 example.com AXFR
dig +trace example.com

# Lookup with response time and flags
dig example.com ANY +stats

# Verify reverse DNS
dig -x 8.8.8.8 +ptr

# Test local /etc/hosts takes precedence
getent host example.com
```

## 6. TCP-level diagnostics

```bash
# Test TCP connectivity and measure round-trip time
nc -zv example.com 80

# Connect with a timeout (exits if no response)
timeout 5 nc example.com 443 < /dev/null && echo "port open" || echo "no response"

# Show TCP route via /proc/net/tcp6 (hex addresses, readable with python)
python3 -c "
import sys
with open('/proc/net/tcp') as f:
    for line in f:
        parts = line.split()
        if len(parts) > 1:
            local = parts[1]
            addr = int(local.split(':')[0], 16)
            port = int(local.split(':')[1], 16)
            print(f'local={addr:#x}:{port}')
"
```

## 7. Common patterns and anti-patterns

| Symptom | Likely cause | Command to verify |
|---|---|---|
| `curl` hangs | DNS failure or firewall block | `dig +time=2 +tries=1 example.com`; `iptables -L -n` |
| `curl` connects but times out | Server not listening or rate-limited | `ss -tlnp | grep :8080`; `curl -v http://localhost:8080/health` |
| High latency on localhost | Interrupt throttling or socket buffer size | `ethtool -C eth0 rx-usecs=100`; `sysctl net.core.rmem_max` |
| `Connection refused` on expected port | Service not started or wrong port | `systemctl status <service>`; `ss -tlnp | :${PORT}` |

## 8. One-liner quick checks

```bash
# Is port 3000 listening locally?
ss -tlnp | grep ':3000 ' && echo "listening" || echo "not listening"

# How many SYN packets per second is the host receiving?
watch -n 1 'iptables -A INPUT -p tcp --syn -j DROP 2>/dev/null; cat /proc/net/tcp | awk "{print \$2}" | grep -c ":01\b"'

# Is the default gateway reachable?
ip route | grep default && ping -c 2 $(ip route | grep default | awk "{print \$3}")
```

---

**Remember**: Network troubleshooting is iterative. Start broad (ping, traceroute), narrow the scope (ss, tcpdump), and drill into the specific layer (TCP flags, DNS, routing). All commands shown are read-only and safe to run on production hosts. When in doubt, capture a pcap first, then analyze — never modify live network configurations without a rollback plan.

---
*This article is intended for educational and operational use. Commands should be adapted to your environment and run with appropriate permissions. Never run packet capture or diagnostic commands on production systems without monitoring the impact.*