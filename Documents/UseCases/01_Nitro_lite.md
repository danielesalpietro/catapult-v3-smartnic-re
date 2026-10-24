# Caso d'uso 1 – "Nitro-lite": SmartNIC di virtualizzazione su Catapult v3

Documento di progetto per riprodurre, in forma ridotta, l'architettura di AWS Nitro
sulle schede Catapult v3 documentate in questo repository (`Longs Peak` e `Dragontails Peak`).

L'obiettivo non è la parità di prestazioni o di sicurezza con Nitro, ma una piattaforma
didattica e sperimentale che ne riproduca i blocchi funzionali su hardware FPGA reperibile
a basso costo.

---

## 1. Obiettivo e non-obiettivi

### Obiettivo
Una scheda PCIe che:
- espone a un host Linux più dispositivi PCIe indipendenti (funzioni fisiche e virtuali),
  ognuno assegnabile a una macchina virtuale tramite VFIO;
- implementa in hardware i device model `virtio-net` e, in una fase successiva, `virtio-blk`,
  così che le VM usino i driver già presenti in ogni kernel;
- instrada il traffico di rete delle VM verso la porta QSFP+ a 40 GbE con incapsulamento
  e filtraggio in hardware;
- è gestita da un piano di controllo out-of-band eseguito su un soft core (Nios II/V) o,
  sulla variante Arria 10 SX, sull'HPS Cortex-A9;
- cifra i dati in transito con core AES a velocità di linea.

### Non-obiettivi
- Parità di throughput, latenza o densità di VM con una scheda Nitro reale.
- Un root of trust hardware equivalente al Nitro Security Chip.
- Compatibilità con il control plane o con i formati di AWS.
- Uso del NIC Mellanox a bordo: il collegamento FPGA ↔ ConnectX non è ancora stato
  stabilito (vedi TODO nel README). Tutto il traffico passa dal QSFP collegato all'FPGA.

---

## 2. Riferimento: i componenti di AWS Nitro e il loro equivalente

| Componente Nitro | Funzione | Equivalente in questo progetto | Fattibilità |
|---|---|---|---|
| Nitro Card for VPC | NIC virtualizzata con SR-IOV, encapsulation, security group | PF/VF PCIe + `virtio-net` HW + pipeline pacchetti + MAC 40G | Alta (lavoro principale) |
| Nitro Card for EBS | Disco NVMe "locale" servito via rete | `virtio-blk` HW con backend NVMe-oF/iSCSI gestito dal piano di controllo | Media |
| Nitro Card for Instance Storage | NVMe locale con cifratura | Fuori scopo (nessun flash a bordo) | – |
| Nitro Controller | SoC Arm, piano di controllo out-of-band | Nios II/V soft core, oppure HPS su Arria 10 SX (`Dragontails Peak` 10AS066) | Alta |
| Nitro Security Chip | Root of trust, boot attestato | Bitstream cifrato Arria 10 + soft RoT dimostrativo (riferimento: Caliptra) | Bassa (solo concetto) |
| Nitro Hypervisor | Hypervisor minimale su host | KVM + VFIO su Linux standard | Alta (solo software) |

---

## 3. Hardware target e risorse già disponibili

### Schede
- **Longs Peak** (PCIe): Arria 10 `10AXF40GAA`, due interfacce PCIe Gen3 x8 sull'edge x16
  (lane 1-8 e 9-16), 2 canali DDR4 72 bit, 1x QSFP+ via retimer DS250DF810, FT232H per JTAG.
- **Dragontails Peak** (OCP mezzanine): stesse risorse, PCIe sul connettore SAMTEC e su J8.
  La variante con `10AS066N3F40I2SGES` dispone di HPS Cortex-A9 dual core, ideale per il
  piano di controllo. Nota: questa variante non è ancora documentata nel repository.

### Blocchi già verificati in questo repository
| Blocco | Progetto | Stato |
|---|---|---|
| PCIe Gen3 x8, interfaccia 1 e 2, BAR + PIO | `Projects/PCIe/PCIe1`, `PCIe2`, `PCIe_Dual` | Testato su entrambe le schede |
| DDR4 72 bit, canale top e bottom | `Projects/DDR4/*` | Testato su entrambe le schede |
| Transceiver 10.3125 / 12.5 Gbps verso QSFP via retimer | `Projects/Network/SuperliteII_V4_QSFP` | Testato tra due schede |
| Configurazione retimer via I2C (indirizzo 0x22) | README, sezione retimer | Testato |
| Nios II + I2C + UART su JTAG | `SuperliteII_V4_QSFP/Software` | Testato |

### Vincoli numerici
| Risorsa | Valore | Implicazione |
|---|---|---|
| PCIe per interfaccia | Gen3 x8, ~7 GB/s utili | Sufficiente per una decina di VM a 40G aggregati |
| Link QSFP | 4x 10.3125 Gbps = 40 GbE | Un solo uplink, nessuna ridondanza |
| DDR4 | 2x 2.25 GB (ECC) | Code, buffer, tabelle di flusso, metadata virtio |
| Clock di fabric realistico | 250 MHz (PCIe) / 156.25 MHz (XGMII 64 bit per lane) | Pipeline a 512 bit per 40G con margine |
| Logica | equivalente 10AX115 (~1150k LE) | Abbondante per NIC + virtio + crypto |

---

## 4. Architettura proposta

```
                 Host x86 (Linux, KVM)
      VM1 (virtio-net/blk)  VM2 ...   Hypervisor (PF driver, VFIO)
            |                 |              |
   =========|=================|==============|========== PCIe Gen3 x8 (interfaccia 1)
            v                 v              v
   +----------------------------------------------------------------+
   |  Hard IP PCIe (Avalon-ST, SR-IOV)   PF0 + VF0..VFn              |
   |----------------------------------------------------------------|
   |  Device model virtio  (per VF: virtqueue TX/RX, MSI-X, config)  |
   |     virtio-net  |  virtio-blk (fase 4)                          |
   |----------------------------------------------------------------|
   |  DMA engine (descrittori in DDR4, scatter/gather)               |
   |----------------------------------------------------------------|
   |  Pipeline pacchetti @512 bit                                    |
   |   parser -> classificazione per VF -> ACL (security group)      |
   |   -> encap/decap (VXLAN) -> AES-GCM (fase 5) -> scheduler       |
   |----------------------------------------------------------------|
   |  MAC/PCS 40GBASE-R (4x10.3125)  -> retimer DS250DF810 -> QSFP+  |
   +-------------------+--------------------------------------------+
                       |
            Piano di controllo (Nios II/V o HPS)
            - enumerazione VF, MAC address, VLAN/VNI per VF
            - tabelle ACL e di flusso
            - configurazione retimer via I2C
            - backend storage (fase 4) e chiavi (fase 5)
            - console su UART/JTAG, poi su rete dedicata
```

### Scelte di progetto
- **virtio invece di un'interfaccia proprietaria.** Le VM usano `virtio_net` e `virtio_blk`
  di serie; nessun driver guest da scrivere. In alternativa, il driver ENA di AWS è open
  source nel kernel Linux e la sua interfaccia potrebbe essere riprodotta, ma senza vantaggi
  per questo progetto.
- **SR-IOV sull'hard IP Arria 10.** Il core "Avalon-ST Interface with SR-IOV" supporta più
  PF e un numero elevato di VF per Gen3 x8 (verificare i limiti esatti nello user guide Intel
  per la release di Quartus usata). Le VF vengono passate alle VM con VFIO.
- **Seconda interfaccia PCIe.** Riservata: può ospitare un secondo host (`Dragontails Peak`,
  connettore J8) o fungere da canale di debug/telemetria. Non necessaria per le fasi 1-5.
- **DDR4 per lo stato.** Descrittori, code e tabelle risiedono in DDR4; i buffer a bassa
  latenza in BRAM.
- **Piano di controllo in software.** Tutto ciò che non deve stare nel data path vive nel
  soft core o nell'HPS: semplifica l'hardware e permette di iterare senza risintetizzare.

---

## 5. Roadmap per fasi

| Fase | Deliverable | Criterio di accettazione | Stima (una persona con esperienza PCIe) |
|---|---|---|---|
| 1 | NIC PCIe semplice con DMA e driver Linux minimale | `ping` tra due schede collegate via QSFP, driver out-of-tree | 2-4 mesi |
| 2 | SR-IOV: PF + N VF, una VF passata a una VM | VM con VFIO vede la VF e fa `ping` | 1-2 mesi |
| 3 | `virtio-net` in hardware per ogni VF | VM usa `virtio_net` di serie, iperf tra VM su host diversi | 3-4 mesi |
| 4 | `virtio-blk` con backend in rete | VM avvia da disco servito via NVMe-oF/iSCSI dal piano di controllo | 3-6 mesi |
| 5 | Cifratura AES-GCM sul data path e chiavi per VF | Traffico cifrato tra due schede a velocità di linea | 1-2 mesi |
| 6 | Boot attestato dimostrativo | Bitstream cifrato, misura del firmware Nios, report via console | 1-2 mesi |

Le fasi 1 e 2 costituiscono già il cuore dell'idea e sono il primo traguardo da raggiungere.

### Fase 1 in dettaglio – NIC PCIe con DMA

**Hardware**
1. Partire da `Projects/PCIe/PCIe1` (interfaccia 1, BAR0/BAR1 già funzionanti).
2. Aggiungere un DMA engine con descrittori in anello (ring) in memoria host, scrittura
   di completamento e interrupt MSI-X. Opzioni: Platform Designer mSGDMA come prima
   iterazione, oppure un DMA custom a 256 bit allineato al bus Avalon-ST dell'hard IP.
3. Collegare una coda TX e una coda RX a un MAC 10G/40G. Prima iterazione: un solo lane a
   10.3125 Gbps riusando la configurazione transceiver del progetto Superlite II; poi
   4 lane con PCS 40GBASE-R.
4. Configurare il retimer da Nios II come già documentato (registro 0xFF = 0x3, registro
   0x2F maschera 0xF0 = 0x00 per 10.3125 Gbps).
5. Esporre su BAR0: registri di controllo, base/size degli anelli, puntatori head/tail,
   indirizzo MAC, contatori.

**Software host**
- Driver Linux out-of-tree `netdev` + `pci_driver`: alloca gli anelli con l'API DMA
  coerente, gestisce NAPI per RX, MSI-X per TX/RX, `ethtool` per i contatori.
- Nessun firmware aggiuntivo richiesto: il Nios II gestisce solo retimer e console.

**Test**
- Loopback interno del MAC, poi cavo QSFP tra due schede, `ping`, `iperf3`.
- Misura: throughput a 10G con MTU 1500 e 9000, latenza RTT, CPU host.

**Rischi specifici**
- Timing closure del DMA a 250 MHz sull'Arria 10 se il datapath non è pipelinato.
- Il MAC 40G richiede allineamento multi-lane e il retimer deve essere in modalità
  retimed (non bypass) per avere CDR lock stabile su tutte e quattro le lane.

---

## 6. Lato host: KVM e VFIO

Il "Nitro Hypervisor" viene sostituito da Linux standard:
1. Abilitare IOMMU (`intel_iommu=on` o `amd_iommu=on`) e verificare i gruppi IOMMU della scheda.
2. Caricare il driver PF (fase 2) e creare le VF con `sriov_numvfs`.
3. Associare le VF a `vfio-pci` e passarle alle VM con QEMU/libvirt.
4. Dalla fase 3 in poi la VM non ha bisogno di driver specifici: `virtio_net` e `virtio_blk`
   riconoscono il dispositivo dal vendor/device ID virtio (0x1AF4) esposto dalla VF.

---

## 7. Rischi e incognite

- **Documentazione hardware assente.** Pinout e vincoli derivano da reverse engineering;
  problemi di integrità di segnale vanno diagnosticati da soli. I blocchi PCIe, DDR4 e QSFP
  sono però già verificati in questo repository.
- **Licenza Quartus.** L'Arria 10 richiede Quartus Prime Standard o Pro con licenza.
- **SR-IOV sull'hard IP.** Variante di IP diversa da quella usata nei progetti attuali;
  richiede rifare la configurazione del core PCIe e del Platform Designer.
- **Raffreddamento.** Le schede richiedono flusso d'aria forzato; a pieno carico 40G più
  crypto il consumo cresce sensibilmente.
- **Un solo uplink a 40G.** Nessun failover, a differenza di Nitro.
- **Variante SX non documentata.** Usare l'HPS per il piano di controllo richiede prima
  completare il reverse engineering di quella scheda.

---

## 8. Riferimenti

- Progetti open comparabili
  - Corundum, NIC FPGA open source con supporto Intel (Stratix 10, Agilex): https://github.com/corundum/corundum
  - Caliptra, root of trust open (OCP): https://github.com/chipsalliance/caliptra
  - Driver ENA (AWS) nel kernel Linux: `drivers/net/ethernet/amazon/ena`
- Specifiche
  - Virtual I/O Device (VIRTIO) 1.2, OASIS
  - NVM Express Base Specification e NVMe over Fabrics
  - PCI-SIG Single Root I/O Virtualization
- Documentazione Intel
  - Arria 10 Avalon-ST Interface with SR-IOV for PCIe Solutions User Guide
  - Arria 10 Transceiver PHY User Guide (già in `Documents/Datasheets/U1 - 10AXF40GGA/`)
- Microsoft Catapult / Azure SmartNIC
  - Azure Accelerated Networking: SmartNICs in the Public Cloud (NSDI 2018)
  - Materiale già elencato nella sezione "External resources" del README
