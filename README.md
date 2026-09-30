# my-fedora-server

## CREDITS

- TechHut — [the ULTIMATE Fedora Server Guide](https://www.youtube.com/watch?v=kOEcTGZWiUQ)
  · [written version](https://techhut.tv/fedora-server-guide-cockpit-zfs-podman)

Fedora Server 44 setup

**Hardware:** Intel Celeron N5095A (4c, 800–2900 MHz) · 8 GB RAM · 238 GB SSD

**Status:** done through automatic updates · next: Docker

---

## 1. BIOS

Menu names vary on this board — `# check` all of them.

1. `Advanced` → `CPU Configuration` → Intel Virtualization Technology (VT-x) = `Enabled`
2. `Chipset` / `Power` → Restore on AC Power Loss = `Power On`
3. `Boot` → boot order: USB first to install, internal SSD after
4. `Boot` → Secure Boot = left enabled (only disable it if a DKMS module is needed)

## 2. First access

1. Get the server IP (from its own screen, or the router's DHCP list):

```bash
ip -4 addr show scope global | grep inet
```

2. Open Cockpit and accept the self-signed certificate:

```
https://{server_ip}:9090
```

3. Log in with the user created during installation. Terminal tab gives a full shell.

## 3. Install nano

First thing, so the config files below can be edited:

```bash
sudo dnf install -y nano
```

## 4. Config SSH

1. Generate the key on the client machine (Enter through all prompts):

```bash
ssh-keygen -t ed25519 -C "{key_comment}"
```

2. Print the public key and copy the whole line:

```bash
cat ~/.ssh/id_ed25519.pub
```

3. Add it to the server (Cockpit terminal). Paste into nano, `Ctrl+X` `Y` `Enter`:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

4. Test key login from the client:

```bash
ssh {user}@{server_ip}
```

5. Turn off password authentication:

```bash
sudo nano /etc/ssh/sshd_config.d/50-disable-password.conf
```

```
PasswordAuthentication no
```

```bash
sudo systemctl restart sshd
```

6. Confirm it is off — expected: `Permission denied (publickey,gssapi-keyex,gssapi-with-mic)`:

```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no {user}@{server_ip}
```

## 5. Expand root partition

The installer allocates ~15 GB to root regardless of disk size.

1. Check size, resolve the logical volume path and the free space in the volume group:

```bash
df -h /
findmnt -no SOURCE /     # -> /dev/mapper/{vg}-{lv}
sudo vgs                 # VFree column
```

2. Extend and grow the filesystem:

```bash
sudo lvextend -l +100%FREE {lv_path}
sudo xfs_growfs /
df -h /
```

## 6. Update the system

```bash
sudo dnf upgrade -y
```

## 7. Config dnf

1. Read the current config and the available options:

```bash
cat /etc/dnf/dnf.conf
man dnf5.conf
```

2. Edit:

```bash
sudo nano /etc/dnf/dnf.conf
```

```
[main]
defaultyes=True
max_parallel_downloads=10
fastestmirror=True
keepcache=True
```

## 8. Essential packages

```bash
sudo dnf install -y curl wget git htop net-tools unzip util-linux-user nano
```

## 9. Static IP

1. Get the device and connection names:

```bash
nmcli device status       # DEVICE and CONNECTION columns
nmcli connection show
```

2. Read the values DHCP already assigned — these are the ones to reuse in manual mode:

```bash
nmcli -f IP4 device show {iface}   # IP4.ADDRESS[1], IP4.GATEWAY, IP4.DNS[1]
ip -4 addr show {iface}            # address/prefix
ip route show default              # gateway, after "via"
resolvectl dns                     # DNS in use
```

3. Pick an address outside the router's DHCP range, keeping the same prefix and gateway:

```bash
sudo nmcli connection modify "{connection_name}" \
    ipv4.method manual \
    ipv4.addresses {static_ip}/{prefix} \
    ipv4.gateway {gateway} \
    ipv4.dns "{dns}"

sudo nmcli connection up "{connection_name}"
ip addr show {iface}
```

4. Or keep DHCP and set a reservation in the router instead:

```bash
sudo nmcli connection modify "{connection_name}" \
    ipv4.method auto \
    ipv4.addresses "" \
    ipv4.gateway "" \
    ipv4.dns ""

sudo nmcli connection up "{connection_name}"
```

If the connection does not apply the change:

```bash
sudo systemctl restart NetworkManager
```

## 10. Config firewall

1. Check state and current rules:

```bash
sudo firewall-cmd --state
sudo firewall-cmd --list-all
```

2. Resolve the service name to open, and which ports are actually listening:

```bash
sudo firewall-cmd --get-services | tr ' ' '\n' | grep -i {term}
sudo ss -tlnp
```

3. Open and reload:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=cockpit
sudo firewall-cmd --permanent --add-port={port}/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

## 11. Automatic updates

Fedora 44 runs dnf5: the package resolves to `dnf5-plugin-automatic` and the unit is
`dnf5-automatic.timer` — `dnf-automatic.timer` is the legacy compat unit.

1. Install and confirm the unit name:

```bash
sudo dnf install -y dnf-automatic
systemctl list-unit-files 'dnf*automatic*'
```

2. Configure — defaults live in `/usr/share/dnf5/dnf5-plugins/automatic.conf`, overrides go here:

```bash
sudo nano /etc/dnf/automatic.conf
```

```
[commands]
upgrade_type = security
apply_updates = yes
```

3. Enable the timer:

```bash
sudo systemctl enable --now dnf5-automatic.timer
systemctl list-timers 'dnf*'
```

---

## 12. Setup containerization

Might use docker..


## Not configured

- **ZFS** — single disk, not needed yet.
- **NVIDIA** — no dedicated GPU on this machine.
