# Proxmox VE Homelab Build Documentation

Turning an old laptop into an always-on virtualization server to run a blue-team security lab (Wazuh, Suricata, and other SOC analyst tools).

This is a step-by-step record of how I installed Proxmox VE from scratch and got it fully online written so a beginner can follow it and a recruiter can see the thought process behind each decision.

**Status:** Proxmox installed, network configured, web dashboard reachable. Ready for the first security VM.

---

## What I Built and Why

I repurposed an aging Lenovo ThinkPad into a dedicated **hypervisor** a server whose whole job is to run virtual machines. I chose **Proxmox VE** because it is:

- **Free and open source**
- A **type-1 (bare-metal) hypervisor**, meaning it installs directly on the hardware instead of inside another operating system, so it runs VMs efficiently
- **Purpose-built** for running many VMs and containers on modest hardware

The end goal is a safe, isolated environment where I can build and break security tools without risking a production machine.

---

## Hardware

| Component | Detail |
|-----------|--------|
| Model | Lenovo ThinkPad (20BFS3KW00) |
| CPU | Intel Core i7-4710MQ @ 2.50GHz (4 cores / 8 threads) |
| RAM | 8 GB DDR3 |
| Storage | 932 GB HDD |
| Virtualization | Intel VT-x supported |

**Honest hardware notes:**
- 8 GB RAM is the main limit. Proxmox uses ~1-2 GB, leaving ~6 GB for VMs enough for a few small VMs. A RAM upgrade (this model supports 16 GB) is planned.
- The mechanical hard drive is slower than an SSD; an SSD swap is on the roadmap.
- The CPU is more than enough for this lab.

---

## Software

- **Proxmox VE 9.2** (built on Debian Linux)
- **Filesystem:** ext4

---

## Step-by-Step Build

### Step 1 Download the Proxmox ISO

Downloaded the installer from the official site only:
`https://www.proxmox.com/en/downloads` -> **Proxmox VE 9.2 ISO Installer**.

**Security habit:** I verified the download using its SHA256 checksum (published on the same page) to confirm the file wasn't corrupted or tampered with. On Windows PowerShell:

```powershell
Get-FileHash "$env:USERPROFILE\Downloads\proxmox-ve_9.2-1.iso" -Algorithm SHA256
```

If the result matches the website's hash, the file is intact.

### Step 2 Make a Bootable USB

An ISO file can't just be copied onto a USB stick it has to be *written* in a special bootable format using a flashing tool.

**What didn't work:** I first tried **balenaEtcher**. It kept refusing to load the ISO (showed a red "no entry" symbol and never accepted the file).

**What worked Rufus:**
1. Downloaded Rufus from `https://rufus.ie`.
2. Selected my USB drive.
3. Boot selection -> **SELECT** -> chose the Proxmox ISO.
4. Clicked **START**.
5. Rufus detected an "ISOHybrid image" and asked how to write it — I chose **DD Image mode** (this is required for Proxmox to boot).
6. Confirmed the warnings about erasing the USB.
7. Waited for the green **READY** bar.

**Lesson:** for Proxmox, use Rufus in **DD mode**. Etcher and file-copy mode don't reliably produce a bootable Proxmox stick.

### Step 3 Boot From the USB

1. Shut the laptop down fully.
2. Inserted the USB and powered on, tapping **F12** to open the boot menu.
3. Selected the USB drive.
4. At the Proxmox menu, chose **Install Proxmox VE (Graphical)**.

### Step 4 The Virtualization Warning

The installer warned:

```
No support for hardware-accelerated KVM virtualization detected.
Check BIOS settings for Intel VT / AMD-V / SVM.
```

This is a **warning, not an error.** My CPU supports Intel VT-x, but it was switched off in the BIOS.

- **To fix properly:** reboot into BIOS (F1 on ThinkPad) -> Security -> Virtualization -> set *Intel Virtualization Technology* to **Enabled** -> save with F10.
- **To continue anyway:** Proxmox still installs and runs; VMs just run slower until VT-x is enabled.

I continued the install and noted VT-x as a follow-up task.

### Step 5 Installer Settings

| Setting | What I chose | Why |
|---------|--------------|-----|
| Target disk | The 932 GB internal drive | Only disk; Windows was wiped |
| Filesystem | **ext4** | Simplest, lightest choice for one disk |
| Country / Timezone | South Africa / Johannesburg | Local |
| Keyboard | U.S. English | Standard |

**Why ext4?** The installer also offers ZFS and BTRFS. ZFS needs lots of RAM and only makes sense with multiple disks; BTRFS is still marked "technology preview." For a single disk with limited RAM, **ext4** is the correct, low-overhead choice.

### Step 6 Password and Network (Install-Time)

- Set a strong **root** password (this logs into both the console and the web dashboard stored securely, never in this repo).
- The laptop had no network connection during install, so I used **placeholder** network values. Proxmox installs fine regardless; the IP gets corrected once the server is on a real network.

### Step 7 — Install and First Boot

- Confirmed the summary and clicked **Install**.
- Removed the USB during the reboot so it booted from the hard drive.
- The system booted and showed the login banner with a web address and a `pve login:` prompt.
- Logged in as `root` at the console to confirm the install was healthy reached the `root@pve:~#` command prompt successfully.

**At this point Proxmox is fully installed.**

---

## Networking Getting the Dashboard Online

This was the final hurdle and a good networking lesson.

### Diagnosing the connection

At first boot, I checked the network with:

```bash
ip a
```

It showed **no active connection** at first — the Ethernet port had no cable, and Wi-Fi was down. So the web dashboard wasn't reachable yet.

**Wi-Fi limitation I discovered:** Proxmox doesn't include Wi-Fi tools by default. Checking for them returned nothing:

```bash
which wpa_supplicant iw
# (no output = not installed)
```

This creates a catch-22: installing Wi-Fi tools needs internet, but getting internet over Wi-Fi needs those tools. **The takeaway:** Proxmox is wired-first. A server should connect by Ethernet, which is also more reliable than Wi-Fi.

### Fixing the IP to match my network

Once an Ethernet cable was connected to the router, `ip a` showed the Ethernet interface (`nic0`) and bridge (`vmbr0`) as **UP**, but the bridge still held the placeholder IP `192.168.100.2`, which didn't match my home network (`192.168.0.x`).

I edited the network config:

```bash
nano /etc/network/interfaces
```

In the `vmbr0` block I changed the placeholder values to match the home network:

```
address 192.168.0.50/24
gateway 192.168.0.1
```

Saved (Ctrl+O, Enter) and exited (Ctrl+X), then applied the change without rebooting:

```bash
ifreload -a
```

### Verifying connectivity

Confirmed the new IP and tested reachability:

```bash
ip a                 # vmbr0 now shows inet 192.168.0.50/24
ping -c 4 8.8.8.8    # 4 received, 0% packet loss -> internet works
```

### A networking gotcha worth remembering

When I first tried to open the dashboard from my Mac, it failed with "Destination Net Unreachable." A `ping` from the Mac revealed it was on a **different subnet** (`192.168.52.x`) than the server (`192.168.0.x`). Two devices on different subnets can't talk to each other directly. After getting the Mac onto the same `192.168.0.x` network, the dashboard loaded.

**Lesson:** the client and the server must be on the same network/subnet. Always confirm both ends with `ping` before blaming the application.

### Reaching the dashboard

From a browser on the same network:

```
https://192.168.0.50:8006
```

The browser showed a **"Connection Is Not Private" / "Not Secure"** warning. This is **expected** Proxmox uses a self-signed TLS certificate, so the browser can't verify it against a public certificate authority. On a private network with your own server, it is safe to proceed past the warning.

Logged in with user `root`, realm "Linux PAM standard authentication", and reached the Proxmox dashboard. A **"No valid subscription"** popup appeared normal for the free version; dismissed with OK. It does not limit functionality for a homelab.

**Result: a healthy, online Proxmox VE 9.2 host, managed remotely from a browser.**

---

## Key Lessons (Summary)

- **Use Rufus in DD mode** to write Proxmox USBs Etcher failed for me.
- **Choose ext4** on a single-disk, low-RAM host avoids unnecessary overhead.
- **Enable VT-x in BIOS** for full VM performance.
- **Proxmox is wired-first** Wi-Fi isn't supported out of the box; use Ethernet for a server.
- **Client and server must share the same subnet** confirm with `ping` before debugging the app.
- **The self-signed certificate warning is expected** safe to bypass on your own private network.
- **The "No valid subscription" popup is normal** for the free version and limits nothing.

---

## Security Note

All sensitive values root password, Wi-Fi key, router login, device IMEI, and MAC addresses are deliberately **left out of this documentation**. Real secrets should never be committed to a public repository. The internal IP addresses shown are private (RFC 1918) lab addresses and are not sensitive.

---

## Roadmap

- [x] Install Proxmox VE on bare metal
- [x] Configure networking and reach the web dashboard
- [ ] Enable Intel VT-x in BIOS
- [ ] Upgrade RAM (8 GB -> 16 GB) and add an SSD
- [ ] Deploy first VM **Wazuh** monitoring lab (SIEM/XDR)
- [ ] Add a monitored "victim" VM with the Wazuh agent
- [ ] Add **Suricata** network IDS
- [ ] Add a **Kali** attacker VM to generate and then detect attacks
- [ ] Document each VM build and detection exercise here

---

## Repository Structure

```
proxmox-homelab/
└── README.md
```
