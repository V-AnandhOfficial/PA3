# Orchestrator  
**Author:** Vivek Anandh  
**Date:** April 19, 2025

This README guides you through setting up the Docker‑based network topology, installing FRR/OSPF on the routers, configuring the hosts, and using the Python orchestrator to dynamically switch traffic between the two OSPF paths (north: R1→R2→R3, south: R1→R4→R3) without packet loss.

---

## Table of Contents

1. [Prerequisites](#prerequisites)  
2. [Topology & IP Plan](#topology--ip-plan)  
3. [Repository Layout](#repository-layout)  
4. [Initial Setup](#initial-setup)  
5. [Docker Compose Topology](#docker-compose-topology)  
6. [Orchestrator Usage](#orchestrator-usage)  
7. [Demo Steps](#demo-steps)  
8. [Cleanup](#cleanup)  

---

## Prerequisites

On POWDER VM:

- Docker & Docker Compose  
- Python 3  
- `curl`, `git`, `md5sum`  
- `xterm` (for demo windows)

---

## Topology & IP Plan

```
                -R2-
              /       \
             /         \
   HostA —— R1         R3 —— HostB
             \        /
              \      /
                -R4-
```

| Link          | Subnet          | A IP /24   | B IP /24   |
|--------------:|:---------------:|:----------:|:----------:|
| HostA–R1      | 10.0.14.0/24    | 10.0.14.3  | 10.0.14.4  |
| R1–R2 (north) | 10.0.10.0/24    | 10.0.10.3  | 10.0.10.4  |
| R1–R4 (south) | 10.0.13.0/24    | 10.0.13.4  | 10.0.13.3  |
| R2–R3         | 10.0.11.0/24    | 10.0.11.3  | 10.0.11.4  |
| R4–R3         | 10.0.12.0/24    | 10.0.12.4  | 10.0.12.3  |
| R3–HostB      | 10.0.15.0/24    | 10.0.15.4  | 10.0.15.3  |

---

## Repository Layout

```
PA3/
├── docker-compose.yaml
├── Dockerfile            # Builds base host/router container
├── dockersetup           # Installs Docker & Compose
├── Vivek_Anandh_U1241037.py  # Orchestrator script (executable)
└── README.md             # ← you are here
```

---

## Initial Setup

```bash
# Clone the repo
git clone https://github.com/V-AnandhOfficial/PA3.git
cd PA3

# Install Docker & Compose
./dockersetup

# Verify Docker is running
docker ps
docker network ls

# Make orchestrator executable
chmod +x Vivek_Anandh_U1241037.py
```

---

## Docker Compose Topology

Bring up containers:

```bash
docker compose up -d
docker ps
docker network ls
```

---

## Orchestrator Usage

Show help:

```bash
./Vivek_Anandh_U1241037.py --help
```

### Key Flags

- `--start`               Start containers  
- `--setup`               Start containers, install FRR, configure OSPF, set host routes  
- `--show-neighbors`      `show ip ospf neighbor` on all routers  
- `--show-routes`         Display OSPF routes on all routers  
- `--ping`                `ping -c4 10.0.15.3` from HostA  
- `--traceroute`          `traceroute -n 10.0.15.3` from HostA  
- `--continuous-ping`     Continuous `ping 10.0.15.3` from HostA  
- `--switch-to-south`     Force path R1→R4→R3  
- `--switch-to-north`     Force path R1→R2→R3  
- `--stop`                `docker compose down`

---

## Demo Steps

1. **Setup**  
   ```bash
   ./Vivek_Anandh_U1241037.py --setup
   ```

2. **Open four xterms**:  
   ```
   sudo bash
   docker exec -it r1 tcpdump -i any icmp
   ```
   ```
   sudo bash
   docker exec -it r4 tcpdump -i any icmp
   ```
   ```
   sudo bash
   docker exec -it hostb tcpdump -i any icmp
   ```
   ```
   sudo bash
   docker exec -it hosta ping 10.0.15.3
   ```

3. **Baseline path**  
   ```bash
   ./Vivek_Anandh_U1241037.py --traceroute
   # Expect hops via 10.0.10.4 → 10.0.11.4
   ```

4. **Switch North→South**  
   ```bash
   ./Vivek_Anandh_U1241037.py --switch-to-south
   ```
   Watch no packet loss.

5. **Verify South path**  
   ```bash
   ./Vivek_Anandh_U1241037.py --traceroute
   # Expect hops via 10.0.13.3 → 10.0.12.3
   ```

6. **Switch South→North**  
   ```bash
   ./Vivek_Anandh_U1241037.py --switch-to-north
   ```
   Watch no packet loss.

7. **End ping**  
   Ctrl+C in HostA xterm.

---

## Cleanup

```bash
./Vivek_Anandh_U1241037.py --stop
```
