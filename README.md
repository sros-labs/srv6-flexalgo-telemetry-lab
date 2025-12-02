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

Objective: Creation of a traffic-engineered path based on SRv6 transport between 2 endpoints (R1 and R5) using delay as a metric to provide lowest latency connectivity between 2 clients over a L3VPN.
* Transport: Base [SRv6](https://www.nokia.com/networks/ip-networks/segment-routing/) (end-dt46) and FlexAlgo 128 (with STAMP dynamic delay measurement)
* Service: [EVPN](https://www.nokia.com/networks/ethernet-vpn/) IFL (Interface-less)

The purpose of this pre-configured lab is to demonstrate the use of an end-to-end SRv6 transport on Nokia SR OS routers spanning from Access/Aggregation ([7250 IXR](https://www.nokia.com/networks/ip-networks/7250-interconnect-router/) Gen2/2c) to Edge/Core ([7750 SR](https://www.nokia.com/networks/ip-networks/7750-service-router/), [FP4/FP5-based](https://www.nokia.com/networks/technologies/fp-network-processor-technology/)):

![wan_nodes drawio](https://github.com/thcorre/SRv6-FlexAlgo-Telemetry-Lab-with-Nokia-SROS/assets/12113139/943a1061-fb6c-4263-9717-9e602507dc20)


This relies on usage of a Flex-Algorithm (Algo 128) with delay used as metric to achieve the lowest latency path.
The Flex-Algorithm for SRv6-based VPRNs feature allows the computation of constraint-based paths across an SRv6-enabled network, based on metrics other than the default IGP metrics. This allows carrying data traffic over an end-to-end path that is optimized using the best suited metric IGP, delay, or TE).

Nowadays, observability is becoming essential for every organisation.
An open source GPG ([gnmic](https://gnmic.openconfig.net/)/[prometheus](https://prometheus.io/)/[grafana](https://grafana.com/)) telemetry stack is used to collect and report all the objects of interest via Telemetry/gRPC (links delay, interfaces state, metrics, cpu, mem, etc.):

![Screenshot 2024-03-04 at 12 53 12 PM](https://github.com/thcorre/SRv6-FlexAlgo-Telemetry-Lab-with-Nokia-SROS/assets/12113139/cafa2ed8-b933-4e48-9b67-b8001b72ae17)

gnmic is collecting streaming telemetry data (push-based approach from devices) from all routers with a 5s sampling interval via subscription to certain paths of interest:

      - /state/router[router-name=Base]/interface[interface-name=*]/statistics
      - /state/router[router-name=*]/interface[interface-name=*]/oper-state
      - /state/service/vprn[service-name=50]/interface[interface-name=*]/oper-state
      - /state/service/vprn[service-name=50]/interface[interface-name=*]/statistics
      - /state/test-oam/link-measurement/router[router-instance=Base]/interface[interface-name=*]/last-reported-delay
      - /state/system/cpu[sample-period=60]/summary/usage/
      - /state/system/memory-pools/summary/
      - /state/router[router-name=Base]/route-table/unicast/ipv6
      - /state/router[router-name=Base]/bgp/statistics/peers
      - /state/router[router-name=Base]/bgp/statistics/routes-per-family/vpn-ipv4/remote-active-routes
      - /state/router[router-name=Base]/bgp/statistics/routes-per-family/vpn-ipv6/remote-active-routes

Note: All Nokia SR OS YANG models are publicly available on: [https://github.com/nokia/7x50_YangModels](https://github.com/nokia/7x50_YangModels).

gnmic is then using prometheus TSDB as output for storing the metrics which can then be fetched by Grafana for monitoring (PromQL).

Grafana dashboards are provided to check:
* The state of the interfaces for each node
* The latency on the links in "real" time (delay measurement interval via STAMP)
* The number of BGP peers/routes per node
* The CPU/memory per node

## Network Topology

![Screenshot 2024-03-04 at 12 52 16 PM](https://github.com/thcorre/SRv6-FlexAlgo-Telemetry-Lab-with-Nokia-SROS/assets/12113139/b76b684c-4b13-41a7-bfb9-e61d17e214cd)

All routers are pre-configured - startup configuration can be found in ‘config/Rx/Rx.cfg’.

Each router has 2 locators:
- Locator ‘c000:db8:aaa:10n::/64’ in ISIS Algo 0
- Locator ‘c128:db8:aaa:10n::/64’ in ISIS Algo 128 (used by VPRN 50) where n is Node-ID, so 1 is R1, 5 is R5

R1 and R5 are ready to send/receive customer traffic through VPRN 50 (locator ‘c128:db8:aaa:10n::/64’).

The protocol used to dynamically measure link delay is the light version of the Two-Way Active Measurement Protocol (TWAMP-Light), defined in IETF RFC 5357. The corresponding configuration on interface "to-R3" of router R1 is shown below:
```
interface "to-R3" {
-- Snip --
      if-attribute {
            delay {
                  delay-selection dynamic
                  dynamic {
                        measurement-template "standard-direct"
                        twamp-light {
                            ipv4 {
                                admin-state enable
                            }
                        }
                  }
            }
      }
}
```

Using Grafana dashboard, it is possible to get direct correlation between the sum of TWAMP delay measurement on individual links and the IPv6 route table as shown below:

![Screenshot 2024-03-04 at 1 03 17 PM](https://github.com/thcorre/SRv6-FlexAlgo-Telemetry-Lab-with-Nokia-SROS/assets/12113139/36074d70-ab1a-419c-9584-15aa651eea39)

## Requirements

To deploy this lab you need:
1. a server with Docker and [containerlab](https://containerlab.dev/) (0.69.3 version used at time of writing, a more recent release of clab to support SR-SIM container).
2. [nokia_sros SR-SIM](https://containerlab.dev/manual/kinds/vr-sros/) 25.7.R1+ image (25.7.R1 being the initial version introducing SR-SIM), and a valid license file.

## Clone the lab on your server
To deploy this lab, you must clone it to your server with git.

```bash
# change to a working directory of your choice and clone the lab
git clone https://github.com/thcorre/SRv6-FlexAlgo-Telemetry-Lab-with-Nokia-SROS.git
```

## Deploying the lab
The lab is deployed with the [containerlab](https://containerlab.dev/) project, where [`srv6-flexalgo.clab.yml`](srv6-flexalgo.clab.yml) file declaratively describes the lab topology.
```
clab deploy --reconfigure
```
To remove the lab:
```
clab destroy --cleanup
```

## Accessing the network elements and telemetry stack
Once the lab has been deployed, the different SR Linux nodes can be accessed via SSH through their management IP address, given in the summary displayed after the execution of the deploy command. It is also possible to reach those nodes directly via their hostname, defined in the topology file. Linux clients cannot be reached via SSH, as it is not enabled, but it is possible to connect to them with a docker exec command.

```bash
# reach a SR OS IP/MPLS router via SSH
ssh admin@R1
ssh admin@R5-a

# reach a Linux client via Docker
docker exec -it client1 bash
```

If you are accessing from a remote host, then replace localhost by the CLAB Server IP address:
* Grafana: http://localhost:3000. Built-in user credentials: admin/admin
* Prometheus: http://localhost:9090/graph

## Launching traffic and modifying delay on links
One Linux client (Client1) is sending unidirectional traffic to another client (Client2) through a L3VPN (EVPN IFL).

2Mbps UDP traffic can be launched from Client1 to Client2 via [`start_traffic.sh`](start_traffic.sh) script in main directory. Traffic can be stopped via [`stop_traffic.sh`](stop_traffic.sh).

A fine-grained control on links delay can be achieved via [NetEm](https://www.linux.org/docs/man8/tc-netem.html) commands applied at host level or directly through containerlab tool command (since release [0.44](https://containerlab.dev/rn/0.44/)) to influence the lowest latency path:
```bash
# Add 100ms latency on eth2 interface for node R1
containerlab tools netem set -n R1 -i eth2 --delay 100ms
```

