<!--
SPDX-License-Identifier: Apache-2.0
Copyright 2020 Intel Corporation
-->

# **Cloud Native Data Plane (CNDP)**

Cloud Native Data Plane (CNDP) is a collection of user space libraries for accelerating packet processing for cloud applications. It aims to provide better performance than that of standard network socket interfaces by using an I/O layer primarily built on AF_XDP, an interface that delivers packets directly to user space, bypassing the kernel networking stack. For more details refer https://cndp.io/

CNDP BESS port enables sending/receiving packets to/from network interface using AF-XDP.  CNDP integration with OMEC BESS UPF enables a software-based datapath and also provides deployment flexibility for kubernetes based deployments where CNDP uses [AF-XDP device plugin](https://github.com/intel/afxdp-plugins-for-kubernetes).

Following are the steps required to build and test CNDP BESS UPF docker image:

### Step 1: Build the OMEC UPF docker container

> Note: If you are behind a proxy make sure to export/setenv http_proxy and https_proxy

From the top level directory call:

```
$ make docker-build
```

### Step 2: Test setup

Following diagram shows is a test setup. 

![Test setup] (./images/cndp-omec-upf-test-setup.jpg)

There are two systems: System 1 is runs CNDP BESS UPF and System 2 runs DPDK based packet generator which simulates traffic generated from multiple UEs and App servers. 

System 1 and System 2 are connected using  physical links. Setup uses two network ports which represents access and core interface. This test setup uses Intel Ethernet 800 series network adapter (hereafter referred to as NIC).  NIC in System 1 has Intel Dynamics Device Personalization (DDP) for telecommunications workload enabled. DDP profile can help in GTPU packet traffic steering to required NIC hardware queues using RSS (Receive Side Scaling). DDP feature works along with XDP offload feature in NIC hardware to redirect GTPU packets directly to user space via AF-XDP sockets. Refer the Deployment section in this [document](https://builders.intel.com/docs/networkbuilders/intel-ethernet-controller-800-series-device-personalization-ddp-for-telecommunications-workloads-technology-guide.pdf) to enable DDP.

Install NIC driver in System 1 from this link :  https://sourceforge.net/projects/e1000/files/ice%20stable/1.9.7/ .  This driver is used instead of in-tree kernel driver since this driver supports tc filter for creating queue groups and RSS for GTPU packet traffic steering. 

The setup uses Physical Function (PF) of PCIe network adaptor (where the driver supports AF-XDP zero copy ) for improved network I/O performance. AF-XDP zero copy support for SR-IOV driver and sub function support using devlink will be supported in future releases.

### Step 3: Run UPF in System 1

1) Setup hugepages in the system 1. For example, to setup 8GB of total huge pages (8 pages each of size 1GB in each NUMA node) using DPDK script, use the command `dpdk-hugepages.py -p 1G --setup 8G`

2) Modify [cndp_upf_1worker.jsonc](./conf/cndp_upf_1worker.jsonc) file lports section to update the access and core netdev interface name and required queue id.

3) Modify the script [docker_setup.sh](./scripts/docker_setup.sh) and update the access and core interface names (s1u, sgi), access/core interface mac addresses and neighbor gateway interfaces mac addresses. In our example test setup, neighbor mac address (n-s1u, n-sgi) corresponds to access/core interfaces used by packet generator in system 2 to send/receive n/w packets. Update following values based on your system configuration.

- ifaces=("enp134s0" "enp136s0")   // (s1u, sgi)
- macaddrs=(40:a6:b7:78:3f:ec 40:a6:b7:78:3f:e8)  // mac address (s1u, sgi)
- nhmacaddrs=(40:a6:b7:78:3f:bc 40:a6:b7:78:3f:b8) //  mac address (n-s1u, n-sgi)

4) Modify the script [docker_setup.sh](./scripts/docker_setup.sh) and update the function `move_ifaces()` in condition `if [ "$mode" == 'cndp' ]` .

Update `start_q_idx` to choose the start queue index to receive n/w packets. This should match the queue id used in lports section of [cndp_upf_1worker.jsonc](./conf/cndp_upf_1worker.jsonc)

5) Modify the script [docker_setup.sh](./scripts/docker_setup.sh) and [upf.json](./conf/upf.json) to set `mode=cndp`

6)  Modify [upf.json](./conf/upf.json) to use appropriate netdev interface and CNDP jsonc config file.

```
"access": {
        "ifname": "enp134s0",
        "cndp_jsonc_file": "/opt/bess/bessctl/conf/cndp_upf_1worker.jsonc"
    },
 "core": {
        "ifname": "enp136s0",
        "cndp_jsonc_file": "/opt/bess/bessctl/conf/cndp_upf_1worker.jsonc"
    },
```
7) Modify the script [cndp_reset_upf.sh](./scripts/cndp_reset_upf.sh) to use appropriate PCIE device address, network interface name and set_irq_affinity script in NIC driver. 

```
ACCESS_PCIE=0000:86:00.0
CORE_PCIE=0000:88:00.0
ACCESS_IFACE=enp134s0
CORE_IFACE=enp136s0
SET_IRQ_AFFINITY=~/nic/driver/ice-1.9.7/scripts/set_irq_affinity
```

This script is used to stop any running containers, disable irqbalance, set irq affinity to all queues for access and core interface. The script also set XDP socket busy poll settings for access and core interfaces. `set_irq_affinity` script used by this script can be found in the NIC driver install path. irq affinity and AF_XDP busypoll settings are done to get improved network I/O performance. These settings are recommended but not mandatory.

8) From the top level directory call:

```
$ ./scripts/cndp_reset_upf.sh
$ ./scripts/docker_setup.sh
```
Insert rules into relevant PDR and FAR tables
```
$ docker exec bess-pfcpiface pfcpiface -config /conf/upf.json -simulate create
```
9) From browser, use localhost:8000 to view the UPF pipeline in GUI. If you are remotely connecting to system via ssh, you need to setup a tunnel with local port forwarding.

10) To stop the containers run following command

```
cndp_reset_upf.sh
```

### Step 4: Run DPDK packet generator in System 2

From system 2, bind the two interfaces used by pktgen to DPDK (used to send n/w pkts to access/core ). Also setup huge pages in the system.

Modify [pktgen_cndp.bess](conf/pktgen_cndp.bess) script as follows.

1) Update source and destination interface mac addresses of the access and core interface  - smac_access, smac_core, dmac_access, dmac_core. Here smac_xxx corresponds to mac address of NIC in system 2 where we run pktgen and dst_xxx corresponds to mac address of NIC in system 1 which runs UPF pipeline.

2) Update worker core ids to use core id in NUMA node where NIC is attached. For example if NIC is attached to NUMA node 1, use worker core ids in NUMA node 1. eg:  `workers=[22, 23, 24, 25]`

3) Bind the NICs to DPDK and note the vfio device number in "/dev/vfio"

From the top level directory call: (**Note**: Update below command to set `cpuset-cpus`range same as worker core ids in step 2 above and use the vfio device number from step 3)

```
docker run --name pktgen -td --restart unless-stopped \
        --cpuset-cpus=22-25 --ulimit memlock=-1 --cap-add IPC_LOCK \
        -v /dev/hugepages:/dev/hugepages -v "$PWD/conf":/opt/bess/bessctl/conf \
        -v /lib/firmware/intel:/lib/firmware/intel \
        --device=/dev/vfio/vfio --device=/dev/vfio/119 --device=/dev/vfio/120 \
        upf-epc-bess:"$(<VERSION)" -grpc-url=0.0.0.0:10514

docker exec -it pktgen ./bessctl run pktgen
```

We can monitor if pktgen is sending packets using the following command:

```
docker exec -it pktgen ./bessctl monitor tc
```

If we need to stop sending packets at some point use the following command:

```
docker exec -it pktgen ./bessctl daemon reset
```

### CNDP OMEC-UPF multiple worker threads setup

Modify OMEC UPF and CNDP configuration files to support multiple worker threads. Each thread will run UPF pipeline in a different core.

To test multiple worker thread, we need to use CNDP jsonc file with appropriate configuration. An example CNDP jsonc file for 4 worker threads is in [cndp_upf_4worker.jsonc](./conf/cndp_upf_4worker.jsonc). Follow below steps to configure OMEC-UPF pipeline using 4 worker threads which runs on 5 cores (4 BESS worker threads and 1 main thread).

1) Update [upf.json](./conf/upf.json) to set number if worker threads as 4. `"workers": 4,`

2) Update [upf.json](./conf/upf.json) to use appropriate CNDP jsonc file with required number of queues.

`"cndp_jsonc_file": "/opt/bess/bessctl/conf/cndp_upf_4worker.jsonc"`

3) Modify the script [docker_setup.sh](./scripts/docker_setup.sh) and update the function `move_ifaces()` in condition `if [ "$mode" == 'cndp' ]`.

Update `num_q` value same as number of worker threads

Update `start_q_idx` to choose the start queue index to receive n/w packets.

4) Modify the script  [docker_setup.sh](./scripts/docker_setup.sh) to assign 5 cores (4 worker thread, 1 main thread) to run BESS UPF pipeline. `--cpuset-cpus=22-26`

5) From the top level directory call:
```
$ ./scripts/cndp_reset_upf.sh
$ ./scripts/docker_setup.sh
```
Insert rules into relevant PDR and FAR tables
```
$ docker exec bess-pfcpiface pfcpiface -config /conf/upf.json -simulate create
```
6) From browser, use localhost:8000 to view the UPF pipeline in GUI. If you are remotely connecting to system via ssh, you need to setup a tunnel with local port forwarding.

