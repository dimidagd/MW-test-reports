# Middleware 3.1 test report



## Report content
- [General Test Setup](#test-setup)
	+ [Middleware build](#mw-build)
	+ [Application builds](#app-builds)
	+ [Application arguments](#app-args)
	+ [Simulation Application](#simulation-app)
- [Specific Test Layout](#test-specific-application-layout)
- [Findings](#findings)
	+ [Timeout warning](#timeout-warning)
- [Attachments](#attachments)


## Test Setup

```
Threadripper-PC
│-- Docker network namescapes
│   └── Network 1 @ 127.200.0.0/24
│      
│
│-- MW config file 
│
├─<CPU 0-2> MW containers
│   ├── MW0 @ Network1 / 127.200.0.2
│   ├── Mw1 @ Network1 / 127.200.0.3
│   └── MW2 @ Network1 / 127.200.0.4
├─<CPU 3-7> Wireshark containers
│   ├── Wire0 listening on container:MW0@eth0 -> output file $HOME/tshark/Node0.pcapng
│   ├── Wire1 listening on container:MW1@eth0 -> output file $HOME/tshark/Node1.pcapng
│   └── Wire2 listening on container:MW2@eth0 -> output file $HOME/tshark/Node2.pcapng
└─<CPU 3-7> Application containers
    ├── @container:MW2
    │   ├─ ✓ MWS (Simulator)
    │   ├─ ✓ ANS
    │   ├─ ✓ SHP
    │   ├─ ✓ RUT
    │   ├─ ✓ HMI
    │   ├─ ✓ HMJ
    │   ├─ ✓ ACS
    │   └─ ✓ SFU
    ├── @container:MW1
    │   └─ ✓ SAS
    └── @container:MW0
        └─ ✓ TST (test app)

```

## MW build

The image used for MW is `autdtu/mw:3.1-exp` . This image has compiled MW3.1 with STDOUT disabled. We have observed that for time-critical tasks, disabling STDOUT saves up execution time and the MW is more stable. The configuration file for the `test_failmode` test can be found in `integration_tests/test_failmode/config.yaml`

## App builds
For the application builds you should look into the commits of the repos in `integration_tests/modules` . The apps are built using the `autdtu/app:latest` image as cache.

## App args

The input arguments to the applications can be found in the test compose file, for example in
`integration_tests/test_failmode/docker-compose.yml`

## Simulation App

The MWS application is a simulator that mimicks several other necessary applications for the test, the simulator extrapolates the detections for each mimicked app by using the position of the targets and the ownships position.

 - XBR (X-band radar detections)
 - COS (RGB object server detections)
 - IOS (Infrared object server detections)
 - NAD (navigational data)
 - AIS

# Test Specific Application Layout
The actual test that was used in order to demonstrate the following defect is the following one.

```
Threadripper-PC
│-- Docker network namescapes
│   └── Network 1 @ 127.200.0.0/24
│      
│
│-- MW config file 
│
├─<CPU 0-2> MW containers
│   ├── MW0 @ Network1 / 127.200.0.2
│   ├── Mw1 @ Network1 / 127.200.0.3
│   └── MW2 @ Network1 / 127.200.0.4
├─<CPU 3-7> Wireshark containers
│   ├── Wire0 listening on container:MW0@eth0 -> output file $HOME/tshark/Node0.pcapng
│   ├── Wire1 listening on container:MW1@eth0 -> output file $HOME/tshark/Node1.pcapng
│   └── Wire2 listening on container:MW2@eth0 -> output file $HOME/tshark/Node2.pcapng
└─<CPU 3-7> Application containers
    ├── @container:MW2
    │   ├─ x MWS (Simulator)
    │   ├─ ✓ ANS
    │   ├─ x SHP
    │   ├─ x RUT
    │   ├─ ✓ HMI
    │   ├─ x HMJ
    │   ├─ x ACS
    │   └─ x SFU
    ├── @container:MW1
    │   └─ ✓ SAS
    └── @container:MW0
        └─ ✓ TST (test app)

```

The **TST** application is subscribing on all the topics from the APPS marked with **x** above, but those applications are not getting started.

## Findings
### Timeout warning
The fact that most of teh applications do not exist leads naturally to the following warnings

```
2022-11-23 11:33:21.709000114|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C13 isn't connected or doesn't exist in the local configuration
2022-11-23 11:33:21.709090381|DEBUG|Node1.MLW| packet_manager/packetmanager.cpp.866: network: register_subscriber: SFU.C13
2022-11-23 11:33:21.709865024|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C14 isn't connected or doesn't exist in the local configuration
2022-11-23 11:33:21.709932823|DEBUG|Node1.MLW| packet_manager/packetmanager.cpp.866: network: register_subscriber: SFU.C14
2022-11-23 11:33:21.710756564|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C15 isn't connected or doesn't exist in the local configuration
2022-11-23 11:33:21.710823379|DEBUG|Node1.MLW| packet_manager/packetmanager.cpp.866: network: register_subscriber: SFU.C15
2022-11-23 11:33:21.711654307|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C16 isn't connected or doesn't exist in the local configuration

```
The unexpected part is the unstable behaviour appearing after a while on **Node2**:

-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=UNSTABLE NODE LOGS-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
#### MW logs Node2

```
2022-11-23 11:32:22.163374797|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 5782
2022-11-23 11:32:22.163457466|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 5780
2022-11-23 11:32:22.163530260|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 5777
2022-11-23 11:32:22.163603487|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 5773
2022-11-23 11:32:22.163680449|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 5768
2022-11-23 11:32:22.163773622|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 5762
2022-11-23 11:32:22.163863283|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 5778
2022-11-23 11:32:22.163934674|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 5770
2022-11-23 11:32:22.164009838|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 5761
2022-11-23 11:32:22.164229704|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 5774
2022-11-23 11:32:22.164327399|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 5763
2022-11-23 11:32:22.194349585|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 5783
2022-11-23 11:32:22.194405324|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 5782
2022-11-23 11:32:22.194436748|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 5780
2022-11-23 11:32:31.621542949|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 6275
2022-11-23 11:32:31.621651595|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 6274
2022-11-23 11:32:33.434185684|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 6371
2022-11-23 11:32:33.434255761|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 6370
2022-11-23 11:32:33.434303149|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 6368
2022-11-23 11:32:33.434347163|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 6365
2022-11-23 11:32:38.261466885|WARN |Node2.MLW| packet_manager/resendList.cpp.86: Packet acknowledge timeout from node 1 with sequence number 6627

```
-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-END OF UNSTABLE NODE LOGS =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
#### MW logs Node1
```
2022-11-23 11:32:22.141574591|DEBUG|Node1.MLW| packet_manager/packetmanager.cpp.866: network: register_subscriber: SFU.C02
2022-11-23 11:32:22.142709971|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C03 isn't connected or doesn't exist in the local configuration
2022-11-23 11:32:22.142908422|DEBUG|Node1.MLW| packet_manager/packetmanager.cpp.866: network: register_subscriber: SFU.C03
2022-11-23 11:32:22.144113547|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C04 isn't connected or doesn't exist in the local configuration
2022-11-23 11:32:22.144219705|DEBUG|Node1.MLW| packet_manager/packetmanager.cpp.866: network: register_subscriber: SFU.C04
2022-11-23 11:32:22.145394791|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C05 isn't connected or doesn't exist in the local configuration
2022-11-23 11:32:22.145496488|DEBUG|Node1.MLW| packet_manager/packetmanager.cpp.866: network: register_subscriber: SFU.C05
2022-11-23 11:32:22.146633959|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C06 isn't connected or doesn't exist in the local configuration
2022-11-23 11:32:22.146745731|DEBUG|Node1.MLW| packet_manager/packetmanager.cpp.866: network: register_subscriber: SFU.C06
2022-11-23 11:32:22.147939657|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C07 isn't connected or doesn't exist in the local configuration
2022-11-23 11:32:22.148050361|DEBUG|Node1.MLW| packet_manager/packetmanager.cpp.866: network: register_subscriber: SFU.C07
2022-11-23 11:32:22.149882249|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C08 isn't connected or doesn't exist in the local configuration
2022-11-23 11:32:22.149996745|DEBUG|Node1.MLW| packet_manager/packetmanager.cpp.866: network: register_subscriber: SFU.C08
2022-11-23 11:32:22.171908348|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C09 isn't connected or doesn't exist in the local configuration
2022-11-23 11:32:22.171988548|DEBUG|Node1.MLW| packet_manager/packetmanager.cpp.866: network: register_subscriber: SFU.C09
2022-11-23 11:32:22.172726915|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C10 isn't connected or doesn't exist in the local configuration
2022-11-23 11:32:22.172788956|DEBUG|Node1.MLW| packet_manager/packetmanager.cpp.866: network: register_subscriber: SFU.C10
2022-11-23 11:32:22.173463212|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C11 isn't connected or doesn't exist in the local configuration
2022-11-23 11:32:22.173525180|DEBUG|Node1.MLW| packet_manager/packetmanager.cpp.866: network: register_subscriber: SFU.C11
2022-11-23 11:32:22.174217324|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C12 isn't connected or doesn't exist in the local configuration
2022-11-23 11:32:22.174279677|DEBUG|Node1.MLW| packet_manager/packetmanager.cpp.866: network: register_subscriber: SFU.C12
2022-11-23 11:32:22.174960279|WARN |Node1.MLW| packet_manager/packetmanager.cpp.847: App: SFU.C13 isn't connected or doesn't exist in the local configuration
```
## Attachments

In the report you can find attached a tarfile *(local_setup_logs_2022-11-23_12-43-33.tar.gz)* that includes.

1. The Middleware logfiles from all nodes
2. Wireshark captured packets for all Nodes.
3. CPU usage logs
