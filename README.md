# Cisco-Trex---GNS3
Step-by-step lab guide for running Cisco TRex traffic generator in GNS3 using Docker. Covers OSPF routing between two routers, ARP resolution, and stateless traffic testing with IMIX profiles.

# 🚀 TRex Traffic Generator Lab — GNS3 Setup

A complete lab guide for running **Cisco TRex** as a stateless traffic generator between two Cisco routers (R1 & R3) using GNS3 and Docker.

---

## 📋 Table of Contents

- [Lab Topology](#lab-topology)
- [Prerequisites](#prerequisites)
- [Router Configuration](#router-configuration)
- [TRex Setup](#trex-setup)
- [Running TRex](#running-trex)
- [TRex Console — Service Mode & ARP](#trex-console--service-mode--arp)
- [Sending Traffic](#sending-traffic)
- [Verification](#verification)

---

## 🗺️ Lab Topology

```
[TRex Port 0]──────[R1 Gi0/0]──OSPF──[R3 Gi0/0]──────[TRex Port 1]
10.10.10.2/30     10.10.10.1/30     7.7.7.1/30        7.7.7.2/30

         R1 Gi0/2 ─────────────────── R3 Gi0/2
          13.1.1.1/30               13.1.1.2/30
```

| Device   | Interface  | IP Address    | Role             |
|----------|------------|---------------|------------------|
| TRex     | Port 0     | 10.10.10.2/30 | Traffic Source   |
| R1       | Gi0/0      | 10.10.10.1/30 | Default GW       |
| R1       | Gi0/2      | 13.1.1.1/30   | OSPF Link        |
| R3       | Gi0/2      | 13.1.1.2/30   | OSPF Link        |
| R3       | Gi0/0      | 7.7.7.1/30    | Default GW       |
| TRex     | Port 1     | 7.7.7.2/30    | Traffic Sink     |

---

## ✅ Prerequisites

- GNS3 installed with Docker support
- Cisco IOS images for R1 and R3
- Docker installed on the host

---

## 🔧 Router Configuration

### R1

```
interface GigabitEthernet0/0
 ip address 10.10.10.1 30
!
interface GigabitEthernet0/2
 ip address 13.1.1.1 30
!
router ospf 1
 network 0.0.0.0 255.255.255.255 area 0
```

### R3

```
interface GigabitEthernet0/0
 ip address 7.7.7.1 30
!
interface GigabitEthernet0/2
 ip address 13.1.1.2 30
!
router ospf 1
 network 0.0.0.0 255.255.255.255 area 0
```

> Both routers use OSPF area 0 with a wildcard `network 0.0.0.0 255.255.255.255` to advertise all interfaces.

---

## 🐳 TRex Setup

### Pull and Run the Docker Image

```bash
# Pull the TRex image from Docker Hub
docker pull trexcisco/trex

# Run the container in privileged mode
docker run --rm -it --privileged --cap-add=ALL trexcisco/trex

# Start TRex interactive mode
./t-rex-64 -i
```

### Configuration File

TRex is configured via `/etc/trex_cfg.yaml`. The MAC addresses must match the **destination router interfaces**.

```yaml
- port_limit    : 2
  version       : 2
  low_end       : true
  interfaces    : ["veth0", "veth1"]
  port_info     :
                 - ip         : 10.10.10.2
                   default_gw : 10.10.10.1
                   dest_mac   : "0c:2f:6e:70:00:00"   # R1 Gi0/0 MAC
                 - ip         : 7.7.7.2
                   default_gw : 7.7.7.1
                   dest_mac   : "0c:d2:b4:d8:00:00"   # R3 Gi0/0 MAC
```

> 💡 **Tip:** Get the destination MAC addresses from `show arp` or `show interfaces` on the routers.

---

## ▶️ Running TRex

```bash
[root@trex v2.41]# ./t-rex-64 -i
```

Expected output confirms 2 ports are detected and links are UP:

```
Number of ports found: 2
port : 0  →  Link Up - speed 10000 Mbps - full-duplex
port : 1  →  Link Up - speed 10000 Mbps - full-duplex
```

---

## 🖥️ TRex Console — Service Mode & ARP

From a **second terminal**, attach to the running container and open the console:

```bash
# Find the container ID
docker ps

# Exec into the container
docker exec -it <CONTAINER_ID> bash

# Launch TRex console
./trex-console
```

### ARP Resolution & Connectivity Test

```bash
trex> service -a              # Enter service mode (all ports)
trex(service)> arp            # Resolve ARP for both ports
trex(service)> ping -p 0 -d 10.10.10.1   # Ping R1 from Port 0
trex(service)> ping -p 0 -d 7.7.7.1      # Ping R3 from Port 0 (via OSPF)
trex(service)> service --off  # Exit service mode
```

### Expected ARP Results

```
Port 0 - Received ARP reply from: 10.10.10.1, hw: 0c:2f:6e:70:00:00
Port 1 - Received ARP reply from: 7.7.7.1,    hw: 0c:d2:b4:d8:00:00
```

---

## 📡 Sending Traffic

Start a stateless IMIX traffic profile at 10,000 packets per second from Port 0:

```bash
trex> start -f stl/imix.py -m 10kpps --port 0
```

### Sample Output

```
Total-Tx   : 28.99 Mb/sec
Total-Rx   : 163.80 b/sec
Total-PPS  : 10.02 Kpkt/sec
drop_rate  : 28.99 Mb/sec      ← High drop = no return path configured on Port 1
```

> ⚠️ **Note:** High drop rate is expected when only Port 0 is transmitting. Configure both ports for bidirectional testing.

---

## 🔍 Verification

### Check ARP Table on R1

```
R1# show arp

Protocol  Address        Age  Hardware Addr    Type  Interface
Internet  10.10.10.1      -   0c2f.6e70.0000   ARPA  GigabitEthernet0/0
Internet  10.10.10.2      1   0242.9ff0.0b00   ARPA  GigabitEthernet0/0  ← TRex Port 0
```

### Check Port Attributes in TRex Console

```bash
trex(service)> portattr
```

| Field          | Port 0        | Port 1      |
|----------------|---------------|-------------|
| src IPv4       | 10.10.10.2    | 7.7.7.2     |
| Destination    | 10.10.10.1    | 7.7.7.1     |
| ARP Resolution | 0c:2f:6e:70:00:00 | 0c:d2:b4:d8:00:00 |
| Link           | UP            | UP          |

---

## 📚 References

- [TRex Documentation](https://trex-tgn.cisco.com/trex/doc/)
- [TRex Docker Hub](https://hub.docker.com/r/trexcisco/trex)
- [TRex Stateless API](https://trex-tgn.cisco.com/trex/doc/trex_stateless.html)
