# BGP Triangle Lab (Day 7)

Three routers in three autonomous systems, connected in a triangle, running eBGP. Built with FRR 10.2.1 and Containerlab 0.81.1 on an Ubuntu VM (OrbStack, M2 Mac). The lab shows how BGP picks the best path using AS_PATH length, and how it fails over when a link goes down.

## Topology

| Router | AS | Loopback |
|---|---|---|
| r1 | 65001 | 1.1.1.1/32 |
| r2 | 65002 | 2.2.2.2/32 |
| r3 | 65003 | 3.3.3.3/32 |

| Link | Subnet | Interfaces |
|---|---|---|
| r1-r2 | 10.0.12.0/30 | r1:eth1 (.1) to r2:eth1 (.2) |
| r1-r3 | 10.0.13.0/30 | r1:eth2 (.1) to r3:eth1 (.2) |
| r2-r3 | 10.0.23.0/30 | r2:eth2 (.1) to r3:eth2 (.2) |

Each router has two eBGP neighbors and announces its own loopback with a network statement. no bgp ebgp-requires-policy is set because FRR blocks eBGP routes until a policy exists.

## How to run

    sudo containerlab deploy -t triangle.clab.yml
    docker exec clab-bgp-triangle-r3 vtysh -c "show ip bgp summary"
    docker exec clab-bgp-triangle-r3 vtysh -c "show ip bgp"

Break and heal the r1-r3 link:

    docker exec clab-bgp-triangle-r1 ip link set eth2 down
    docker exec clab-bgp-triangle-r1 ip link set eth2 up

## What I observed

- 01-r3-bgp-normal.txt: r3 holds 5 paths. For 1.1.1.1/32 it learns two: direct from r1 (AS_PATH 65001) and via r2 (65002 65001). The shorter one is best.
- 02-r3-bgp-failover.txt: with r1 eth2 down, the session to r1 goes Active and its routes are withdrawn. r3 has 3 paths and reaches 1.1.1.1/32 only via r2 (65002 65001).
- 03-r3-bgp-recovered.txt: after eth2 comes back up, the session re-establishes and the direct path 65001 is best again (5 paths).

## Key concepts

- AS_PATH is the list of ASes a route has crossed. Each eBGP hop adds its own AS, and a shorter AS_PATH wins.
- BGP keeps backup paths in its table. They are marked * (valid) and only the best is marked >.
- When a session dies, every route learned over it is withdrawn, and the next best path takes over with no config change.
