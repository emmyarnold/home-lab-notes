# Ubuntu Linux Client Joined to the Windows AD Domain

**Date:** June 22, 2026
**Goal:** Join my existing Ubuntu-Lab VM to the `lab.local` Active Directory domain (hosted on Server-Lab), then log in as a domain user and confirm AD group membership resolves on a Linux client.

### What I did
- Moved both Server-Lab (Windows Server 2025 DC) and Ubuntu-Lab off their separate NAT adapters onto a shared NAT Network so they could see each other.
- Gave Server-Lab a static IP (`10.0.2.4`). A DC needs one, since AD clients depend on it for DNS.
- Pointed Ubuntu-Lab's DNS at `10.0.2.4` so it would resolve `lab.local` through the DC.
- Installed the join tools on Ubuntu (`realmd`, `sssd`, `sssd-tools`, `libnss-sss`, `libpam-sss`, `adcli`, `packagekit`).
- Ran `sudo realm join lab.local -U Administrator` and checked it with `realm list`.
- Confirmed the whole thing by running `id jdoe@lab.local` and getting back the domain user's UID, primary group (`domain users`), and custom group (`it staff`).

### Tools / environment
- **Host:** macOS (Intel)
- **Hypervisor:** Oracle VirtualBox
- **Server-Lab:** Windows Server 2025, promoted to DC for `lab.local`
- **Ubuntu-Lab:** Ubuntu 26.04 LTS
- **Networking:** VirtualBox NAT Network (shared, default name `NatNetwork`)

### Commands used
```bash
# On Server-Lab (Windows, PowerShell):
netsh interface ip set address name="Ethernet" static 10.0.2.4 255.255.255.0 10.0.2.1

# On Ubuntu-Lab:
sudo apt update
sudo apt install realmd sssd sssd-tools libnss-sss libpam-sss adcli packagekit -y
sudo realm discover lab.local
sudo realm join lab.local -U Administrator
realm list
id jdoe@lab.local

# During the DNS troubleshooting:
nslookup lab.local
sudo resolvectl flush-caches
sudo systemctl restart systemd-resolved
resolvectl query lab.local
sudo resolvectl domain enp0s3 ~lab.local
nmcli connection show
sudo nmcli connection modify "netplan-enp0s3" ipv4.dns-search "~lab.local"
sudo nmcli connection up "netplan-enp0s3"
resolvectl status
```

### What I ran into

- **Server-Lab was assigning itself a link-local `169.254.x.x` address** instead of pulling a real IP from NAT, so DHCP wasn't behaving in the mixed setup. `ipconfig /release` + `/renew` didn't help. Gave Server-Lab a **static IP (`10.0.2.4`)** via `netsh`, which is what a domain controller should have anyway.

![Server-Lab autoconfig IP](screenshots/server-lab-autoconfig-ip.png)
![Static IP set on Server-Lab](screenshots/server-lab-static-ip-10.0.2.4.png)

- **`.local` DNS routing on Ubuntu.** After pointing Ubuntu-Lab's DNS at Server-Lab, `nslookup lab.local 10.0.2.4` worked when I named the DNS server explicitly, but plain `nslookup lab.local` came back with `** server can't find lab.local: REFUSED`. The culprit was `systemd-resolved`: Ubuntu treats `.local` as reserved for mDNS (link-local multicast) by default, so it wouldn't send `lab.local` queries to a regular DNS server at all. Fixed by adding `~lab.local` as a routing domain on the interface:

```bash
sudo resolvectl domain enp0s3 ~lab.local
```

That worked right away but wouldn't survive a reboot. To make it stick I edited the NetworkManager connection profile, which on this system is `netplan-enp0s3` rather than the generic "Wired connection 1" (worth running `nmcli connection show` to get the real name first):

```bash
sudo nmcli connection modify "netplan-enp0s3" ipv4.dns-search "~lab.local"
sudo nmcli connection up "netplan-enp0s3"
```

Checked it with `resolvectl status` — Link 2 (`enp0s3`) now shows `DNS Domain: ~lab.local` on its own, with no manual command needed.

![DNS troubleshooting and fix](screenshots/ubuntu-dns-troubleshooting-and-fix.png)

- **`nmcli` didn't recognize the default connection name.** Trying to modify "Wired connection 1" errored with `Error: unknown connection`. Ran `nmcli connection show`, saw it was actually `netplan-enp0s3`, and used that instead.

### What I learned
- Domain controllers need static IPs. AD clients rely on DNS pointing at the DC, so it can't be a moving target.
- `systemd-resolved` is picky about `.local`. It treats it as mDNS-only by default, which is fine on a normal home network but fails the moment you need to reach a real DNS server that serves a `.local` zone. The `~domain` routing syntax tells it to send queries for that domain to a normal DNS server instead.
- `resolvectl` changes things for the current session; `nmcli connection modify` writes it into the profile so it survives a reboot. You need both.
- The name Linux uses for a "connection" isn't always what you'd expect (`netplan-enp0s3` on a netplan-managed system, not "Wired connection 1"), so check before assuming.

### Verification
`id jdoe@lab.local` returned the domain user with the right UID and group membership, including the custom `it staff` group I'd made in AD:

![id jdoe domain user](screenshots/ubuntu-id-jdoe-domain-user.png)

Ubuntu resolving a Windows AD user through Kerberos with the correct groups.

### Screenshots
- `server-lab-autoconfig-ip.png` — Server-Lab stuck on link-local before fix
- `server-lab-release-renew-fail.png` — `ipconfig /release`/`/renew` didn't help
- `server-lab-static-ip-10.0.2.4.png` — static IP set via `netsh`
- `ubuntu-dns-set-to-dc.png` — Ubuntu IPv4 dialog with DNS = 10.0.2.4
- `ubuntu-dns-troubleshooting-and-fix.png` — the DNS routing saga
- `ubuntu-realm-join-success.png` — `realm join` and `realm list`
- `ubuntu-id-jdoe-domain-user.png` — end-to-end proof of domain user resolution
