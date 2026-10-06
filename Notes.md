# ITEC 2332 Systems Administration: Consolidated Notes

**Jump to:** §5 ports table (deck) · §8 all ports + how to check them · §9 Linux ⇄ Windows equivalents for every command that only had one side.

---

## 1. The Role

**Sysadmin** = installs, configures, maintains, secures and recovers the systems an organization depends on (servers, networks, storage, accounts, services). The job is the *working*, not the servers.

**Five kinds of work:** Provisioning, Maintenance, Support, Security, Recovery.

**Three responsibilities** (every ticket maps to one; say which):
- **Availability**: running when needed; measured as uptime.
- **Security**: only the right people get in. CIA triad = Confidentiality, Integrity, Availability.
- **Recoverability**: **RPO** = data you can afford to lose; **RTO** = downtime you can afford. *A backup never restored is a hope, not a backup.*

| Uptime | Down/year | Down/month | Typical of |
|---|---|---|---|
| 99% | 3.65 days | 7.3 h | unmonitored small-biz file server |
| 99.9% | 8.8 h | 44 min | well-run internal service |
| 99.99% | 52.6 min | 4.4 min | redundant public web service |
| 99.999% | 5.3 min | 26 s | telecom, payments, big cloud |

Each extra nine costs ~10x more. (3 h down in a month breaks a 99.9% promise.)

**Works with:** users (report symptoms, not causes), management (wants money/time/risk), other IT roles, vendors (what they leave is only as good as what they documented). Translate user symptoms into causes and back.

**System lifecycle:** Plan > Build > Operate > Change > Retire. Most problems come from skipping the last two (unpatched servers, never-retired servers still answering with old passwords).

**Practice answers:** disk fails = Recoverability; ex-employee still has email = Security; site down in sale = Availability ("when is it back?"); only one person knows month-end = Recoverability (single point of failure); untested backups = Recoverability.

---

## 2. What You Manage

**IT environment:** Clients, Network (switches, routers, firewall, Wi-Fi), Servers, Storage (disks, arrays, NAS, SAN, backups), Cloud. Services are what users touch: SMB/NFS, HTTP(S), SMTP/IMAP, AD/LDAP, DNS, DHCP, databases, printing, backup, monitoring. "Network is down" usually means one service is unreachable.

**Physical vs VM vs Cloud**
- Physical: one OS per machine; you own hardware, power, cooling.
- VM: hypervisor runs many isolated OSes (ESXi, Hyper-V, Proxmox, KVM, VirtualBox). Snapshots, cloning, migration. All labs are VMs.
- Cloud: rented VMs (AWS, Azure, GCP). OS, accounts, patching, backups, security still yours. "Someone else's building, not someone else's problem."

**OS anatomy (top to bottom):** Applications/services > Shell & tools (bash, PowerShell) > **Kernel** (CPU, memory, files, devices, network) > Hardware. OS provides processes, files, users (root/Administrator/SYSTEM), devices & network. "Server is slow" = CPU, memory, disk, or network.

**Linux vs Windows Server**

| | Linux | Windows Server |
|---|---|---|
| Cost | Free (paid support optional) | Per-core + CALs |
| Interface | CLI first | GUI first; PowerShell; Server Core = no GUI |
| Admin | root (UID 0); use sudo | Administrator / Domain Admins |
| Users | /etc/passwd or LDAP | Active Directory |
| Services | systemd, systemctl | Services console, sc.exe, Get-Service |
| Packages | apt, dnf | Windows Update, WSUS, installers |
| Files | one tree from `/`, case-sensitive, rwx | C:, D:, not case-sensitive, ACLs |
| Logs | journalctl, /var/log | Event Viewer |
| Typical | web, DB, containers | file servers, AD, Windows-only apps |

**Linux = kernel (Torvalds, 1991; `uname -r`) + GNU utilities + distribution.**

| Family | Distros | Packages | Notes |
|---|---|---|---|
| Debian | Debian, Ubuntu, Mint | apt/.deb | Ubuntu LTS every 2 yrs, 5 yrs support (class system) |
| Red Hat | RHEL, Rocky, Alma, CentOS Stream, Fedora | dnf/.rpm | RHEL paid; Rocky/Alma free rebuilds (enterprise example) |
| SUSE | SLES, openSUSE | zypper/.rpm | enterprise, SAP |
| Other | Arch, Alpine, Gentoo | pacman, apk, portage | Alpine runs most containers |
| Specialized | Kali, Raspberry Pi OS, Proxmox, TrueNAS | varies | |

**Terms:** host, server/client, service/daemon (sshd, httpd), port (web 80/443, SSH 22), protocol, instance/node. Root/Administrator/superuser/domain admin all = no limits: use rarely, log use, never share.

**Practice answers:** shared drive problem = servers/storage, SMB (or network/DNS); cloud move removes hardware/power/cooling/physical security but keeps patching, accounts, backups, firewall, monitoring; port 22 daemon = sshd; frozen-but-pings usually memory, sometimes CPU.

---

## 3. The Shell

**Why CLI:** servers have no GUI, it's repeatable (scriptable), precise. *Sentence model:* **command** (verb) + **options** (adverbs, -l -a -r) + **target** (noun). `ls -l /etc` = list, long form, /etc.

**Prompt:** `user@host:dir$` ($ = normal user, `#` = root, be careful).

**Getting help, in order:** `man` > `--help` > `which`/`type` > ask (say what you tried).
- `man ls` (q quit, `/` search, n next); sections: 1 commands, 5 file formats, 8 admin (`man 5 passwd`)
- `man -k "copy files"` (= apropos), `info coreutils`, `type cd`, `which nano`, `whatis grep`
- Synopsis: `[ ]` optional, `...` repeatable. Builtins (cd, echo, export) are in `man bash`.

**Commands to know**

| Command | Use |
|---|---|
| pwd | where am I |
| ls / -l / -la | list; details; hidden |
| cd | `..` up, `~` home, `/` top, `-` back |
| cat, less, head, tail (`tail -f` follows) | read files (read-only) |
| nano | Ctrl+O save, Ctrl+X exit, Ctrl+W search, Ctrl+K cut, Ctrl+U paste |
| mkdir, touch | make folder / empty file |
| cp, mv, rm | copy; move/rename; delete (no undo) |
| grep | search text |
| sudo | one command as root |

**History & editing keys:** Tab (twice = choices), Up/Down, `history`, `!42`, `!!` (`sudo !!`), `!ssh`, **Ctrl+R** reverse search, Ctrl+A/E line start/end, Ctrl+U/K delete to start/end, Ctrl+C stop, Ctrl+D logout/EOF, Ctrl+L clear. Saved in `~/.bash_history`. Remember three: **Tab, Up arrow, pipe.**

**Streams & redirection:** stdin(0), stdout(1), stderr(2).
```
> file       overwrite (ERASES)        >> file     append
2> err.txt   stderr to file            > out 2>&1  both
2>/dev/null  discard errors (dangerous in nightly scripts)
< file       stdin from file           << EOF      here-document
cmd1 | cmd2  pipe stdout to stdin      cmd | tee log.txt   see and save
```
Examples: `ps aux | grep nginx`; `dmesg | tail -20`; `du -sh /var/* | sort -h | tail -5` (what filled the disk).

**Environment variables:** `echo $HOME $PATH $SHELL $USER $PWD`; `env`, `printenv`, `set`. `MYVAR=x` is shell-only; `export MYVAR` makes children inherit it. `export EDITOR=nano`; `PATH=$PATH:/opt/tools/bin`. PATH is why `ls` works without /bin/ls; never add `.` to PATH (hence `./script.sh`). Permanent: `~/.bashrc` (interactive), `~/.profile` (login), `/etc/profile` (all users), `/etc/environment` (system-wide, no shell syntax). Convention: env vars UPPERCASE.

**PowerShell equivalents**

| Linux | PowerShell |
|---|---|
| pwd | Get-Location |
| ls -la | Get-ChildItem -Force |
| cd | Set-Location |
| cat | Get-Content |
| tail -f | Get-Content f -Wait -Tail 20 |
| grep | Select-String |
| echo $HOME | $env:USERPROFILE |
| ps aux | Get-Process |
| man | Get-Help X -Full |
| sudo | Run as Administrator |

Key difference: Linux pipes pass **text**; PowerShell pipes pass **objects** (`Get-Process | Where-Object CPU -gt 100`). Verb-Noun naming (Get-, Set-, New-, Remove-).

**Practice answers:** `wc -l /etc/passwd`; `grep bash /etc/passwd | cut -d: -f1 | sort`; `ls -la /etc > etc-listing.txt; ls /root 2> errors.txt`; re-run with `history` then `!N`; unexported variable is empty in child bash, visible after `export`; **`sudo rm -rf / tmp/build`** (stray space) = deletes `/` and `tmp/build`, catastrophic.

**Sept 8 lecture drills:** (1) `pwd; cd /etc; ls -l` (2) `cat /etc/hostname` vs `less` (3) `mkdir ~/lab2; cp /etc/hostname ~/lab2/myhost.txt; mv ... server.txt` (4) `ls -t` sorts by mtime, newest first; `ls -lt /var/log` (5) Up arrow, Ctrl-A or arrows to the typo, Enter. Exit ticket: the command for "where am I" (`pwd`) and for the manual (`man`). Next class: install a server; bring laptop with VirtualBox.

---

## 4. Files and Text

**Filesystem (FHS; Ubuntu and Rocky agree).** One tree from `/`, no drive letters, case-sensitive, everything is a file.

| Path | Purpose |
|---|---|
| /etc | config (plain text) |
| /var, /var/log, /var/www | changing data; logs (start here when broken); web content |
| /home, /root | user homes; root's home |
| /usr, /opt | programs/libraries; third-party software |
| /tmp | scratch, cleared at boot |
| /dev, /proc, /sys | devices; kernel exposed as files |
| /boot | kernel + bootloader |
| /mnt, /media | mount points |

Absolute path starts at `/`; relative starts where you stand (`log`, `../etc`). Quote or escape names with spaces.

**File types in `ls -l`:** `-` regular, `d` dir, `l` symlink, `b` block device, `c` char device. Extensions mean nothing: `file` reads contents. `ls -i` = inode, `stat` = all metadata.

**Inodes & links:** the name is a directory entry pointing at an inode (so rename is instant, a file can have two names).
- **Hard link** `ln a b`: same inode, can't cross filesystems or link dirs; data lives until last name removed.
- **Symlink** `ln -s a b`: pointer to a path; crosses filesystems, works on dirs, breaks if target moves.
- Delete original: hard link still works, symlink breaks.

**Wildcards/quoting:** shell expands wildcards (`rm *.log` becomes the list), so `echo *.log` first. `*`, `?`, `[abc]`, `[0-9]`, `[!a]`, `{jan,feb}.csv`. Double quotes expand `$vars`; single quotes literal; always quote `"$file"`; `rm -- -weird.txt`.

**Finding things:** `which` (PATH programs) > `whereis` > `locate` (nightly index; `sudo updatedb`) > `find` (live, slow).
```
find /var/log -name "*.log"          find / -iname "*README*" 2>/dev/null
find /home -user devon               find /srv -type d -perm 777   (world-writable)
find / -perm -4000 -type f           (setuid audit)
find /var/log -size +100M            find /tmp -mtime +7
find /var/www -newer /etc/motd       find ... -delete   (careful)
find /srv -type f -exec chmod 664 {} \;     find ... -print0 | xargs -0 ls -l
```
Test `-exec ls -l` before the real command. Week 1: `find /etc -name '*.conf'`, `find / -name sshd_config`; `grep error /var/log/syslog`, `grep -r ubuntu /etc` ("find = know the name; grep = know a word inside").

**Archiving:** flags `c` create, `x` extract, `t` list; `z` gzip, `j` bzip2 (smaller, slower), `J` xz (smallest); `v`; `f` + filename.
```
tar -czvf etc.tar.gz /etc      tar -tzvf etc.tar.gz      tar -xzvf etc.tar.gz -C /tmp/restore
tar -xzvf etc.tar.gz etc/hosts (one file)
gzip/gunzip/zcat/zless; bzip2; xz; zip -r / unzip -l
tar -czf - /var/www | ssh backup01 "cat > www-$(date +%F).tar.gz"
```
tar strips leading `/` so restores don't overwrite live files.

**Editing:** vim is on every system incl. rescue; learn ten keys.
`Esc` normal mode; `i a o` insert; `:w :q :wq :q!` (ZZ); `h j k l`; `0 $ gg G`; `w b`; `x dd yy p`; `u` / Ctrl+R; `/text n N`; `:%s/old/new/g`; `:set nu`; `:12`; `:e file`; `vimtutor`. Config habits: copy first (`cp f f.bak`), comment don't delete, test config (`sshd -t`, `nginx -t`, `named-checkconf`) before restart. `dos2unix` fixes `\r\n`.

**Text tools:**
```
sort (-n numeric, -r, -u, -h human, -t: -k3)     uniq -c (needs sorted input)
grep -i -v -r -n -E -c -l                        grep -v "^#" f | grep -v "^$"
cut -d: -f1,7 / cut -c1-10   wc -l/-w/-c   awk '{print $1,$3}'   diff -u
sed 's/old/new/g' f   (sed -i changes in place; try without -i; sed -i.bak keeps copy)
head -1 data.csv | tr ',' '\n' | nl
```
Classic pipeline: `grep x log | cut/awk col | sort | uniq -c | sort -rn | head`. First thing on an inherited server: `grep -r` for old passwords in /etc, /opt, /home, cron.

**Safe-edit routine (Week 1):** `cp file file.bak` > `nano file` > `grep newword file` > `diff file.bak file`. Do it every time on a server.

**Practice answers:** `find /etc -mtime -2`; `find / -type d -perm -0002 2>/dev/null`; `tar -czvf ~/etc.tar.gz /etc` / `tar -tzvf` / extract `etc/hosts` with `-C`; vim `:20` then `:1,40s/yes/no/g` then `:wq`; shells count: `cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn | head -5`; "permission denied" lines are root-only files (shadow, sudoers).

---

## 5. Discovery and Documentation

**Three identifiers:** IP (location, can change), hostname (hint, never proof), MAC (burned in; first 3 pairs = vendor; e.g. B8:27:EB = Raspberry Pi). `192.168.1.0/24` = .0-.255, 254 usable hosts, gateway usually .1. Address+port (`192.168.1.10:443`) identifies one service.

**Ports to know** (full list with extras in §8)

| Port | Service | Implies |
|---|---|---|
| 22 | SSH | Linux or network device |
| 53 | DNS | name server, often DC |
| 80/443 | HTTP(S) | web server or device admin page |
| 88 | Kerberos | Windows DC |
| 135/139/445 | RPC/NetBIOS/SMB | Windows (445 = file sharing) |
| 389/636 | LDAP/LDAPS | directory; with 88 = DC |
| 3389 | RDP | Windows remote desktop |
| 25/587 | SMTP | mail server / alert sender |
| 631 | IPP (CUPS) | Linux print server |
| 3306/5432/1433 | MySQL/Postgres/SQL Server | DB; never reachable externally |

Read **patterns**: 22+80 = Linux web server; 135+445 = Windows; 88+389+53 = domain controller.

**nmap (outside view):** `-sn` ping scan; `-p 22,53,...` chosen ports (all 65,535 takes hours, noisy); `-O` OS guess (sudo); `-sV` versions; `-oN file` save. **Scanning without written permission is illegal in most countries**; scans miss anything off, so also walk the building.

**From the host (inside view):** `ip addr`, `ip route`, `hostnamectl`, `ss -tulpn` (listening + process), `lsblk; df -h`, `free -h; nproc`, `uptime`, `last reboot`. `127.0.0.1` = this machine only (correct for a DB); `0.0.0.0` = all addresses (network-reachable). nmap and ss should agree; if not, a firewall or something hiding. Windows: `ipconfig /all`, `systeminfo`, `netstat -ano | findstr LISTENING`, `Get-NetTCPConnection -State Listen`.

**Documentation = what stays when the person leaves.** Excuses ("I know it", "no time", "makes me replaceable") are all about the admin, not the org. A change isn't done until written down. Good docs: short (one page), dated and signed, in one searchable place.

**Five things to write, in order of regret:** (1) what exists (inventory) (2) how to get in (credentials) (3) how things connect (network diagram) (4) how to fix common things (runbooks) (5) what we promised (service levels). Later: change records (Mod 9), incident/problem (Mod 8), backup/restore (Mod 4), monitoring runbooks (Mod 8).

**Asset list (1 row/host):** Hostname, IP, MAC, Type, OS, Role, Location, Owner, Found by, Last verified, Notes. Fill Type/OS from evidence; fill Role only when known (**blank is honest, a guess is not**); Owner = who depends on it (ask, don't scan); Last verified is a date. Spreadsheet OK to ~50 hosts, then a **CMDB** (adds relationships, Module 9). Example hosts: BUSTER (DC/DNS/DHCP, Server 2012), track01 (Ubuntu 22.04 tracking site), WH2-OLDPC (Windows XP, customs labels, 60 loads/day), unknown 192.168.1.200 (Raspberry Pi MAC, 80/443).

**Credentials:** Never sticky notes, passwords.xlsx, plain text in scripts/configs, email/chat/tickets, or one shared password. Always a password manager (Bitwarden, KeePass, 1Password; vaults like HashiCorp/Azure Key Vault for servers), one account per person, shared secrets with reader list + change date, change when someone leaves. Scripts: env vars or `chmod 600` secrets file, a least-privilege service account (never domain admin), rotate and retest.

**Runbook (one page):** Title (symptom in user words) / Affects (sets priority) / Likely cause / Steps (numbered, exact) / Verify / If that fails (who to call) / Last tested. Test: the 3 a.m. reader shouldn't have to think.

**Lab:** ping scan subnet; port scan the 11 ports (save with `-oN`); build asset list; identify DC (88, 389, plus 53); identify unknown host without logging in (switch port/cable, MAC vendor, browse port 80, DHCP lease name, ask staff); fill own VM row from host commands; write a one-page runbook.

**Reading (2021 Wiley, ISBN 9781119794455):** S1 Ch1 pp3-18; S2 Ch1 pp18-20 + Ch22 pp481-486; S3 Ch6 pp115-133; S4 Ch7 pp135-163, Ch8 pp165-187; S5 Ch12 pp269-277. Bottom Line answers from p505.
**Next:** Module 2: Ubuntu Server and Rocky in VirtualBox, packages (apt, dpkg, dnf, rpm, snap, flatpak), GRUB2, hardware (/dev /proc /sys), post-install checklist (hostname, time, updates, SSH, firewall, asset list).

---

## 6. Weekly Class Decks (Weeks 1-6)

### Week 1: The Shell
Linux = one tree under `/`. Navigate: `pwd`, `cd /etc`, `cd ..`, `cd ~`, `ls`. Read without changing: `cat`, `less` (q), `head`, `tail`, `tail -f`. Find: `find`, `grep` (see section 4). Safe edit: backup > edit > check > diff. Redirect: `>` overwrites, `>>` appends (`echo hello > f` erases; `ls > list.txt`).

### Week 2: Installing a Server and Disk Layout
- Build the server as a **VirtualBox VM** so it never touches the laptop: New > Settings/Storage (attach Ubuntu Server ISO) > Settings/Network (**Adapter 1 NAT** = internet, **Adapter 2 Host-only** = private network) > Start.
- **Partitions** protect the server (full logs on a single partition stop everything): `/` ~15 GB, `/home` ~5 GB, `/var` ~4 GB kept separate. Choose **Custom storage layout** at install.
- **SSH:** tick "Install OpenSSH server", or `sudo apt install openssh-server -y` then `sudo systemctl enable --now ssh`.
- After install: `lsblk`, `df -h`, then VirtualBox Machine > **Take Snapshot** named `clean-install`.

### Week 3: Boot, systemd, SSH
- **Boot stages:** Firmware > GRUB (menu, rescue door) > Kernel > systemd (mounts disks, starts services) > Login. Check which stage stopped.
- **SSH:** `ssh user@192.168.56.10`, `hostname`, `exit`, `ssh-copy-id user@server` (key login, no password).
- **systemd:** `systemctl status ssh`, `start` (now), `stop`, `enable` (every boot), `systemctl --failed`. **start != enable.**
- **Bad `/etc/fstab` line = emergency mode.** Fix: `mount -o remount,rw /` > `sudo nano /etc/fstab` (fix/remove line; add `nofail` for a missing disk) > `mount -a` (no error = safe) > `reboot`.

### Week 4: Users, Groups, Permissions
**Words:** rwx, UID, GID, `-aG` (append to groups, never wipes others), setgid, ACL, NTFS.

| Task | Linux | Windows |
|---|---|---|
| Create user | `sudo useradd -m alice` | `New-LocalUser alice` |
| Password | `sudo passwd alice` | `Set-LocalUser alice -Password (...)` |
| Group | `sudo groupadd dispatch` | `New-LocalGroup dispatch` |
| Add to group | `sudo usermod -aG dispatch alice` | `Add-LocalGroupMember dispatch alice` |
| Show groups | `id alice` | `Get-LocalGroupMember dispatch` |
| See perms | `ls -l file` | `icacls file` |

- Permission line = **owner, group, other**, each rwx (r read, w write, x run/enter). `-rw-r-----` = owner rw, group r, other none. Windows shows an ACL (R, W, RX, M).
- **Numbers:** 7=rwx, 5=r-x, 4=r, 0=none. `chmod 750 file`, `chmod 640 file`, `chmod o-rwx file`, `sudo chgrp dispatch folder`. Windows: `icacls file /grant "dispatch:(RX)"`, `/remove "Users"`; NTFS inherits from parent.
- **Shared team folder:** `sudo chgrp dispatch /srv/manifests; sudo chmod 2775 /srv/manifests`. Leading **2 = setgid** (`drwxrwsr-x`, the `s`): new files keep the folder's group, so teammates aren't locked out. Windows: `icacls D:\manifests /grant "dispatch:(M)"`, inheritance on by default.
- **Prove permissions by testing as the blocked user, never root/Administrator** (they're almost never refused): `sudo su - alice`, `ls /srv/accounts`, `exit`. Windows: `runas /user:alice cmd`, `dir D:\accounts`. "Permission denied" = success.

### Week 5: sudo, Leavers, Active Directory
**Words:** sudo, visudo, PAM, AD, DC, OU, SSH key.
- **sudo** runs one command as root and logs who. Limit a user to one command: `sudo visudo -f /etc/sudoers.d/dispatch`, add `devon ALL=(root) /usr/bin/systemctl restart tracking`. Always use **visudo** (validates before saving). Check admins: `getent group sudo` / `Get-LocalGroupMember Administrators`. Windows has no per-command equal (`runas /user:Administrator ...`, or add to Administrators).
- **Removing a leaver = close every door:**

| Step | Linux | Windows |
|---|---|---|
| Lock password | `sudo passwd -l devon` | `Disable-ADAccount devon` (covers most doors) |
| Block shell | `sudo usermod -s /usr/sbin/nologin devon` | covered |
| Remove SSH key | `sudo rm ~devon/.ssh/authorized_keys` (the forgotten door) | covered |
| End sessions | `sudo pkill -u devon` | `logoff <id>` |
| Drop admin | `sudo deluser devon sudo` | `Remove-ADGroupMember 'Domain Admins' devon` |

- **Active Directory:** central directory of users, groups, computers; run by a **Domain Controller**; **OU** = folder for sorting; AD group = shared role. Linux analogue: LDAP/SSSD. `New-ADOrganizationalUnit -Name Dispatch`.
- **AD onboarding mirrors Linux:** `New-ADUser -Name alice -Enabled $true`, `New-ADGroup dispatch -GroupScope Global`, `Add-ADGroupMember dispatch alice`, `Get-ADGroupMember dispatch`. **Grant the group, not the person.**

### Week 6: Disks, LVM, RAID
**Words:** LVM, PV, VG, LV, RAID, fstab, mount.
- **Blank disk, fixed order: partition > format > mount > save to fstab.** `lsblk`, `sudo mkfs.ext4 /dev/sdb1`, `sudo mount /dev/sdb1 /data`, `echo '/dev/sdb1 /data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab` (**nofail** so a missing disk can't block boot). Windows: `Get-Disk`, `New-Partition | Format-Volume`, drive letter persists.
- **LVM** grows volumes live, no downtime: `sudo pvcreate /dev/sdc` > `sudo vgextend data /dev/sdc` > `sudo lvextend -r -l +100%FREE /dev/data/vol` (`-r` resizes filesystem too). Check `sudo lvs; df -h`. Windows equivalent: Storage Spaces (`Add-PhysicalDisk`, `Resize-VirtualDisk`, `Get-VirtualDisk`, `Get-Volume`).
- **RAID 1** mirrors two disks; survives one failure. `sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc`; health `cat /proc/mdstat` (**[UU]** healthy, **[U_]** one failed, still running); `sudo mdadm --detail /dev/md0`. Windows: mirror via `New-StoragePool ... -ResiliencySetting Mirror`, `Get-PhysicalDisk`.
- **RAID is NOT a backup:** deletions (`rm -rf`), ransomware and fire hit both disks. It buys uptime, not recovery. Need separate, tested backups (next topic).

---

## 7. Quick Cross-Reference

- **Never as root to test permissions**; **never `rm` without reading twice**; **never `>` when you mean `>>`**.
- Before any config edit: backup, edit, test config, diff, then restart.
- Always `nofail` in fstab; snapshot after clean install.
- Close every door for leavers: password, shell, SSH key, sessions, admin rights.
- Group-based access everywhere (Linux groups, AD groups, setgid shared dirs).

---

## 8. Port Numbers: Full Reference

Items marked **(added)** were not in the decks; the rest are from Module 1 Session 5.

| Port | Proto | Service | What it tells you |
|---|---|---|---|
| 21 (added) | TCP | FTP | file transfer, legacy, plain text |
| 22 | TCP | SSH | Linux or network device; secure remote login |
| 23 (added) | TCP | Telnet | legacy plain-text remote login; should be off |
| 25 / 587 | TCP | SMTP | mail server, or device sending alerts by mail |
| 53 | TCP+UDP | DNS | name server, often the domain controller |
| 67 / 68 (added) | UDP | DHCP | address assignment (server / client) |
| 80 / 443 | TCP | HTTP / HTTPS | web server, or device with web admin page (printer, camera, switch) |
| 88 | TCP+UDP | Kerberos | Windows domain controller |
| 110 / 143 (added) | TCP | POP3 / IMAP | mail retrieval (993 / 995 = encrypted versions) |
| 123 (added) | UDP | NTP | time sync |
| 135 / 139 / 445 | TCP | RPC / NetBIOS / SMB | Windows; 445 = file sharing |
| 161 (added) | UDP | SNMP | network device monitoring |
| 389 / 636 | TCP | LDAP / LDAPS | directory; with 88 = domain controller |
| 631 | TCP | IPP (CUPS) | Linux print server |
| 2049 (added) | TCP | NFS | Linux/Unix file sharing |
| 3306 | TCP | MySQL | database; never reachable from outside |
| 5432 | TCP | PostgreSQL | database; never reachable from outside |
| 1433 | TCP | SQL Server | database; never reachable from outside |
| 3389 | TCP | RDP | Windows remote desktop; someone can get a screen |
| 5985 / 5986 (added) | TCP | WinRM (PowerShell Remoting) | Windows remote management; the Windows counterpart to SSH |

**Patterns:** 22+80 = Linux web server · 135+445 = Windows · 88+389+53 = domain controller · 3389 open on a DC = note it.
**Rule:** a port is "a numbered door"; address + port (e.g. `192.168.1.10:443`) = one service on one host. 80/443 = web, 22 = SSH.

**Checking ports**

| Task | Linux | Windows |
|---|---|---|
| What's listening here, and which process | `ss -tulpn` | `netstat -ano \| findstr LISTENING` or `Get-NetTCPConnection -State Listen` |
| Is that remote port open (outside view) | `nmap -p 22,445 host` | same nmap, or `Test-NetConnection host -Port 445` (added) |
| Scan a subnet for live hosts | `nmap -sn 192.168.1.0/24` | same nmap (works on Windows) |
| Service versions / OS guess | `nmap -sV -p 22,80 host`, `nmap -O host` | same |

`127.0.0.1:3306` = only this machine (correct for a DB). `0.0.0.0:22` = reachable from the network.

---

## 9. Linux ⇄ Windows Equivalents (filling the one-sided items)

Rows from the decks keep their original side; the missing side is **(added)** where the deck didn't give it. PowerShell aliases (`ls`, `cat`, `cd`, `pwd`, `cp`, `mv`, `rm`, `man`, `ps`) also work.

### Navigating, files, and help
| Linux | Windows (PowerShell) |
|---|---|
| `pwd` / `cd` / `ls -la` | `Get-Location` / `Set-Location` / `Get-ChildItem -Force` |
| `cat` / `less` | `Get-Content` / `Get-Content f \| Out-Host -Paging` (or `more f`) (added) |
| `head -n 10` / `tail -n 10` | `Get-Content f -TotalCount 10` / `Get-Content f -Tail 10` (added) |
| `tail -f` | `Get-Content f -Wait -Tail 20` |
| `mkdir d` / `touch f` | `New-Item d -ItemType Directory` / `New-Item f -ItemType File` (added) |
| `cp` / `mv` / `rm` | `Copy-Item` / `Move-Item` (also renames) / `Remove-Item` (no recycle bin) (added) |
| `rm -rf dir` | `Remove-Item dir -Recurse -Force` (added) |
| `ln -s a b` (symlink) | `New-Item b -ItemType SymbolicLink -Target a` (added) |
| `ln a b` (hard link) | `New-Item b -ItemType HardLink -Target a` or `mklink /H b a` (added) |
| `file f` / `stat f` | no direct `file`; `Get-Item f \| Format-List *` for metadata (added) |
| `man ls` / `--help` | `Get-Help Get-ChildItem -Full` |
| `man -k` / `apropos` | `Get-Help *keyword*` / `Get-Command *noun*` (added) |
| `which` / `whereis` / `type` | `Get-Command name` (added) |
| `nano` / `vim` | `notepad` (handles Unix line endings now); `dos2unix` fixes `\r\n` on the Linux side |
| `clear` (Ctrl+L) | `cls` / `Clear-Host` (added) |
| `history`, `!!`, Ctrl+R | `Get-History`; Up arrow; Ctrl+R works in PSReadLine (added) |
| `sudo` | Run PowerShell as Administrator (no per-command equivalent) |

### Redirection, pipes, variables
| Linux | Windows (PowerShell) |
|---|---|
| `>` `>>` | same `>` `>>` (PowerShell, added) |
| `2> err.txt`, `> out 2>&1` | same syntax in PowerShell (`*>` redirects all streams) (added) |
| `2>/dev/null` | `2>$null` (added) |
| `cmd \| tee log.txt` | `cmd \| Tee-Object log.txt` (added) |
| `<< EOF` here-doc | here-string `@" ... "@` (added) |
| pipes pass **text** | pipes pass **objects** |
| `echo $HOME`, `env` | `$env:USERPROFILE`, `Get-ChildItem env:` |
| `MYVAR=x` / `export MYVAR` | `$MYVAR = "x"` (session) / `$env:MYVAR = "x"` (env var) (added) |
| permanent vars (`~/.bashrc`, `/etc/environment`) | `setx MYVAR x` or `[Environment]::SetEnvironmentVariable("MYVAR","x","User")` (added) |
| `PATH=$PATH:/opt/tools/bin` | `$env:Path += ";C:\tools"` (added) |

### Text and data tools
| Linux | Windows (PowerShell) |
|---|---|
| `grep text f` | `Select-String text f` (also `findstr`) |
| `grep -r` | `Get-ChildItem -Recurse \| Select-String text` (added) |
| `grep -v` / `-c` / `-i` | `Select-String -NotMatch` / `(...).Count` / case-insensitive by default (`-CaseSensitive` to restrict) (added) |
| `find / -name x` | `Get-ChildItem C:\ -Recurse -Filter x` (added) |
| `find -mtime +7` / `-size +100M` | `Get-ChildItem -Recurse \| Where-Object LastWriteTime -lt (Get-Date).AddDays(-7)` / `Where-Object Length -gt 100MB` (added) |
| `locate` | no built-in index tool; use `Get-ChildItem` or Windows Search (added) |
| `sort` / `sort -u` | `Sort-Object` / `Sort-Object -Unique` (added) |
| `uniq -c` | `Group-Object` (added) |
| `wc -l` | `Measure-Object -Line` (added) |
| `cut -d: -f1` | `-split ":"` or `Select-Object` on named properties (added) |
| `awk '{print $1}'` | `ForEach-Object { ($_ -split '\s+')[0] }` or `Select-Object` (added) |
| `sed 's/a/b/g'` | `-replace 'a','b'` (added) |
| `diff -u a b` | `Compare-Object (gc a) (gc b)` (added) |
| `du -sh /var/*` | `Get-ChildItem \| ForEach-Object {...Measure-Object Length -Sum}` (added) |
| `tar -czvf` / `tar -xzvf` | `tar` is built into Windows 10+; or `Compress-Archive` / `Expand-Archive` (added) |
| `zip -r` / `unzip -l` | `Compress-Archive` / `Expand-Archive` (added) |

### Processes, system info, networking
| Linux | Windows |
|---|---|
| `ps aux` | `Get-Process` |
| `pkill -u devon` | `logoff <id>` (sessions, deck); `Stop-Process -Name x` / `taskkill /IM x` for processes (added) |
| `hostnamectl` | `systeminfo`, `hostname` (added) |
| `ip addr` | `ipconfig /all` |
| `ip route` | `route print` / `Get-NetRoute` (added) |
| `uptime` / `last reboot` | `systeminfo \| findstr /B /C:"System Boot Time"` |
| `free -h` | `systeminfo` (memory) or `Get-CimInstance Win32_OperatingSystem` (added) |
| `nproc` | `$env:NUMBER_OF_PROCESSORS` (added) |
| `lsblk` / `df -h` | `Get-Disk` / `Get-Volume` |
| `ss -tulpn` | `netstat -ano`, `Get-NetTCPConnection -State Listen` |
| `ssh user@host` | `ssh` is built into Windows 10+ (OpenSSH); PowerShell Remoting `Enter-PSSession` (added) |
| `ssh-copy-id` | no equivalent; append the public key to `authorized_keys` manually (added) |

### Services, packages, logs, updates
| Linux | Windows (PowerShell) |
|---|---|
| `systemctl status ssh` | `Get-Service sshd` (added) |
| `systemctl start` / `stop` | `Start-Service` / `Stop-Service` (added) |
| `systemctl enable` (every boot) | `Set-Service name -StartupType Automatic` (added) |
| `systemctl --failed` | `Get-Service \| Where-Object Status -eq Stopped` (added) |
| Services console / `sc.exe` / `Get-Service` | `systemctl` / `systemd` units |
| `apt` / `dnf` (one command for everything) | Windows Update, WSUS, per-app installers; `winget` as a package manager (added) |
| WSUS (central updates) | `unattended-upgrades` on Ubuntu, or a local mirror (added) |
| `journalctl`, `/var/log` | Event Viewer; `Get-WinEvent` / `Get-EventLog` (added) |
| Event Viewer | `journalctl`, `/var/log` |

### Boot and recovery
| Linux | Windows (added) |
|---|---|
| GRUB menu, rescue prompt | Windows Boot Manager; Safe Mode / Recovery Environment (WinRE) |
| fix `/etc/fstab`, `mount -a` | no fstab; drive letters persist by default; `bcdedit` / `bootrec` for boot problems |
| emergency mode: `mount -o remount,rw /` | WinRE command prompt; `sfc /scannow` |

### Permissions, users, directory
| Linux | Windows |
|---|---|
| `ls -l`, `chmod 750` | `icacls file`, `icacls file /grant "dispatch:(RX)"` |
| `chown` (added) | `takeown` / `icacls file /setowner user` (added) |
| setuid audit (`find / -perm -4000`) | no setuid concept; audit with `Get-Acl` for write access by Everyone/Users (added) |
| POSIX ACLs `getfacl` / `setfacl` (added) | NTFS ACL via `icacls` / `Get-Acl` |
| `su - alice` | `runas /user:alice cmd` |
| `sudo visudo` | no equivalent; add to Administrators or `runas` |
| `id alice` | `whoami /groups` (own) or `Get-LocalGroupMember` |
| LDAP / SSSD, directory server | Active Directory, Domain Controller |
| group, shared role | AD group (`Add-ADGroupMember`) |
| Group Policy (added) | closest Linux idea: config management (e.g. Ansible) and `/etc` templates |

### Disk, LVM, RAID
| Linux | Windows |
|---|---|
| `fdisk` / `parted` (added) | Disk Management / `New-Partition` |
| `mkfs.ext4` / `mount` / fstab | `Format-Volume` / drive letter (persists by default) |
| LVM: `pvcreate`, `vgextend`, `lvextend -r` | Storage Spaces: `Add-PhysicalDisk`, `Resize-VirtualDisk`, `Resize-Partition` |
| RAID 1: `mdadm`, `/proc/mdstat` | Storage Spaces mirror: `New-StoragePool ... -ResiliencySetting Mirror`, `Get-VirtualDisk` |
