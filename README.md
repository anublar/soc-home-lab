# SOC Home Lab (KVM/QEMU)

A personal blue-team lab built to practice the day-to-day skills of a SOC analyst (L1): collecting logs, generating realistic attack activity, and investigating the alerts it produces.

> **Status: work in progress.** Phase 1 (virtualization foundation) is complete. Later phases are listed in the [roadmap](#roadmap) and are marked as planned until they are actually built and documented.

## Goals

- Build a small but realistic detection environment on a single laptop.
- Practice the full loop: attack → log → alert → investigation → conclusion.
- Document every decision and every problem, so the lab can be reproduced and explained.

## Host hardware

| Component | Detail |
|---|---|
| Machine | Acer Nitro 5 (dedicated to the lab) |
| CPU | AMD Ryzen 7 2700U (4 cores / 8 threads, AMD-V) |
| RAM | ~14 GiB usable |
| System disk | 116 GB SSD (Ubuntu, Wazuh VM) |
| Lab disk | 1 TB HDD, ext4, mounted at `/mnt/soc-lab` (other VM disks, ISOs) |
| Host OS | Ubuntu 26.04.1 LTS |

The CPU is the main constraint, so not every VM will run at the same time: Wazuh plus one or two other machines per session.

## Architecture

```
Host: Ubuntu 26.04 LTS  (KVM + QEMU + libvirt)
│
├── default            NAT network, internet access (installs/updates only)
│
└── soc-lab-net        192.168.100.0/24, ISOLATED (no forwarding to LAN/internet)
    ├── Wazuh server   Ubuntu Server 24.04   [planned]  (second NIC on "default")
    ├── Kali Linux     attacker              [planned]
    ├── Metasploitable 2   intentionally vulnerable target   [planned]
    └── Windows endpoint   [planned]
```

## Design decisions

**KVM/QEMU instead of VirtualBox.** KVM runs inside the Linux kernel and uses the CPU's hardware virtualization directly, which matters on a modest laptop CPU. libvirt provides managed networks, storage pools and snapshots. It is also the same technology stack used by much of the cloud, so it is useful to understand.

**Separate disks for system and lab.** The VM disks live on a dedicated ext4 HDD (`/mnt/soc-lab`), registered as a libvirt storage pool, so lab data does not fill the system SSD. The disk is mounted by UUID in `/etc/fstab` with `nofail`, so the machine still boots if the disk is missing.

**Isolated lab network.** Metasploitable is vulnerable on purpose, so the attacker and target machines live on a network with no route to the home LAN or the internet. This is done by defining the libvirt network without a `<forward>` element.

**Wazuh on Ubuntu Server 24.04.** The Wazuh server documentation lists Ubuntu up to 24.04 as supported, so the SIEM VM uses 24.04 LTS even though the host runs 26.04.

**Verified downloads.** Every image was checked against its published SHA-256 hash before use.

## Phase 1 — Virtualization foundation (done)

### 1. Check hardware virtualization

```bash
lscpu | grep -E 'Model name|Virtualization'
ls -l /dev/kvm
sudo apt install -y cpu-checker && sudo kvm-ok
```

Expected: `AMD-V` listed, `/dev/kvm` present, and `KVM acceleration can be used`.

### 2. Install the virtualization stack

```bash
sudo apt install -y qemu-system-x86 libvirt-daemon-system libvirt-clients virt-manager ovmf swtpm
sudo usermod -aG libvirt,kvm $USER      # log out and back in afterwards
```

| Package | Role |
|---|---|
| `qemu-system-x86` | Emulator that runs the VMs (with KVM acceleration) |
| `libvirt-*` | Service and tools that manage VMs, networks and storage |
| `virt-manager` | Graphical interface |
| `ovmf` | UEFI firmware for VMs |
| `swtpm` | Virtual TPM (needed for Windows 11 later) |

### 3. Storage pool on the HDD

```bash
virsh pool-define-as soc-lab-vms dir --target /mnt/soc-lab/vms
virsh pool-build soc-lab-vms
virsh pool-start soc-lab-vms
virsh pool-autostart soc-lab-vms
virsh pool-list --all
```

### 4. Isolated virtual network

`soc-lab-net.xml`:

```xml
<network>
  <name>soc-lab-net</name>
  <bridge name='virbr-soc' stp='on' delay='0'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.100.100' end='192.168.100.200'/>
    </dhcp>
  </ip>
</network>
```

```bash
virsh net-define soc-lab-net.xml
virsh net-start soc-lab-net
virsh net-autostart soc-lab-net
virsh net-list --all
```

How it works: the `bridge` is a virtual switch on the host, the `ip` element gives the host an address on that network (defining the `192.168.100.0/24` subnet), `dhcp` hands out addresses to the VMs, and the absence of a `forward` element is what keeps the network isolated.

### 5. Download and verify images

```bash
sha256sum -c SHA256SUMS --ignore-missing   # Ubuntu Server 24.04
sha256sum <file>                           # Kali, Metasploitable: compare with the publisher's hash
```

### 6. Prepare the VM disks

Kali ships as a ready-to-use `qcow2` image. Metasploitable ships as a VMware `vmdk`, which is converted to `qcow2` (the native QEMU format, with snapshot support):

```bash
7z x kali-linux-*-qemu-amd64.7z -okali
unzip metasploitable-linux-2.0.0.zip -d metasploitable

sudo mv kali/kali-linux-*.qcow2 /mnt/soc-lab/vms/kali.qcow2
sudo qemu-img convert -p -f vmdk -O qcow2 \
  metasploitable/Metasploitable2-Linux/Metasploitable.vmdk \
  /mnt/soc-lab/vms/metasploitable2.qcow2

sudo chown libvirt-qemu:kvm /mnt/soc-lab/vms/kali.qcow2 /mnt/soc-lab/vms/metasploitable2.qcow2
sudo chmod 660 /mnt/soc-lab/vms/kali.qcow2 /mnt/soc-lab/vms/metasploitable2.qcow2
virsh pool-refresh soc-lab-vms && virsh vol-list soc-lab-vms
```

## Problems and lessons learned

**`sudo chown ... *.qcow2` failed with "No such file or directory".**
The storage directory is owned by root with mode `711` (`drwx--x--x`), so my user can enter it but cannot list it. The shell expands the `*` wildcard *before* `sudo` runs, as my user, so it found no matches and passed the literal text `*.qcow2` to the command. Fix: name the files explicitly (or expand the wildcard inside a root shell).

## Roadmap

- [x] Verify KVM and install the virtualization stack
- [x] Storage pool on a dedicated disk
- [x] Isolated virtual network
- [x] Download and verify images (Ubuntu Server 24.04, Kali, Metasploitable 2)
- [x] Prepare Kali and Metasploitable disks
- [ ] Create the Metasploitable and Kali VMs, confirm they reach each other on `soc-lab-net`
- [ ] Deploy Wazuh (all-in-one) on Ubuntu Server 24.04
- [ ] Enroll agents (Linux and Windows endpoints)
- [ ] Attack scenarios (scans, brute force, exploitation) with one write-up per scenario: attack → alert → investigation → conclusion
- [ ] Add Suricata (network IDS)
- [ ] Incident management and automation (TheHive, Shuffle)

## Safety note

Metasploitable 2 is deliberately insecure. It must only ever be attached to the isolated `soc-lab-net` network and never exposed to a real network.
