# SRv6 FlexAlgo Telemetry Lab (Nokia SR OS)


[![Discord][discord-svg]][discord-url] [![DevPod][devpod-svg]][devpod-url] [![Codespaces][codespaces-svg]][codespaces-url]  
![w212][w212][Learn more](https://containerlab.dev/macos/#devpod) ![w90][w90][Learn more](https://containerlab.dev/manual/codespaces)

[discord-svg]: https://gitlab.com/rdodin/pics/-/wikis/uploads/b822984bc95d77ba92d50109c66c7afe/join-discord-btn.svg
[discord-url]: https://discord.gg/tZvgjQ6PZf
[devpod-svg]: https://gitlab.com/rdodin/pics/-/wikis/uploads/dfc36636ecaa60f3e70340686d5800db/open-in-devpod-btn.svg
[devpod-url]: https://devpod.sh/open#https://github.com/sros-labs/srv6-flexalgo-telemetry-lab
[codespaces-svg]: https://gitlab.com/rdodin/pics/-/wikis/uploads/80546a8c7cda8bb14aa799d26f55bd83/run-codespaces-btn.svg
[codespaces-url]: https://codespaces.new/sros-labs/srv6-flexalgo-telemetry-lab?quickstart=1&devcontainer_path=.devcontainer%2Fdocker-in-docker%2Fdevcontainer.json
[w212]: https://gitlab.com/rdodin/pics/-/wikis/uploads/718a32dfa2b375cb07bcac50ae32964a/w212h1.svg
[w90]: https://gitlab.com/rdodin/pics/-/wikis/uploads/bf1b8ea28b4528eb1b66567355a13c5c/w90h1.svg

---

## 📖 Table of Contents

- [🎯 Lab Overview](#-lab-overview)
- [🧠 Concepts Deep Dive](#-concepts-deep-dive)
  - [Understanding SRv6](#-understanding-srv6)
  - [FlexAlgo Explained](#-flexalgo-explained)
- [🗺️ Network Topology](#️-network-topology)
- [⚙️ SR-SIM Configuration Deep Dive](#️-sr-sim-configuration-deep-dive)
- [🚀 Quick Start](#-quick-start)
- [🔬 Checking the Lab](#-checking-the-lab)
- [📊 Telemetry Stack](#-telemetry-stack)
- [🔧 Traffic Generation & Delay Manipulation](#-traffic-generation--delay-manipulation)
- [📚 References](#-references)

---

## 🎯 Lab Overview

This lab demonstrates **traffic-engineered paths using SRv6 transport with FlexAlgo** between two customer endpoints (Client1 → R1 → R5 → Client2) with:

| Component | Technology |
|-----------|------------|
| **Transport** | Classic [SRv6](https://www.nokia.com/networks/ip-networks/segment-routing/) (Algo 0) + FlexAlgo 128 with delay metric |
| **Service** | [EVPN](https://www.nokia.com/networks/ethernet-vpn/) IFL (Interface-Less) for VPRN 50 |
| **Delay Measurement** | STAMP (Simple Two-Way Active Measurement Protocol) |
| **Telemetry** | gNMIc → Prometheus → Grafana |

An end-to-end SRv6 transport based on Nokia SR OS routers is spanning from Access/Aggregation using ([7250 IXR](https://www.nokia.com/networks/ip-networks/7250-interconnect-router/) Gen2/2c) to Edge/Core using ([7750 SR](https://www.nokia.com/networks/ip-networks/7750-service-router/), [FP4/FP5-based](https://www.nokia.com/networks/technologies/fp-network-processor-technology/)):

![wan_nodes drawio](https://github.com/sros-labs/srv6-flexalgo-telemetry-lab/assets/12113139/943a1061-fb6c-4263-9717-9e602507dc20)

### 🎓 Learning Objectives

After completing this lab, you will understand:

1. ✅ SRv6 locator and function concepts (End, End.DT46, End.X)
2. ✅ FlexAlgo configuration for delay-optimized routing
3. ✅ TWAMP-light dynamic delay measurement
4. ✅ Streaming telemetry with gNMI subscriptions

---

## 🧠 Concepts Deep Dive

### 🔷 Understanding SRv6

**Segment Routing over IPv6 (SRv6)** encodes forwarding instructions in IPv6 addresses:

```
SRv6 Address Structure
══════════════════════════════════════════════════════════════════════

         ◄────── Locator (64 bits) ──────►◄──── Function (16 bits) ─►
         ┌────────────────────────────────┬─────────────────────────┐
         │     c128:0db8:0aaa:0101        │       0001 (End)        │
         │                                │       0040 (End.DT46)   │
         │     Identifies the node        │       0041 (End.X)      │
         └────────────────────────────────┴─────────────────────────┘
                      │                                │
                      ▼                                ▼
                "Route to me"                  "What to do here"
```

#### SRv6 Functions Used in This Lab

| Function | Purpose | Example |
|----------|---------|---------|
| **End** | Continue to next segment | Node traversal |
| **End.X** | Exit via specific interface | Adjacency SID |
| **End.DT46** | Decap and lookup in VRF | Service termination |

### 🔷 FlexAlgo Explained

**Flexible Algorithm (FlexAlgo)** allows multiple routing topologies using different metrics:

```
                      ┌───────────────────────────────────────────────┐
                      │          FlexAlgo 128 (Delay Metric)          │
                      │                                               │
             ┌──────► │    R1 ─────── R3 ─────── R5    ← Lowest       │
             │        │     │   5ms    │   3ms    │       Delay       │
             │        │     │          │          │       Path        │
             │        │     ▼          ▼          │                   │
             │        │    R2 ─────── R4 ─────────┘                   │
             │        │        15ms        10ms                       │
             │        └───────────────────────────────────────────────┘
Client1  ────│ OR       
             │        ┌───────────────────────────────────────────────┐
             │        │          Algorithm 0 (IGP Metric)             │
             │        │                                               │
             └──────► │    R1 ─────── R2 ─────── R4 ─────── R5        │
                      │          10         10        10              │
                      │                                               │
                      │    Shortest hop count path                    │
                      └───────────────────────────────────────────────┘
```

#### FlexAlgo Configuration Building Blocks

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. FLEX-ALGO DEFINITION                                            │
│     routing-options {                                               │
│         flexible-algorithm-definitions {                            │
│             flex-algo "flexalgo-128" {                              │
│                 admin-state enable                                  │
│                 metric-type delay    ◄── Use delay instead of IGP   │
│             }                                                       │
│         }                                                           │
│     }                                                               │
├─────────────────────────────────────────────────────────────────────┤
│  2. ISIS PARTICIPATION                                              │
│     isis 0 {                                                        │
│         flexible-algorithms {                                       │
│             flex-algo 128 {                                         │
│                 participate true                                    │
│                 advertise "flexalgo-128"                            │
│             }                                                       │
│         }                                                           │
│     }                                                               │
├─────────────────────────────────────────────────────────────────────┤
│  3. SRV6 LOCATOR WITH ALGORITHM                                     │
│     segment-routing {                                               │
│         segment-routing-v6 {                                        │
│             locator "loc-128" {                                     │
│                 algorithm 128        ◄── Bind to FlexAlgo 128       │
│                 prefix {                                            │
│                     ip-prefix c128:db8:aaa:101::/64                 │
│                 }                                                   │
│             }                                                       │
│         }                                                           │
│     }                                                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🗺️ Network Topology

![Screenshot 2024-03-04 at 12 52 16 PM](https://github.com/sros-labs/srv6-flexalgo-telemetry-lab/assets/12113139/b76b684c-4b13-41a7-bfb9-e61d17e214cd)

### Router Types and Roles

| Node | Chassis Type | Role | SRv6 Locators |
|------|--------------|------|---------------|
| **R1** | 7250 IXR-e2c | PE (Provider Edge) | `c000:db8:aaa:101::/64` (Algo 0)<br>`c128:db8:aaa:101::/64` (Algo 128) |
| **R2** | 7250 IXR-e2 | P (Transit) | `c000:db8:aaa:102::/64` (Algo 0)<br>`c128:db8:aaa:102::/64` (Algo 128) |
| **R3** | 7250 IXR-R6D | P (Transit) | `c000:db8:aaa:103::/64` (Algo 0)<br>`c128:db8:aaa:103::/64` (Algo 128) |
| **R4** | 7750 SR-1 | P (Transit) | `c000:db8:aaa:104::/64` (Algo 0)<br>`c128:db8:aaa:104::/64` (Algo 128) |
| **R5** | 7750 SR-1se | PE (Provider Edge) | `c000:db8:aaa:105::/64` (Algo 0)<br>`c128:db8:aaa:105::/64` (Algo 128) |

---

## ⚙️ SR-SIM Configuration Deep Dive

### 📋 Containerlab Topology Configuration

The lab uses `nokia_srsim` kind for SR OS simulation:

```yaml
# Node Definition for Different Chassis Types
topology:
  kinds:
    nokia_srsim:
      image: registry.srlinux.dev/pub/nokia_srsim:25.10.R2
      license: /opt/nokia/sros/license-25.txt
      env:
        CLAB_SROS_DISABLE_COMPONENT_CONFIG: "xyz"

  nodes:
    # Simple chassis (IXR-e2c)
    R1:
      kind: nokia_srsim
      type: ixr-e2c                              # Chassis type
      mgmt-ipv4: 172.90.90.11
      startup-config: configs/R1/R1.partial.cfg

    # Multi-component chassis (IXR-R6D)
    R3:
      kind: nokia_srsim
      type: ixr-r6d
      components:
        - slot: A                                 # CPM slot
        - slot: 1
          type: cpm-ixr-r6d/iom-ixr-r6d          # Line card type
      env:
        NOKIA_SROS_SFM: cpm-ixr-r6d/iom-ixr-r6d
        NOKIA_SROS_MDA_1: m5-100g-qsfp28          # MDA type
```

### 📋 SRv6 Segment Routing Configuration

```
╔═══════════════════════════════════════════════════════════════════════╗
║                    SRv6 CONFIGURATION HIERARCHY                       ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────────┐  ║
║  │ 1. GLOBAL SRV6 LOCATORS (segment-routing context)               │  ║
║  │                                                                 │  ║
║  │    segment-routing {                                            │  ║
║  │        segment-routing-v6 {                                     │  ║
║  │            locator "loc-0" {              ◄── Algorithm 0       │  ║
║  │                admin-state enable                               │  ║
║  │                block-length 48            ◄── Prefix portion    │  ║
║  │                prefix {                                         │  ║
║  │                    ip-prefix c000:db8:aaa:101::/64              │  ║
║  │                }                                                │  ║
║  │            }                                                    │  ║
║  │            locator "loc-128" {            ◄── FlexAlgo 128      │  ║
║  │                admin-state enable                               │  ║
║  │                block-length 48                                  │  ║
║  │                algorithm 128              ◄── Bound to FlexAlgo │  ║
║  │                prefix {                                         │  ║
║  │                    ip-prefix c128:db8:aaa:101::/64              │  ║
║  │                }                                                │  ║
║  │            }                                                    │  ║
║  │        }                                                        │  ║
║  │    }                                                            │  ║
║  └─────────────────────────────────────────────────────────────────┘  ║
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────────┐  ║
║  │ 2. ISIS SRV6 ADVERTISEMENT                                      │  ║
║  │                                                                 │  ║
║  │    isis 0 {                                                     │  ║
║  │        segment-routing-v6 {                                     │  ║
║  │            admin-state enable                                   │  ║
║  │            locator "loc-0" {                                    │  ║
║  │                level-capability 2         ◄── Advertise in L2   │  ║
║  │            }                                                    │  ║
║  │            locator "loc-128" {                                  │  ║
║  │                level-capability 2                               │  ║
║  │            }                                                    │  ║
║  │        }                                                        │  ║
║  │    }                                                            │  ║
║  └─────────────────────────────────────────────────────────────────┘  ║
║                                                                       ║
║  ┌─────────────────────────────────────────────────────────────────┐  ║
║  │ 3. SRV6 FUNCTIONS FOR BASE ROUTING                              │  ║
║  │                                                                 │  ║
║  │    base-routing-instance {                                      │  ║
║  │        locator "loc-0" {                                        │  ║
║  │            function {                                           │  ║
║  │                end 1 {                    ◄── Node SID          │  ║
║  │                    srh-mode usp           ◄── Ultimate Segment  │  ║
║  │                }                               Pop              │  ║
║  │                end-x-auto-allocate usp    ◄── Adjacency SIDs    │  ║
║  │                    protection protected                         │  ║
║  │            }                                                    │  ║
║  │        }                                                        │  ║
║  │    }                                                            │  ║
║  └─────────────────────────────────────────────────────────────────┘  ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### 📋 VPRN Service with SRv6 FlexAlgo

```
╔═══════════════════════════════════════════════════════════════════════╗
║                    VPRN 50 - FlexAlgo 128 Service                     ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║    service {                                                          ║
║        vprn "50" {                                                    ║
║            admin-state enable                                         ║
║            description "vprn_using_flex-algo_128_srv6"                ║
║            customer "1"                                               ║
║                                                                       ║
║            ┌───────────────────────────────────────────────────────┐  ║
║            │  SRv6 Instance - Binds service to FlexAlgo locator    │  ║
║            │                                                       │  ║
║            │  segment-routing-v6 1 {                               │  ║
║            │      locator "loc-128" {    ◄── Uses FlexAlgo 128     │  ║
║            │          function {                                   │  ║
║            │              end-dt46 { }   ◄── Decap & VRF lookup    │  ║
║            │          }                                            │  ║
║            │      }                                                │  ║
║            │  }                                                    │  ║
║            └───────────────────────────────────────────────────────┘  ║
║                                                                       ║
║            ┌───────────────────────────────────────────────────────┐  ║
║            │  BGP-EVPN Control Plane                               │  ║
║            │                                                       │  ║
║            │  bgp-evpn {                                           │  ║
║            │      segment-routing-v6 1 {                           │  ║
║            │          admin-state enable                           │  ║
║            │          route-distinguisher "1.1.1.1:50"             │  ║
║            │          vrf-target {                                 │  ║
║            │              community "target:100:50"                │  ║
║            │          }                                            │  ║
║            │          vrf-import {                                 │  ║
║            │              policy ["vprn50"]  ◄── Applies FlexAlgo  │  ║
║            │          }                                            │  ║
║            │          srv6 {                                       │  ║
║            │              instance 1                               │  ║
║            │              default-locator "loc-128"                │  ║
║            │          }                                            │  ║
║            │      }                                                │  ║
║            │  }                                                    │  ║
║            └───────────────────────────────────────────────────────┘  ║
║                                                                       ║
║            ┌───────────────────────────────────────────────────────┐  ║
║            │  Customer Interface                                   │  ║
║            │                                                       │  ║
║            │  interface "to-Client1" {                             │  ║
║            │      ipv4 {                                           │  ║
║            │          primary {                                    │  ║
║            │              address 172.17.11.1                      │  ║
║            │              prefix-length 30                         │  ║
║            │          }                                            │  ║
║            │      }                                                │  ║
║            │      sap 1/1/c3/1 { }        ◄── Service Access Point │  ║
║            │  }                                                    │  ║
║            └───────────────────────────────────────────────────────┘  ║
║        }                                                              ║
║    }                                                                  ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### 📋 TWAMP-Light Delay Measurement

```
╔════════════════════════════════════════════════════════════════════════╗
║                    DYNAMIC DELAY MEASUREMENT                           ║
╠════════════════════════════════════════════════════════════════════════╣
║                                                                        ║
║  ┌──────────────────────────────────────────────────────────────────┐  ║
║  │ Interface Delay Configuration                                    │  ║
║  │                                                                  │  ║
║  │    interface "to-R3" {                                           │  ║
║  │        if-attribute {                                            │  ║
║  │            delay {                                               │  ║
║  │                delay-selection dynamic  ◄── Use measured delay   │  ║
║  │                dynamic {                                         │  ║
║  │                    measurement-template "standard-direct"        │  ║
║  │                    twamp-light {                                 │  ║
║  │                        ipv4 {                                    │  ║
║  │                            admin-state enable                    │  ║
║  │                        }                                         │  ║
║  │                    }                                             │  ║
║  │                }                                                 │  ║
║  │            }                                                     │  ║
║  │        }                                                         │  ║
║  │    }                                                             │  ║
║  └──────────────────────────────────────────────────────────────────┘  ║
║                                                                        ║
║  ┌──────────────────────────────────────────────────────────────────┐  ║
║  │ TWAMP-Light Reflector (responds to probes)                       │  ║
║  │                                                                  │  ║
║  │    twamp-light {                                                 │  ║
║  │        reflector {                                               │  ║
║  │            admin-state enable                                    │  ║
║  │            udp-port 862              ◄── Standard TWAMP port     │  ║
║  │            prefix 0.0.0.0/0 { }      ◄── Accept from any source  │  ║
║  │        }                                                         │  ║
║  │    }                                                             │  ║
║  └──────────────────────────────────────────────────────────────────┘  ║
║                                                                        ║
║  ┌──────────────────────────────────────────────────────────────────┐  ║
║  │ Measurement Template                                             │  ║
║  │                                                                  │  ║
║  │    test-oam {                                                    │  ║
║  │        link-measurement {                                        │  ║
║  │            measurement-template "standard-direct" {              │  ║
║  │                admin-state enable                                │  ║
║  │                aggregate-sample-window {                         │  ║
║  │                    multiplier 5         ◄── 5 samples per window │  ║
║  │                    window-integrity 1                            │  ║
║  │                    threshold {                                   │  ║
║  │                        relative 50      ◄── % change threshold   │  ║
║  │                        absolute 50      ◄── Absolute threshold   │  ║
║  │                    }                                             │  ║
║  │                }                                                 │  ║
║  │            }                                                     │  ║
║  │        }                                                         │  ║
║  │    }                                                             │  ║
║  └──────────────────────────────────────────────────────────────────┘  ║
╚════════════════════════════════════════════════════════════════════════╝
```

### 📋 Policy for FlexAlgo Route Import

```
╔═══════════════════════════════════════════════════════════════════════╗
║                    FLEXALGO IMPORT POLICY                             ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║    policy-options {                                                   ║
║        community "vprn50" {                                           ║
║            member "target:100:50" { }                                 ║
║        }                                                              ║
║                                                                       ║
║        policy-statement "vprn50" {                                    ║
║            entry 10 {                                                 ║
║                from {                                                 ║
║                    community {                                        ║
║                        name "vprn50"        ◄── Match RT community    ║
║                    }                                                  ║
║                }                                                      ║
║                action {                                               ║
║                    action-type accept                                 ║
║                    flex-algo 128            ◄── Force FlexAlgo 128    ║
║                }                              resolution for routes   ║
║            }                                                          ║
║        }                                                              ║
║    }                                                                  ║
║                                                                       ║
║    This ensures all VPRN 50 routes use the delay-optimized path!      ║
╚═══════════════════════════════════════════════════════════════════════╝
```

---

## 🚀 Quick Start

### Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Docker | Latest | Container runtime |
| containerlab | ≥ 0.71 | Lab orchestration |
| Nokia SR-SIM image | 25.7.R1+ | SR OS simulation |
| License file | Valid | Required for SR-SIM |

### Option 1: Cloud-Based (Recommended for Quick Start)

Click one of the badges at the top to launch in:
- **GitHub Codespaces** - Zero local setup required
- **DevPod** - Portable dev environment

### Option 2: Local Deployment

```bash
# Clone the repository
git clone https://github.com/sros-labs/srv6-flexalgo-telemetry-lab.git
cd srv6-flexalgo-telemetry-lab

# Deploy the lab
clab deploy --reconfigure

# Wait for all nodes to boot (approximately 2 minutes for SR-SIM)
watch -n 5 'docker ps --format "table {{.Names}}\t{{.Status}}"'
```

### Verify Deployment

```bash
# Check all containers are running
docker ps

# Connect to routers
ssh admin@R1    # Password: admin
ssh admin@R5    # Password: admin

# Access telemetry
open http://localhost:3000   # Grafana (admin/admin)
open http://localhost:9090   # Prometheus
```

---

## 🔬 Checking the Lab

### Check 1: Explore SRv6 Locators

```
A:admin@R1# show router segment-routing srv6 locator 

=========================================================================================
Segment Routing v6 Locators
=========================================================================================
Name                              Admin   Operational  Function    Prefix
                                  State   State        Count
-----------------------------------------------------------------------------------------
loc-0                             Up      Up           3           c000:db8:aaa:101::/64
loc-128                           Up      Up           3           c128:db8:aaa:101::/64
-----------------------------------------------------------------------------------------
No. of SRv6 Locators: 2
```

**🎯 What to observe:**
- Two locators: one for Algo 0, one for FlexAlgo 128
- Different prefix schemes (c000 vs c128)

### Check 2: Verify FlexAlgo Participation

```
A:admin@R1# show router isis flexible-algorithms 

===============================================================================
ISIS Flexible Algorithms
===============================================================================
Flex-Algo   Metric Type    Advertised FAD         
-------------------------------------------------------------------------------
128         delay          flexalgo-128           
-------------------------------------------------------------------------------
No. of Flex-Algos: 1
```

### Check 3: Verify ISIS SRv6 Endpoints

```
A:admin@R1# show router isis srv6-endpoint 

===============================================================================
SRv6 Endpoints
===============================================================================
SID                                          Endpoint  Locator    
-------------------------------------------------------------------------------
c000:db8:aaa:101::1                          End       loc-0      
c000:db8:aaa:101::40                         End.DT46  loc-0      
c128:db8:aaa:101::1                          End       loc-128    
c128:db8:aaa:101::40                         End.DT46  loc-128    
-------------------------------------------------------------------------------
No. of SRv6 Endpoints: 4
```

### Check 4: View FlexAlgo Path Calculation

```
A:admin@R1# show router isis flex-algo 128 spf-log 

===============================================================================
ISIS Flex-Algo 128 SPF Log
===============================================================================
When                  Duration     Reason
-------------------------------------------------------------------------------
2024-03-04 12:30:15   2ms          Link delay change
2024-03-04 12:25:10   3ms          Initial calculation
-------------------------------------------------------------------------------
```

### Check 5: Verify VPRN Service

```
A:admin@R1# show service id 50 base 

===============================================================================
Service Basic Information
===============================================================================
Service Id        : 50
Service Type      : VPRN
Customer Id       : 1
Description       : vprn_using_flex-algo_128_srv6
Admin State       : Up
Oper State        : Up

===============================================================================
SRv6 Information
===============================================================================
Locator           : loc-128
Function          : End.DT46
```

### Check 6: Verify Link Delays

```
A:admin@R1# show router interface "to-R3" delay 

===============================================================================
Interface Delay Information
===============================================================================
Interface         : to-R3
Delay Selection   : dynamic
Current Delay     : 8 microseconds
Last Update       : 2024-03-04 12:35:00

TWAMP-Light Status: Active
Measurement Template: standard-direct
```

### Check 7: Trace Traffic Path

```
A:admin@R1# traceroute 50.0.0.5 source 50.0.0.1 

traceroute to 50.0.0.5, 30 hops max, 60 byte packets
 1  192.168.13.2 (R3)  2.103 ms
 2  192.168.35.2 (R5)  4.215 ms
 3  50.0.0.5           5.891 ms

# Note: Path goes via R3 (lower delay) instead of R2-R4 (higher delay)
```

---

## 📊 Telemetry Stack

Nowadays, observability is becoming essential for every organisation.
An open source GPG ([gnmic](https://gnmic.openconfig.net/)/[prometheus](https://prometheus.io/)/[grafana](https://grafana.com/)) telemetry stack is used to collect and report all the objects of interest via Telemetry/gRPC (links delay, interfaces state, metrics, cpu, mem, etc.).

gnmic is then using prometheus TSDB as output for storing the metrics which can then be fetched by Grafana for monitoring (PromQL).

```
╔═══════════════════════════════════════════════════════════════════════╗
║                    TELEMETRY DATA FLOW                                ║
╠═══════════════════════════════════════════════════════════════════════╣
║                                                                       ║
║                                    ┌─────────┐                        ║
║                                    │         │                        ║
║                                    │         │                        ║
║    ┌─────────┐   gNMI Subscribe    │  gNMIc  │                        ║
║    │  R1-R5  │ ─────────────────►  │         │                        ║
║    └─────────┘                     │  :9804  │                        ║
║                                    │         │                        ║
║                                    │         │                        ║
║                                    └────┬────┘                        ║
║                                         │                             ║
║                                    Prometheus                         ║
║                                    Remote Write                       ║
║                                         │                             ║
║                                         ▼                             ║
║                                  ┌─────────────┐                      ║
║                                  │ Prometheus  │                      ║
║                                  │   :9090     │                      ║
║                                  └──────┬──────┘                      ║
║                                         │                             ║
║                                    PromQL                             ║
║                                    Queries                            ║
║                                         │                             ║
║                                         ▼                             ║
║                                  ┌─────────────┐                      ║
║                                  │   Grafana   │                      ║
║                                  │   :3000     │                      ║
║                                  └─────────────┘                      ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### Collected Telemetry Paths

```yaml
# gNMIc Configuration Excerpt
subscriptions:
  - /state/router[router-name=Base]/interface[interface-name=*]/statistics
  - /state/router[router-name=Base]/isis[isis-instance=0]/statistics
  - /state/system/cpu[sample-period=*]
  - /state/system/memory-pools
  - /state/test-oam/link-measurement
```

### Grafana Dashboards

| Dashboard | Metrics Displayed |
|-----------|------------------|
| **Interface Statistics** | Throughput, errors, discards per interface |
| **Link Delay** | TWAMP-measured delay per link |
| **BGP Overview** | Peers, routes, states |
| **System Resources** | CPU, memory utilization |
| **Flow Visualization** | Traffic flow through the network |

Using Grafana dashboard, it is possible to get direct correlation between the sum of TWAMP delay measurement on individual links and the IPv6 route table as shown below:

![Screenshot 2024-03-04 at 1 03 17 PM](https://github.com/sros-labs/srv6-flexalgo-telemetry-lab/assets/12113139/36074d70-ab1a-419c-9584-15aa651eea39)

### Access Telemetry

```bash
# Grafana UI
http://localhost:3000
# Credentials: admin/admin

# Prometheus UI
http://localhost:9090

# gNMIc metrics endpoint
http://localhost:9804/metrics
```

---

## 🔧 Traffic Generation & Delay Manipulation

### Start Traffic

```bash
# Generate 2Mbps UDP traffic from Client1 to Client2
./start_traffic.sh

# Or manually from Client1
docker exec -it client1 iperf3 -c 172.17.44.2 -u -b 2M -t 3600
```

### Stop Traffic

```bash
./stop_traffic.sh

# Or manually
docker exec -it client1 pkill iperf3
```

### Manipulate Link Delay (Force Path Change!)

```bash
# Add 100ms delay to R1-R2 link (makes R1-R3 path preferred)
containerlab tools netem set -n R1 -i eth1 --delay 100ms

# Add delay to R1-R3 link (may shift traffic to R2)
containerlab tools netem set -n R1 -i eth2 --delay 200ms

# Remove delay
containerlab tools netem set -n R1 -i eth1 --delay 0ms
```

### 🎯 Experiment: Watch FlexAlgo Adapt

1. **Before:** Check the path in Grafana Flow dashboard
2. **Add delay:** `containerlab tools netem set -n R1 -i eth2 --delay 100ms`
3. **Wait:** 30-60 seconds for TWAMP to measure new delay
4. **Observe:** FlexAlgo recalculates and traffic shifts!
5. **Verify:** Check ISIS SPF log and path trace

---

## 📚 References

### Nokia Documentation
- [Nokia SR OS YANG Models](https://github.com/nokia/7x50_YangModels)
- [containerlab sr-sim](https://containerlab.dev/manual/kinds/sros/)
- [Nokia SRv6 Documentation](https://www.nokia.com/networks/ip-networks/segment-routing/)

### Standards
- [RFC 8986 - SRv6 Network Programming](https://datatracker.ietf.org/doc/html/rfc8986)
- [RFC 9350 - IGP Flexible Algorithm](https://datatracker.ietf.org/doc/html/rfc9350)
- [RFC 8762 - STAMP](https://datatracker.ietf.org/doc/html/rfc8762)

### Tools
- [containerlab](https://containerlab.dev/)
- [gNMIc](https://gnmic.openconfig.net/)
- [Prometheus](https://prometheus.io/)
- [Grafana](https://grafana.com/)

---

## 🧹 Lab Cleanup

```bash
# Destroy the lab and remove all resources
clab destroy --cleanup
```

---

## 📝 License

This lab is provided for educational purposes. Nokia SR OS requires a valid license.

---

<div align="center">

**Built with ❤️ for the Network Community**

[![Star History](https://img.shields.io/github/stars/sros-labs/srv6-flexalgo-telemetry-lab?style=social)](https://github.com/sros-labs/srv6-flexalgo-telemetry-lab)

</div>


