# Kali Linux VM on VMware Fusion — Build & Setup

**Host:** Apple Silicon Mac (15 cores, 24 GB RAM), macOS (Darwin 25.4)
**Guest:** Kali Linux 2026.1 (ARM64)
**Hypervisor:** VMware Fusion 26.0.0
**Date built:** 2026-06-03

---

## 0. Why this rebuild happened

The previous Kali VM had recurring **networking/DNS failures**:
- A "NetworkManager failed" error where the guest couldn't get an IP.
- `host can't be found` errors for Terraform/apt (DNS resolution failing).
- `ping` by hostname failing while raw IPs sometimes worked.
- Symptom traced to `/etc/resolv.conf` being overwritten with bad/empty nameservers.

**Root-cause theory:** a corrupt VMware NAT (vmnet8) DHCP/DNS config on the host
side, combined with NetworkManager state issues in the guest. Rather than keep
patching, we did a **clean rebuild** of both Fusion and the VM. A clean rebuild
regenerates the vmnet NAT config from scratch, which is the part most likely to
have been corrupt.

---

## 1. Uninstall old VMware Fusion (host)

Fully removed the old Fusion 26 (build 25388279) to guarantee a clean network stack:
- The app in `/Applications`
- `/Library/Application Support/VMware`
- `/Library/Preferences/VMware Fusion`  ← **the suspect-corrupt vmnet NAT/DHCP config lived here**
- The three `com.vmware.*Helper` LaunchDaemons
- `/var/db/vmware`
- All user-level config

Verified zero `vmware`/`vmnet` processes remained before reinstalling.
Rebooted the Mac (it had 34 days uptime, so a fresh boot was warranted).

---

## 2. Reinstall VMware Fusion (host)

Reinstalled from `~/Downloads/VMware-Fusion-26H1-25388279_universal.dmg`
(drag to `/Applications`, launch, allow it to install network services fresh
with admin password).

**Verification that the reinstall created clean network services:**
```
pgrep -lf vmware
# showed vmnet-dhcpd for vmnet1 AND vmnet8 (NAT) running with fresh configs,
# plus vmware-usbarbitrator. This confirms the corrupt vmnet8 DHCP config is
# gone and regenerated clean — the likely fix for the DNS failure.
```

---

## 3. Build the VM (specs + reasoning)

Created via Fusion's **File → New → Install from disc or image** wizard, then
**Customize Settings** (NOT Finish) so specs could be set before first boot.

| Setting | Value | Why |
|---|---|---|
| Architecture | **ARM64 / AArch64** | Apple Silicon **virtualizes**, it does not emulate. The guest CPU arch must match the host. ARM64 = AArch64 = "Arm" (all the same thing). amd64/x86-64/Intel images would not boot (or only via painfully slow emulation). |
| OS type | Debian 13.x 64-bit **Arm** | Kali Rolling tracks Debian testing; by 2026.1 that base is Debian 13 (Trixie). The OS-type field is only a *hint* Fusion uses to pick sensible device defaults — it doesn't install Debian. |
| Firmware | EFI | Required for ARM guests. The wizard set this automatically. |
| Memory | **4096 MB** | Leaves ~20 GB for macOS. macOS needs ~8-10 GB comfortably, so this keeps host memory pressure green. Fusion allocates guest RAM on demand, so it's not all consumed up front. |
| CPUs | **2 vCPU** | Plenty for a lab VM on a 15-core host. |
| Disk | **40 GB** | See note below — it defaulted to 20 GB and had to be expanded. |
| Network | **NAT** ("Share with my Mac") | Uses vmnet8. Guest gets a private IP behind the Mac, with the Fusion NAT gateway providing DHCP + DNS. This is the config we rebuilt clean. |
| Boot ISO | `~/Downloads/kali-linux-2026.1-installer-arm64.iso` | The **arm64** installer — matches the host arch. |

### Disk size fix (the one thing that didn't take)
The wizard's 40 GB change didn't apply; the disk landed at the default 20 GB.
Caught it via CLI inspection of the `.vmx`/`.vmdk`. Since nothing was installed
yet and the VM was powered off, expanded the empty sparse disk safely:

```
"/Applications/VMware Fusion.app/Contents/Library/vmware-vdiskmanager" \
  -x 40GB "~/Virtual Machines.localized/KaliVM.vmwarevm/Virtual Disk.vmdk"
```

Result: 40.0 GB (83,886,080 sectors × 512 bytes).

> **Note on the "42.9 GB" the installer shows:** 40 GiB (binary, what we set) =
> 40 × 1.073 billion bytes ≈ 42.9 **decimal** GB. Same disk, different unit
> convention. Storage tools constantly disagree on GiB vs GB — not a bug.

### VM bundle location
```
~/Virtual Machines.localized/KaliVM.vmwarevm/
  ├── KaliVM.vmx          ← config file (specs, devices, network)
  ├── Virtual Disk.vmdk   ← disk descriptor (+ split -s00N.vmdk extents)
  └── ...
```
The machine name (**KaliVM**) and the disk filename (**Virtual Disk.vmdk**) are
different layers — the disk just lives *inside* the KaliVM bundle. Naming the
disk to match the VM was unnecessary.

---

## 4. Kali installer choices

- **Graphical Install**
- Disk: `/dev/nvme0n1` (the virtual disk is exposed as NVMe per the `.vmx`)
- Partitioning: **Guided – use entire disk** → **All files in one partition**
  - One partition = all 40 GB in one flexible pool. Separate `/home`/`/var`/`/tmp`
    partitions only help on multi-user servers; for a single-purpose lab VM they
    just risk one partition filling while others sit empty.
- **Finish partitioning and write changes to disk → Yes**

---

## 5. First-boot configuration

Networking verified healthy immediately (the rebuild worked):
```
ping -c2 8.8.8.8      # raw connectivity OK
ping -c2 google.com   # DNS resolution OK
cat /etc/resolv.conf
#   # Generated by NetworkManager
#   search localdomain
#   nameserver 172.16.208.2     ← the VMware NAT gateway's DNS. Correct & clean.
```

System update + base tooling:
```
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y docker.io git open-vm-tools open-vm-tools-desktop
```
- `open-vm-tools` / `-desktop` → clipboard sharing, auto display resize.
- `docker.io` → Docker engine (Debian package). During install it asks
  *"Remove all Docker data?"* → answer **No** (don't wipe `/var/lib/docker`).

Enable Docker + allow running it without `sudo`:
```
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
sudo reboot                # reboot so the docker group membership takes effect
```

Post-reboot verification:
```
docker run --rm hello-world
# "Hello from Docker!" ... (arm64v8)  ← works without sudo, correct arch
```

---

## 6. Shell note

Kali has defaulted to **zsh** since 2020. The fancy
`┌──(user㉿kali)─[~]` prompt = zsh's default Kali theme. No shell change needed.
(Plain `$` would mean bash.) Verify with `echo $0`.

---

## Current state

✅ Fusion reinstalled clean · ✅ VM built to spec (ARM64, 4 GB, 2 CPU, 40 GB, NAT)
· ✅ Kali installed & fully updated · ✅ Networking/DNS healthy · ✅ Docker working

**Next:** CloudGoat install — see `CloudGoat-Docker-Setup.md`.
