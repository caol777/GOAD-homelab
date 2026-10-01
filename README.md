# Setting Up GOAD on Windows + VMware — A Real-World Walkthrough

Standing up the full **GOAD (Game of Active Directory)** lab on a Windows host with
VMware Workstation Pro, and connecting a Kali attacker box — including every snag
that isn't in the official docs and exactly how I got past each one. Written from
an actual build, not a copy of the README.

> **What GOAD is:** a deliberately vulnerable, multi-domain Active Directory
> environment for practicing offensive techniques (Kerberoasting, AS-REP roasting,
> ACL abuse, ADCS ESC, lateral movement). It is **intentionally insecure** — keep
> it on an isolated host-only network and never bridge it to your LAN or expose it
> to the internet.

---

## The lab

- **Host:** Windows, 64 GB RAM, VMware Workstation Pro
- **Lab:** Full GOAD — 5 VMs across 3 domains / 2 forests
  - `dc01` kingslanding `192.168.56.10` — sevenkingdoms.local
  - `dc02` winterfell `192.168.56.11` — north.sevenkingdoms.local
  - `dc03` meereen `192.168.56.12` — essos.local
  - `srv02` castelblack `192.168.56.22` — north.sevenkingdoms.local
  - `srv03` braavos `192.168.56.23` — essos.local
- **Networks:** host-only `192.168.56.0/24` on **VMnet2** (the lab) + NAT (internet)
- **Install path:** `goad.sh` via WSL (Debian 12) to build the VMs, then Ansible
  provisioning from a **separate Ubuntu VM** that sits on the lab network

> **The one lesson that explains this whole guide:** on Windows + VMware, building
> the VMs is easy. The *networking* between your provisioning host and the lab
> subnet is where every problem lives.

---

## Part 1 — Windows host prerequisites

Install on the host (VMware Workstation Pro assumed present):

1. Visual C++ 2019 redistributable
2. Vagrant
3. Vagrant VMware Utility

Then, in an **admin** PowerShell:

```powershell
vagrant.exe plugin install vagrant-reload vagrant-vmware-desktop winrm winrm-fs winrm-elevated
```

Reboot.

> **Disk:** the full lab needs ~115 GB. Use your largest drive, not a
> OneDrive-synced folder.

---

## Part 2 — WSL + building the VMs

Install WSL, then **Debian 12** from the Microsoft Store. Open it and create your
UNIX user.

![WSL first run](images/wsl-setup.png)

### Snag #1 — "Unable to locate package python3"

Fresh Debian has empty package lists; installs fail until you update first:

```bash
sudo apt update          # <-- do this FIRST
sudo apt install python3 python3-pip python3-venv libpython3-dev git
```

> It's `python3`, not `python` — the bare package doesn't exist on Debian 12.

![apt update fixing package lists](images/apt-update.png)

### Snag #2 — the clone directory doesn't exist yet

```bash
mkdir /mnt/c/goad        # <-- create it first or the cd fails
cd /mnt/c/goad
git clone https://github.com/Orange-Cyberdefense/GOAD.git
cd GOAD
./goad.sh
```

Drive the goad.sh console:

```
set_lab GOAD
set_provider vmware
set_ip_range 192.168.56
install
```

This builds all 5 VMs, then attempts Ansible provisioning.

---

## Part 3 — The VMware networking gauntlet

### Snag #3 — Vagrant won't configure the 2nd adapter on Windows

During the build:

```
==> GOAD-DC01: Configuring secondary network adapters through VMware
==> GOAD-DC01: on Windows is not yet supported. You will need to manually
==> GOAD-DC01: configure the network adapter.
```

So the host-only adapter is a manual job. Create the network:
**VMware → Edit → Virtual Network Editor → Change Settings → Add Network** →
**Host-only**, subnet `192.168.56.0 / 255.255.255.0`, **DHCP disabled** (GOAD
assigns static IPs itself). Note the VMnet number — mine was **VMnet2**.

![Virtual Network Editor — VMnet2 host-only](images/vmnet2-editor.png)

### Snag #4 — the VMs are invisible in the Library

`vagrant up` runs them as **background VMs**. To see them: right-click the
**VMware system-tray icon → Open All Background Virtual Machines**.

### Snag #5 — the 2nd adapter lands on the wrong VMnet

Vagrant attached some VMs' second adapter to **VMnet0** instead of VMnet2. Check
every VM: **Settings → Network Adapter 2 → Custom: Specific virtual network →
VMnet2**.

![Adapter 2 set to Custom VMnet2](images/adapter-vmnet2.png)

Verify from inside a VM (login `vagrant` / `vagrant`): `ipconfig` should show the
second adapter on `192.168.56.x`.

![ipconfig on DC01](images/dc01-ipconfig.png)

---

## Part 4 — Provisioning from the right machine

### Snag #6 — Ansible from WSL can't reach the lab

Provisioning from WSL, every host times out on port 5986:

```
fatal: [dc01]: UNREACHABLE! ... Connection to 192.168.56.10 timed out
```

**Why:** WSL sits behind its own NAT with no route to the VMware host-only
`192.168.56.0/24` network (GOAD GitHub issue #398). The VMs are fine — WSL just
can't reach them.

**Fix:** provision from a machine that's actually *on* VMnet2 — a small **Ubuntu VM**
inside VMware, dual-homed:

- **NAT** adapter — internet (to install Ansible, pip, clone GOAD)
- **Custom: VMnet2** adapter — the lab

### Snag #7 — Ubuntu gets no lab IP (DHCP is off)

DHCP is disabled on VMnet2, so set a static (outside GOAD's `.10–.23` range):

```bash
ip a                                          # find the VMnet2 iface (e.g. ens37)
sudo ip addr add 192.168.56.100/24 dev ens37
sudo ip link set ens37 up
```

### Snag #8 — ping lies

Pinging the Windows VMs hangs because **Windows firewall blocks ICMP by default**.
Don't trust ping. Test the actual port Ansible uses:

```bash
nc -zv 192.168.56.10 5986      # "succeeded" / "open" = good, ping or not
```

![nc + ping success](images/nc-ping-success.png)

### Snag #9 — adding the lab adapter can kill internet

If the NAT adapter drops to `NO-CARRIER` or the default route moves to the lab
interface, DNS breaks (`Temporary failure resolving`). Keep the NAT adapter
**Connected** in VM settings, and confirm **both** before provisioning:

```bash
ping -c3 google.com                 # internet + DNS
nc -zv 192.168.56.10 5986           # lab reachability
```

> The static `.56` IP is temporary and doesn't survive reboots — re-add it if
> `ip a` shows the lab interface lost its address.

---

## Part 5 — Run the provisioning

From the Ubuntu VM:

```bash
sudo apt update
sudo apt install git python3-venv python3-pip -y
cd ~
git clone https://github.com/Orange-Cyberdefense/GOAD.git
cd GOAD/ansible
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install ansible-core pywinrm
ansible-galaxy install -r requirements.yml
```

Recreate the per-instance inventory (copy it from the goad.sh workspace on the
Windows side — note the instance ID in the path differs per build):

```bash
mkdir -p ~/GOAD/workspace/<instance-id>-goad-vmware
nano ~/GOAD/workspace/<instance-id>-goad-vmware/inventory
```

```ini
[default]
dc01  ansible_host=192.168.56.10 dns_domain=dc01 dict_key=dc01
dc02  ansible_host=192.168.56.11 dns_domain=dc01 dict_key=dc02
srv02 ansible_host=192.168.56.22 dns_domain=dc02 dict_key=srv02
dc03  ansible_host=192.168.56.12 dns_domain=dc03 dict_key=dc03
srv03 ansible_host=192.168.56.23 dns_domain=dc03 dict_key=srv03
```

### Snag #10 — the inventory was missing srv03

My hand-made workspace inventory left out srv03, so it failed with a
name-resolution error (`Failed to resolve 'srv03'`) while the other four
provisioned. **Make sure all five hosts are listed**, srv03 included.

Run it:

```bash
ansible-playbook \
  -i ../ad/GOAD/data/inventory \
  -i ../workspace/<instance-id>-goad-vmware/inventory \
  main.yml
```

Takes 45–90 min with built-in reboots. **It's idempotent — if a task flakes, just
run it again and it resumes.** (I used a small re-runnable wrapper script that
re-asserts the lab IP and checks reachability before each run.)

![Ansible provisioning](images/ansible-running.png)

### Snag #11 — a host shows UNREACHABLE because it's *suspended*, not broken

On a later run, srv03 came back `No route to host`. The cause wasn't networking —
**the VM was suspended.** A suspended VM looks identical to a broken adapter from
Ansible's side. Always check state before assuming a network issue:

```powershell
# on the Windows host, in the provider folder
vagrant status
vagrant up GOAD-SRV03     # resume/boot the suspended one
```

> **Lesson:** `vagrant status` first. UNREACHABLE ≠ misconfigured network.

### Snag #12 — srv02 MSSQL install fails with a 404

srv02's MSSQL step dies on a dead Microsoft download URL:

```
Error downloading SQL2019-SSEI-Expr.exe ... (404) Not Found
```

This is an **upstream GOAD bug** (issues #521 / #522), not your setup — Microsoft
retired the hardcoded installer URL. It only affects the MSSQL service on braavos;
the rest of the host and every other attack path is unaffected. Fix by updating
the URL in the `mssql` role, or skip it.

---

## Result — a working forest

```
PLAY RECAP
dc01   : failed=0  unreachable=0   ✓
dc02   : failed=0  unreachable=0   ✓
dc03   : failed=0  unreachable=0   ✓
srv03  : failed=0  unreachable=0   ✓
srv02  : failed=1  unreachable=0   ~ (only the known MSSQL 404)
```

![Final PLAY RECAP](images/play-recap.png)

Three domains, two forests, trusts, seeded ACL-abuse paths, ADCS ESC templates,
Kerberoastable/AS-REP accounts, LLMNR/NBT-NS — all live.

---

## Part 6 — Connect a Kali attacker box

Same dual-homed pattern as the Ubuntu provisioner:

- Kali **Adapter 1 → NAT** (tools/internet)
- Kali **Adapter 2 → Custom: VMnet2** (the lab)

```bash
sudo ip addr add 192.168.56.150/24 dev eth1   # static, outside .10-.23
sudo ip link set eth1 up
nc -zv 192.168.56.10 5986                      # confirm lab reachability
```

GOAD is a Kerberos lab — name resolution is mandatory. Add the hosts:

```bash
sudo tee -a /etc/hosts <<'EOF'
192.168.56.10 kingslanding.sevenkingdoms.local sevenkingdoms.local kingslanding
192.168.56.11 winterfell.north.sevenkingdoms.local north.sevenkingdoms.local winterfell
192.168.56.12 meereen.essos.local essos.local meereen
192.168.56.22 castelblack.north.sevenkingdoms.local castelblack
192.168.56.23 braavos.essos.local braavos
EOF
```

### First enumeration

```bash
# map the hosts
nxc smb 192.168.56.10-23

# winterfell (dc02) allows anonymous — enumerate users
nxc smb 192.168.56.11 -u '' -p '' --users
nxc smb 192.168.56.11 -u 'guest' -p '' --rid-brute 10000
```

From a user list → AS-REP roasting → first cred → Kerberoasting + BloodHound →
follow the attack paths across the three domains.

---

## Takeaways

- On Windows + VMware, the build is easy; **host-only networking is the hard part**.
  Plan on manual per-VM adapter config.
- Your provisioning host must sit **on the lab subnet**. WSL can't; a dual-homed
  Ubuntu VM can.
- **Ping lies** (ICMP is firewalled). Test WinRM ports with `nc`.
- **`vagrant status` before blaming the network** — a suspended VM looks like a
  broken one.
- Provisioning is **idempotent** — re-run on failure.
- Keep the lab **isolated**. It's built to be broken into.

---

*Lab built on VMware Workstation Pro. GOAD by Orange Cyberdefense:
<https://github.com/Orange-Cyberdefense/GOAD>*
