# Use case 1 – "Nitro-lite": a virtualization SmartNIC on Catapult v3

Design notes for reproducing, in reduced form, the AWS Nitro architecture on the
Catapult v3 boards documented in this repository (`Longs Peak` and `Dragontails Peak`).

The goal is not performance or security parity with Nitro, but an educational and
experimental platform that reproduces its functional blocks on cheap, readily available
FPGA hardware.

Italian version: [01_Nitro_lite.md](01_Nitro_lite.md)

---

## 1. Goals and non-goals

### Goals
A PCIe card that:
- exposes several independent PCIe devices (physical and virtual functions) to a Linux
  host, each assignable to a virtual machine through VFIO;
- implements the `virtio-net` and, in a later phase, `virtio-blk` device models in
  hardware, so that VMs use the drivers already shipped with every kernel;
- routes VM network traffic to the 40 GbE QSFP+ port with encapsulation and filtering
  done in hardware;
- is managed by an out-of-band control plane running on a soft core (Nios II/V) or, on
  the Arria 10 SX variant, on the Cortex-A9 HPS;
- encrypts data in flight with line-rate AES cores.

### Non-goals
- Throughput, latency or VM-density parity with a real Nitro card.
- A hardware root of trust equivalent to the Nitro Security Chip.
- Compatibility with AWS control plane or formats.
- Use of the on-board Mellanox NIC: the FPGA ↔ ConnectX link has not been established yet
  (see the README TODO list). All traffic goes through the QSFP wired to the FPGA.

---

## 2. Reference: AWS Nitro components and their counterparts

| Nitro component | Function | Counterpart in this project | Feasibility |
|---|---|---|---|
| Nitro Card for VPC | Virtualized NIC with SR-IOV, encapsulation, security groups | PCIe PF/VF + HW `virtio-net` + packet pipeline + 40G MAC | High (main effort) |
| Nitro Card for EBS | "Local" NVMe disk served over the network | HW `virtio-blk` with an NVMe-oF/iSCSI backend managed by the control plane | Medium |
| Nitro Card for Instance Storage | Local NVMe with encryption | Out of scope (no flash on board) | – |
| Nitro Controller | Arm SoC, out-of-band control plane | Nios II/V soft core, or HPS on Arria 10 SX (`Dragontails Peak` 10AS066) | High |
| Nitro Security Chip | Root of trust, attested boot | Encrypted Arria 10 bitstream + demo soft RoT (reference: Caliptra) | Low (concept only) |
| Nitro Hypervisor | Minimal hypervisor on the host | KVM + VFIO on stock Linux | High (software only) |

---

## 3. Target hardware and resources already available

### Boards
- **Longs Peak** (PCIe): Arria 10 `10AXF40GAA`, two PCIe Gen3 x8 interfaces on the x16
  edge (lanes 1-8 and 9-16), two 72-bit DDR4 channels, 1x QSFP+ through the DS250DF810
  retimer, FT232H for JTAG.
- **Dragontails Peak** (OCP mezzanine): same resources, PCIe on the SAMTEC connector and on
  J8. The variant with `10AS066N3F40I2SGES` has a dual-core Cortex-A9 HPS, ideal for the
  control plane. Note: this variant is not documented in the repository yet.

### Blocks already verified in this repository
| Block | Project | Status |
|---|---|---|
| PCIe Gen3 x8, interfaces 1 and 2, BAR + PIO | `Projects/PCIe/PCIe1`, `PCIe2`, `PCIe_Dual` | Tested on both boards |
| 72-bit DDR4, top and bottom channels | `Projects/DDR4/*` | Tested on both boards |
| 10.3125 / 12.5 Gbps transceivers to QSFP through the retimer | `Projects/Network/SuperliteII_V4_QSFP` | Tested between two boards |
| Retimer configuration over I2C (address 0x22) | README, retimer section | Tested |
| Nios II + I2C + UART over JTAG | `SuperliteII_V4_QSFP/Software` | Tested |

### Numeric constraints
| Resource | Value | Implication |
|---|---|---|
| PCIe per interface | Gen3 x8, ~7 GB/s usable | Enough for about ten VMs at 40G aggregate |
| QSFP link | 4x 10.3125 Gbps = 40 GbE | Single uplink, no redundancy |
| DDR4 | 2x 2.25 GB (ECC) | Queues, buffers, flow tables, virtio metadata |
| Realistic fabric clock | 250 MHz (PCIe) / 156.25 MHz (64-bit XGMII per lane) | 512-bit pipeline for 40G with margin |
| Logic | 10AX115 equivalent (~1150k LE) | Plenty for NIC + virtio + crypto |

---

## 4. Proposed architecture

```
                 x86 host (Linux, KVM)
      VM1 (virtio-net/blk)  VM2 ...   Hypervisor (PF driver, VFIO)
            |                 |              |
   =========|=================|==============|========== PCIe Gen3 x8 (interface 1)
            v                 v              v
   +----------------------------------------------------------------+
   |  PCIe Hard IP (Avalon-ST, SR-IOV)   PF0 + VF0..VFn              |
   |----------------------------------------------------------------|
   |  virtio device model (per VF: TX/RX virtqueues, MSI-X, config)  |
   |     virtio-net  |  virtio-blk (phase 4)                         |
   |----------------------------------------------------------------|
   |  DMA engine (descriptors in DDR4, scatter/gather)               |
   |----------------------------------------------------------------|
   |  Packet pipeline @512 bit                                       |
   |   parser -> per-VF classification -> ACL (security group)       |
   |   -> encap/decap (VXLAN) -> AES-GCM (phase 5) -> scheduler      |
   |----------------------------------------------------------------|
   |  40GBASE-R MAC/PCS (4x10.3125) -> DS250DF810 retimer -> QSFP+   |
   +-------------------+--------------------------------------------+
                       |
            Control plane (Nios II/V or HPS)
            - VF enumeration, MAC address, VLAN/VNI per VF
            - ACL and flow tables
            - retimer configuration over I2C
            - storage backend (phase 4) and keys (phase 5)
            - console over UART/JTAG, later over a dedicated network
```

### Design choices
- **virtio instead of a proprietary interface.** VMs use the stock `virtio_net` and
  `virtio_blk` drivers; no guest driver to write. Alternatively, the AWS ENA driver is
  open source in the Linux kernel and its interface could be reproduced, but with no
  benefit for this project.
- **SR-IOV on the Arria 10 hard IP.** The "Avalon-ST Interface with SR-IOV" core supports
  several PFs and a large number of VFs for Gen3 x8 (check the exact limits in the Intel
  user guide for the Quartus release in use). VFs are handed to VMs through VFIO.
- **Second PCIe interface.** Reserved: it can host a second host (`Dragontails Peak`, J8
  connector) or act as a debug/telemetry channel. Not needed for phases 1-5.
- **DDR4 for state.** Descriptors, queues and tables live in DDR4; low-latency buffers in
  BRAM.
- **Control plane in software.** Everything that does not belong in the data path lives in
  the soft core or the HPS: simpler hardware and iteration without resynthesis.

---

## 5. Phased roadmap

| Phase | Deliverable | Acceptance criterion | Estimate (one person with PCIe experience) |
|---|---|---|---|
| 1 | Simple PCIe NIC with DMA and a minimal Linux driver | `ping` between two boards linked over QSFP, out-of-tree driver | 2-4 months |
| 2 | SR-IOV: PF + N VFs, one VF passed to a VM | VM with VFIO sees the VF and can `ping` | 1-2 months |
| 3 | Hardware `virtio-net` per VF | VM uses stock `virtio_net`, iperf between VMs on different hosts | 3-4 months |
| 4 | `virtio-blk` with a network backend | VM boots from a disk served over NVMe-oF/iSCSI by the control plane | 3-6 months |
| 5 | AES-GCM on the data path, per-VF keys | Encrypted traffic between two boards at line rate | 1-2 months |
| 6 | Demo attested boot | Encrypted bitstream, Nios firmware measurement, report over console | 1-2 months |

Phases 1 and 2 already form the core of the idea and are the first milestone to reach.

### Phase 1 in detail – PCIe NIC with DMA

**Hardware**
1. Start from `Projects/PCIe/PCIe1` (interface 1, BAR0/BAR1 already working).
2. Add a DMA engine with ring descriptors in host memory, completion writes and MSI-X
   interrupts. Options: Platform Designer mSGDMA as a first iteration, or a custom 256-bit
   DMA aligned with the hard IP Avalon-ST bus.
3. Connect one TX queue and one RX queue to a 10G/40G MAC. First iteration: a single lane
   at 10.3125 Gbps reusing the transceiver configuration from the Superlite II project;
   then 4 lanes with a 40GBASE-R PCS.
4. Configure the retimer from Nios II as already documented (register 0xFF = 0x3, register
   0x2F mask 0xF0 = 0x00 for 10.3125 Gbps).
5. Expose on BAR0: control registers, ring base/size, head/tail pointers, MAC address,
   counters.

**Host software**
- Out-of-tree Linux `netdev` + `pci_driver`: allocates the rings with the coherent DMA
  API, NAPI for RX, MSI-X for TX/RX, `ethtool` for counters.
- No extra firmware required: Nios II only handles the retimer and the console.

**Testing**
- Internal MAC loopback, then a QSFP cable between two boards, `ping`, `iperf3`.
- Measure: 10G throughput with MTU 1500 and 9000, RTT latency, host CPU.

**Phase-specific risks**
- Timing closure of the DMA at 250 MHz on the Arria 10 if the datapath is not pipelined.
- The 40G MAC needs multi-lane alignment and the retimer must be in retimed mode (not
  bypass) to get a stable CDR lock on all four lanes.

---

## 6. Host side: KVM and VFIO

The "Nitro Hypervisor" is replaced by stock Linux:
1. Enable the IOMMU (`intel_iommu=on` or `amd_iommu=on`) and check the card's IOMMU groups.
2. Load the PF driver (phase 2) and create the VFs with `sriov_numvfs`.
3. Bind the VFs to `vfio-pci` and pass them to the VMs with QEMU/libvirt.
4. From phase 3 on, the VM needs no specific driver: `virtio_net` and `virtio_blk`
   recognize the device from the virtio vendor/device ID (0x1AF4) exposed by the VF.

---

## 7. Risks and unknowns

- **No hardware documentation.** Pinouts and constraints come from reverse engineering;
  signal integrity issues must be diagnosed on your own. The PCIe, DDR4 and QSFP blocks are
  however already verified in this repository.
- **Quartus license.** The Arria 10 requires Quartus Prime Standard or Pro with a license.
- **SR-IOV on the hard IP.** A different IP variant from the one used in the current
  projects; the PCIe core and Platform Designer configuration must be redone.
- **Cooling.** The boards need forced airflow; at full 40G load plus crypto the power draw
  grows noticeably.
- **Single 40G uplink.** No failover, unlike Nitro.
- **Undocumented SX variant.** Using the HPS for the control plane first requires finishing
  the reverse engineering of that board.

---

## 8. References

- Comparable open projects
  - Corundum, open-source FPGA NIC with Intel support (Stratix 10, Agilex): https://github.com/corundum/corundum
  - Caliptra, open root of trust (OCP): https://github.com/chipsalliance/caliptra
  - ENA driver (AWS) in the Linux kernel: `drivers/net/ethernet/amazon/ena`
- Specifications
  - Virtual I/O Device (VIRTIO) 1.2, OASIS
  - NVM Express Base Specification and NVMe over Fabrics
  - PCI-SIG Single Root I/O Virtualization
- Intel documentation
  - Arria 10 Avalon-ST Interface with SR-IOV for PCIe Solutions User Guide
  - Arria 10 Transceiver PHY User Guide (already in `Documents/Datasheets/U1 - 10AXF40GGA/`)
- Microsoft Catapult / Azure SmartNIC
  - Azure Accelerated Networking: SmartNICs in the Public Cloud (NSDI 2018)
  - Material already listed in the README "External resources" section
