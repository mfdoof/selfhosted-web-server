## System Specifications
**Operating System**: Ubuntu Server VM
**Hypervisor**: Proxmox
Machine: q35
SCSI Controller: VirtIO single
**Memory:**  8GB
**Storage:** 32GB

### Overview

This establishes the infrastructure foundation for the entire deployment. Every component in P onwards — Ollama, FastAPI, Nginx — depends on this phase completing successfully. A misconfigured. VM or failed GPU passthrough will cause silent failures downstream 
that are significantly harder to debug after the fact.

This phase must be completed and fully verified before proceeding. Do not move to the next until every item in the verification checklist passes.

### Virtual Machine Set up

#### Step by Step Configuration

1. Create the VM
	Creates an isolated virtual machine on Proxmox to host the entire application stack. q35 machine type and OVMF BIOS are required specifically for PCIe passthrough — without these, the GPU cannot be passed through to the VM.
	
	Go to Proxmox web UI → Create VM. Use these settings exactly:

	**General tab**
	- VM ID: your choice (e.g. 100)
	- Name: `network-monitor`
	
	**OS tab**
	- ISO: Ubuntu 24.04 LTS server (upload to Proxmox local storage first)
	- Type: Linux
	- Version: 6.x - 2.6 Kernel
	
	**System tab — critical settings**
	- Machine: `q35` — required for PCIe passthrough
	- BIOS: `OVMF (UEFI)` — required for GPU passthrough
	- EFI Storage: local-lvm
	- TPM: disable unless you need it
	- SCSI Controller: `VirtIO SCSI`
	
	**Disks tab**
	- Bus: `VirtIO Block`
	- Size: 32GB minimum — Ollama models take space (llama3.2 3B ≈ 2GB)
	- Cache: `Write back`
	
	**CPU tab**
	- Cores: 4 minimum
	- Type: `host` — gives VM access to all host CPU features, needed for CUDA
	
	**Memory tab**
	- RAM: 8GB minimum
	- Uncheck `Ballooning` — causes instability with GPU passthrough
	
	**Network tab**
	- Bridge: vmbr0
	- Model: `VirtIO`

**Prerequisites — Verify Before Adding GPU**
The following must be confirmed on the Proxmox host before attaching the GPU to the VM. Skipping this causes silent failures that are difficult to trace back to this step.

**Verify IOMMU is enabled:**

```bash
find /sys/kernel/iommu_groups/ -type l | sort -V
```

The GPU must appear in its own isolated group. If it shares a group 
with other critical devices, passthrough will be unstable.

**Verify GPU is bound to vfio-pci and not nouveau:**

```bash
lspci -nnk | grep -A3 "NVIDIA"
```


2. Add GPU Passthrough to VM
	Passes the physical GTX 1650 directly to the VM so Ollama can use CUDA for GPU-accelerated inference. Without passthrough, Ollama falls back to CPU which is significantly slower and defeats the purpose of having a dedicated GPU.

	In Proxmox web UI → select your VM → Hardware → Add → PCI Device.

	Settings:
	
	- Device: select your GTX 1650 (01:00.0)
	- Check `All Functions` — passes through both VGA and audio
	- Check `Primary GPU` — only if you have no display output needed from host
	- Check `ROM-Bar`
	- Check `PCI-Express`
	
	**Then edit the VM config directly** to add one line Proxmox UI doesn't expose:

```bash
	# On Proxmox host
	nano /etc/pve/qemu-server/100.conf  # replace 100 with your VM ID
	Add this line:
	cpu: host,hidden=1
	
```

	The `hidden=1` flag masks the hypervisor CPU flags from the NVIDIA driver. NVIDIA drivers detect when they are running inside a virtual machine and refuse to initialize as a licensing restriction — returning Error 43 in dmesg with no clear explanation. Adding `hidden=1` bypasses this check. Without it, the driver will appear to install successfully but nvidia-smi will fail and CUDA will be unavailable.

Full relevant section of your config should look like:

```
bios: ovmf
cpu: host,hidden=1
machine: pc-q35-8.1
memory: 8192
balloon: 0
hostpci0: 01:00,pcie=1,rombar=1,x-vga=1
```

3. Install Ubuntu 24.04
	Ubuntu Server 24.04 LTS was chosen over desktop for minimal footprint — no GUI means more RAM and CPU available for FastAPI and Ollama. OpenSSH is installed during setup so the VM is managed remotely from the Mac browser, never requiring direct console access during normal operation.
	
	Ubuntu server install — key choices:
	
	- Partitioning: use entire disk, LVM
	- Install OpenSSH server: **yes** — you'll want SSH access from your Mac
	- No extra snaps needed
	
	After install completes, eject ISO:

```bash
# On Proxmox host
qm set 100 --ide2 none,media=cdrom
```

Boot the VM, SSH in from the management host

```bash
ssh youruser@192.168.x.x
```

First thing after login:

```bash
sudo apt update && sudo apt upgrade -y
```


4. Install NVIDIA Drivers in VM
	Nouveau is the open source NVIDIA driver that ships with Linux by default. It must be blacklisted before installing the proprietary NVIDIA driver because the two conflict. `ubuntu-drivers install` automatically detects the GPU and installs the recommended driver version, removing the need to manually match driver and CUDA versions.

**Blacklist nouveau inside the VM**:


```bash
sudo nano /etc/modprobe.d/blacklist-nouveau.conf
```

```
blacklist nouveau
options nouveau modeset=0
```

```bash
sudo update-initramfs -u
```

**Install Driver:**

```bash
# Check what's available
sudo apt install ubuntu-drivers-common -y

# Install it
sudo ubuntu-drivers install
sudo reboot
```

Reboot the system then run:

```bash
nvidia-smi
```

If `nvidia-smi` fails:

```bash
# Check if driver loaded
lsmod | grep nvidia

# Check for errors
dmesg | grep -i nvidia

# Common fix — driver not loading because nouveau still active
sudo rmmod nouveau
sudo modprobe nvidia
```

**Install CUDA Toolkit**

```bash
# Match CUDA version to your driver
# Driver 550 supports CUDA 12.x
sudo apt install nvidia-cuda-toolkit -y

# Verify
nvcc --version
```

If `nvcc` returns command not found:

```bash
# Add CUDA to PATH
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
nvcc --version
```

5. Set Static IP or DHCP Reservation
	A stable IP is set before any other configuration because every downstream component depends on it — Nginx server_name, CORS origins, and the frontend API constant all reference this address. If the IP changes, all of these break simultaneously.

**Option A — DHCP reservation on your router (easier)**
Find the VM's MAC address:

```bash
ip link show
# Look for the interface (usually eth0 or ens18) and its MAC
```

Go to your router admin panel → DHCP reservations → bind that MAC to a fixed IP like `192.168.x.x`.

**Option B — Static IP on the VM (more reliable)**

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens18:                        # replace with your interface name
      dhcp4: no
      addresses:
        - 192.168.x.x/24
      routes:
        - to: default
          via: 192.168.1.1        # your router IP
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

```bash
sudo netplan apply
```

Confirm:

```bash
ip addr show ens18
# Should show your static IP
```

#### Verification Checklist

```bash
# 1. GPU visible and driver loaded
nvidia-smi

# 2. CUDA installed
nvcc --version

# 3. IP is stable
ip addr show

# 4. SSH works from Mac
# (run this on your Mac)
ssh youruser@192.168.x.x

# 5. Internet access from VM (needed for Phase 2 installs)
curl -I https://google.com
```

#### Failure Points

|Symptom|Cause|Fix|
|---|---|---|
|`nvidia-smi` not found after driver install|nouveau still loaded|Blacklist nouveau, reboot|
|Error 43 in dmesg|VM detected by driver|Add `hidden=1` to cpu line in VM config|
|`nvidia-smi` shows GPU but CUDA missing|Toolkit not installed|`apt install nvidia-cuda-toolkit`|
|VM won't boot after adding PCI device|ROM-bar issue|Try toggling ROM-bar off in PCI device settings|
|GPU not showing in Proxmox PCI list|vfio-pci not bound on host|Redo Step 1 on host, check lspci driver|
|netplan apply breaks network|YAML indentation error|YAML is indent-sensitive — use spaces not tabs|

### Problems Encountered

**TICKET-001 — `ubuntu-drivers autoinstall` command not found**

- **Symptom:** Running `sudo ubuntu-drivers autoinstall` returned `No such command 'autoinstall'`
- **Cause:** Command syntax changed in Ubuntu 24.04. The `autoinstall` subcommand was removed in the newer version of `ubuntu-drivers-common`
- **Fix:** Use `sudo ubuntu-drivers install` instead
- **Status:** Resolved

**TICKET-002 — Proxmox host shutdown during VM setup**

- **Symptom:** Proxmox host powered off unexpectedly while configuring the Ubuntu Server VM
- **Cause:** Host sleep/suspend was enabled by default on the motherboard. Server sat idle long enough to trigger suspend
- **Fix:** Disable sleep targets on Proxmox host permanently:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

- **Status:** Resolved

**TICKET-003 — GPU not available to add to the new VM**

- **Symptom:** The GTX 1650 did not appear as an addable PCI device — it was still attached to a different VM from a previous project
- **Cause:** GPU was bound to another VM via passthrough and had not been released
- **Fix:** Shut down the other VM, removed its `hostpci0` and `hidden=1` lines from its `/etc/pve/qemu-server/<id>.conf`, then confirmed `vfio-pci` rebound to the GPU on the host via `lspci -nnk`
- **Status:** Resolved
