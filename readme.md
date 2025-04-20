# Orchestrator 
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

On your POWDER VM (or any Ubuntu‑based Linux host):

- **Docker** & **Docker Compose** (v2.x)  
- **Python 3** (≥ 3.6)  
- **xterm** (for demo windows)  
- `curl`, `git`, `md5sum`  

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

| Link          | Subnet          | A IP /24   | B IP /24   |
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
├── Dockerfile            # Builds a base host/router container
├── dockersetup           # Installs Docker & Compose
├── orchestrator.py       # Python script (executable)
└── PA3 Demo Video.mov    # Demo Video
└── README.md             
```

---

## Initial Setup

```bash
# 1. Clone the student repo
git clone https://github.com/V-AnandhOfficial/PA3.git
cd PA3

# 2. Install Docker & Compose (provided script)
./dockersetup

# 3. Verify Docker is running
docker ps
docker network ls

# 4. Make the orchestrator executable
chmod +x orchestrator.py
```

---

## Docker Compose Topology

The `docker-compose.yaml` brings up 6 containers (hosta, r1, r2, r3, r4, hostb) and 6 bridge networks. To start:

```bash
# Start all containers in detached mode
docker compose up -d

# Confirm
docker ps
docker network ls
```

---

## Orchestrator Usage

The orchestrator script wraps all setup and traffic‑switching steps:

```bash
./orchestrator.py --help
```

### Key Flags

- `--start`  
  Starts containers (`docker compose up -d`).

- `--setup`  
  Equivalent to:  
  1. `--start`  
  2. install FRR & OSPF on all routers  
  3. configure OSPF (north path by default)  
  4. configure host routes

- `--show-neighbors`  
  Display `show ip ospf neighbor` on each router.

- `--show-routes`  
  Display OSPF-learned routes on each router.

- `--ping`  
  Run `ping -c4 10.0.15.3` from HostA.

- `--traceroute`  
  Run `traceroute -n 10.0.15.3` from HostA.

- `--continuous-ping`  
  Start a background `ping 10.0.15.3` from HostA (Ctrl+C to stop).

- `--switch-to-south`  
  Adjust OSPF link costs to prefer R1→R4→R3.

- `--switch-to-north`  
  Adjust OSPF link costs to prefer R1→R2→R3.

- `--stop`  
  Tear down containers (`docker compose down`).

---

## Demo Steps

1. **Bring up & configure**  
   ```bash
   ./orchestrator.py --setup
   ```

2. **Open four xterms** (titles for clarity):
   ```bash
   xterm -T HostA -e "docker exec -it hosta bash -c 'ping 10.0.15.3'" &
   xterm -T R1    -e "docker exec -it r1    bash -c 'tcpdump -i any icmp'" &
   xterm -T R4    -e "docker exec -it r4    bash -c 'tcpdump -i any icmp'" &
   xterm -T HostB -e "docker exec -it hostb bash -c 'tcpdump -i any icmp'" &
   ```

3. **Show baseline path**  
   ```bash
   ./orchestrator.py --traceroute
   # Expect hops via 10.0.10.4 → 10.0.11.4
   ```

4. **Start continuous ping** (in HostA xterm)  
   Already running from step 2.

5. **Switch North → South**  
   In your host terminal:
   ```bash
   ./orchestrator.py --switch-to-south
   ```
   Watch all four xterms—no packet loss.

6. **Verify new path**  
   ```bash
   ./orchestrator.py --traceroute
   # Expect hops via 10.0.13.3 → 10.0.12.3
   ```

7. **Switch South → North**  
   ```bash
   ./orchestrator.py --switch-to-north
   ```
   Again, watch zero loss.

8. **End continuous ping**  
   Ctrl+C in the HostA xterm.

---

## Cleanup

```bash
./orchestrator.py --stop
```

All containers and networks will be removed.
