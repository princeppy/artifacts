# Ubuntu After Install — The First Hour

**Scope:** Laptop, desktop or VPS · 13 steps · ~1 hour · copy-paste ready

> A fresh install is not a finished install. Updates, the package managers nobody
> explains, a security baseline, backups that actually run, drivers, a certificate,
> and the housekeeping that stops the disk filling up six months from now.
> One hour, done properly once.

Steps are marked **[server]** or **[desktop]** where they only apply to one kind of
machine. Everything else applies to any Ubuntu install.

---

## Contents

| # | Step | Applies to |
|---|------|-----------|
| 01 | The first five minutes | all |
| 02 | How apt actually works | all |
| 03 | The packages worth installing on day one | all |
| 04 | Stop being root | server |
| 05 | The security baseline | all |
| 06 | Backups, before you have anything to lose | all |
| 07 | Drivers, firmware and graphics | desktop |
| 08 | Make the desktop yours | desktop |
| 09 | Set up the terminal once | all |
| 10 | A development environment | all |
| 11 | Point a domain at it | server |
| 12 | A web server and a real certificate | server |
| 13 | Let the machine tell you it is unwell | all |

Then: laptop battery · monthly housekeeping · common problems · cheat sheet

---

## Read this first — it boots and looks ready, but five things are missing

All five matter within a week — and on a machine with a public address, within minutes.

- **It is already out of date.** The image was built months ago. Everything on it has security updates waiting.
- **The firewall is installed but off.** `ufw` ships with Ubuntu and does nothing at all until you enable it.
- **There is no backup.** The best moment to configure backups is now, while there is nothing to lose.
- **Proprietary drivers may be missing.** NVIDIA graphics, some Wi-Fi chips and firmware updates all need one more step.
- **A public server is already being scanned.** A fresh VPS starts receiving SSH login attempts within minutes of getting its address — automated, constant and entirely impersonal. With password authentication disabled they are harmless. With it enabled, a weak root password is found in hours.

**One hour from now:** fully patched · firewall on, keys only · backups running · drivers correct · a real certificate · disk staying clean

---

## 01 — The first five minutes

*Three commands and one reboot, before anything else.*

```bash
# 1. bring the system fully up to date
sudo apt update
sudo apt upgrade -y

# 2. firmware updates - laptops especially
fwupdmgr refresh
fwupdmgr get-updates
fwupdmgr update

# 3. reboot if a new kernel arrived
ls /var/run/reboot-required && sudo reboot
```

**Why this order**

- The image was built months ago — assume dozens of security fixes are pending
- `apt update` refreshes the package lists; it installs nothing on its own
- `apt upgrade` is what actually installs the new versions
- Firmware updates fix battery, thermal and SSD bugs that look like Linux problems
- A new kernel only takes effect after a reboot

### On a VPS, the same four commands at first login

```bash
ssh root@203.0.113.10

apt update && apt upgrade -y

hostnamectl set-hostname web01
timedatectl set-timezone Asia/Bahrain

ls /var/run/reboot-required && reboot
```

**Check while you are there**

| Command | What it confirms |
|---|---|
| `ip a` | The public address matches the panel |
| `df -h` | How much disk you actually have |
| `free -h` | How much RAM, and whether swap exists |
| `ss -tulpn` | What is listening before you changed anything |
| `cat /etc/os-release` | The release you asked for |

> Name the host something you will recognise at 3 a.m., not `server1`. Thirty seconds of sanity checking confirms the machine you are paying for is the machine you ordered.

> Most VPS images ship with **no swap**. On a 2 GB machine that is worth fixing before you run a database — a 2 GB swap file costs nothing and stops the out-of-memory killer ending your database at 3 a.m.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## 02 — How apt actually works

*Six subcommands cover everything. Two of them are constantly confused with each other.*

| Command | What it does |
|---|---|
| `apt update` | Refreshes the list of available packages. Installs nothing |
| `apt upgrade` | Installs newer versions of what you already have |
| `apt full-upgrade` | Same, but will remove packages if needed to resolve dependencies |
| `apt install NAME` | Installs a package and its dependencies |
| `apt remove NAME` | Removes the package, keeps its config files |
| `apt purge NAME` | Removes the package and its config files |
| `apt autoremove` | Deletes dependencies nothing needs any more |
| `apt search` / `apt show` | Find a package, or read what it actually is |

> `update` and `upgrade` are not the same thing. `update` asks "what is available?" and `upgrade` says "install it". Running `upgrade` without `update` first means upgrading against a stale list — which is why every guide chains them with `&&`.

> Keep `full-upgrade` for release jumps rather than routine updating — letting apt remove a package is a decision worth making deliberately.

---

## 03 — The packages worth installing on day one

*One command. Everything here earns its place within the first week.*

```bash
sudo apt install -y \
    build-essential git curl wget vim nano \
    htop btop net-tools dnsutils traceroute \
    tree unzip zip p7zip-full \
    ufw gnupg ca-certificates \
    tlp gnome-tweaks ubuntu-restricted-extras
```

| Group | Packages |
|---|---|
| Build & dev | build-essential, git, curl, wget |
| Monitoring | htop, btop — see what is using the CPU |
| Network | net-tools, dnsutils, traceroute — ip, dig, ss |
| Archives | unzip, zip, p7zip-full — open anything |
| Security | ufw, gnupg, ca-certificates |
| Desktop | gnome-tweaks, restricted-extras (codecs, fonts) |

> On a server, drop the last line — `tlp`, `gnome-tweaks` and the restricted extras are desktop packages.

---

## 04 — Stop being root **[server]**

*Every action attributed, every mistake recoverable. Every guide from here on assumes it.*

```bash
# create a user for yourself
adduser deploy
usermod -aG sudo deploy

# give it your SSH key
rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy

# test it from a SECOND terminal
ssh deploy@203.0.113.10
sudo whoami        # -> root
```

**Why this matters**

- Root has no undo. `sudo` makes you type an extra word before anything destructive
- Every sudo command is logged with the user who ran it
- Disabling root login removes the account every bot on the internet tries first
- Adding or removing a colleague becomes one command, not a shared password change

> **Keep the first session open.** Do not close the root window until you have logged in as the new user *in a second terminal* and confirmed `sudo whoami` returns root. This is the single most common way people lock themselves out of a brand new server.

---

## 05 — The security baseline

*Keys, three config lines and a firewall in the right order. The highest-value fifteen minutes of the whole build.*

### Keys instead of passwords

```bash
# on your own machine
ssh-keygen -t ed25519 -C 'you@laptop'
ssh-copy-id user@server
```

```bash
# then, on the server - the three lines
sudo nano /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes

sudo sshd -t              # test the config FIRST
sudo systemctl reload ssh
```

> `sshd -t` is not optional ceremony. It catches the typo that would otherwise lock you out the moment you reload.

### The firewall, in this order

```bash
# SSH first, then the web ports, then enable
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

> **Allow SSH before you enable ufw.** Running `ufw enable` without an SSH rule disconnects you immediately and permanently — the firewall denies inbound by default, including the session you are typing in. Recovery means the provider's web console. Every admin does this exactly once; do it on a lab machine rather than a real one.

> Your provider's firewall — Hetzner, DigitalOcean, AWS security groups — sits **in front of** the machine, and `ufw` can neither see nor override it. If `curl` works on the server but not from your laptop, that outer layer is why. Use both: defence in depth costs nothing here.

### Bots, and patches that land by themselves

```bash
# jail the ones knocking
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd

# see what you are protected from
sudo grep 'Failed password' /var/log/auth.log | wc -l
```

```bash
# security patches applied automatically
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
systemctl status unattended-upgrades
```

> unattended-upgrades belongs beside the firewall, not after it: a firewall does not help if the service behind it is months out of date.

**And the four habits behind all of it**

- Never log in as root — use sudo
- One user account per person, never shared
- Full disk encryption on anything portable
- Keep a password manager, not a text file

---

## 06 — Backups, before you have anything to lose

*The one task everybody postpones until the week after they needed it.*

**The 3-2-1 rule**

- **3** copies of anything you care about
- **2** different kinds of media or device
- **1** copy somewhere else entirely — offsite or cloud
- A backup you have never restored is a hypothesis, not a backup

### Snapshots and backups are not the same thing

| | A provider snapshot | File-level backup |
|---|---|---|
| What it is | Whole-disk image of the machine | Versioned copies of the files you choose |
| Good for | Restoring the entire server in minutes, before a risky change | "I deleted one file yesterday", offsite copies, deduplication |
| Useless for | Getting one directory back without rolling the whole machine back | Bare-metal recovery of a broken upgrade |

> Turn on the provider's automatic snapshots when you create the server — they usually cost about 20% and are the fastest possible recovery from a broken upgrade. Then add file backups for what snapshots cannot do.

```bash
# built in, graphical, good enough for a laptop
sudo apt install -y deja-dup
```

```bash
# versioned, encrypted, scriptable
sudo apt install -y restic
restic init --repo /mnt/backup
restic -r /mnt/backup backup /home/you /var/www /etc
restic -r /mnt/backup snapshots
restic -r /mnt/backup forget \
  --keep-daily 7 --keep-weekly 4 --prune

# dump the database, do not copy its live files
mysqldump -u root -p --all-databases > dump.sql

# and the step nobody does
restic -r /mnt/backup restore latest --target /tmp/test
```

> Put a restore in your calendar once a quarter. Restoring one random file proves the whole chain works — and it is the only proof that counts.

---

## 07 — Drivers, firmware and the graphics question **[desktop]**

*Ubuntu handles most hardware automatically. These are the exceptions.*

| Area | Command | Notes |
|---|---|---|
| Additional Drivers | `ubuntu-drivers devices` | Lists proprietary drivers Ubuntu can install for you — mostly NVIDIA and some Wi-Fi chips |
| NVIDIA graphics | `sudo ubuntu-drivers autoinstall` | Installs the recommended driver. Reboot afterwards, then check with `nvidia-smi` |
| Firmware updates | `fwupdmgr refresh && fwupdmgr update` | Vendor firmware for SSDs, docks, keyboards and laptops |
| Printers and scanners | CUPS is already running | Most modern printers work over the network with no driver at all |

> **Wayland or X11?** Ubuntu defaults to Wayland, which is smoother on most hardware. If you hit screen-sharing or NVIDIA oddities, pick "Ubuntu on Xorg" from the gear icon on the login screen — it is a per-session choice, not a permanent one.

---

## 08 — Make the desktop yours **[desktop]**

*Fifteen minutes of settings that you will feel every day afterwards.*

| Setting | Detail |
|---|---|
| GNOME Tweaks | Window buttons, fonts, animations, startup apps |
| Extensions | Dash to Panel, Clipboard Indicator, Vitals — via extensions.gnome.org |
| Keyboard shortcuts | Bind a key to open the terminal. You will use it constantly |
| Night light and scaling | Settings > Displays. Fractional scaling for HiDPI screens |
| Workspaces | Super + Page Up / Page Down. The fastest habit to pick up |
| Default applications | Settings > Apps. Set browser, mail and terminal once |

> Two shortcuts worth learning immediately: **Super** opens the activities overview, and **Super + A** shows every installed application. Almost nobody needs a desktop icon again after that.

---

## 09 — Set up the terminal once, use it for years

*Aliases and a prompt you can read save minutes every single day.*

```bash
# ~/.bashrc - add at the end
alias ll='ls -alF'
alias ..='cd ..'
alias update='sudo apt update && sudo apt upgrade -y'
alias ports='sudo ss -tulpn'
alias myip='curl -s ifconfig.me'

export HISTSIZE=10000
export HISTCONTROL=ignoreboth:erasedups

source ~/.bashrc
```

**Worth the fifteen minutes**

- Keep your dotfiles in a git repository — a new machine is then one clone away
- Learn five tmux commands if you work over SSH: sessions survive a dropped connection
- Ctrl+R searches your command history. It is the single best terminal shortcut
- Tab completion works on commands, paths, package names and git branches
- Add `~/bin` to PATH and put your own scripts there

> Zsh with oh-my-zsh is popular and pretty. Bash is what every server you ever log into actually has. Learn bash first; add zsh on your own machine afterwards if you want it.

---

## 10 — A development environment in ten minutes

*If the machine is for work, these four steps come before anything else.*

```bash
# compilers and headers
sudo apt install -y build-essential

# git, configured properly
sudo apt install -y git
git config --global user.name 'Your Name'
git config --global user.email 'you@example.com'
git config --global init.defaultBranch main

# an SSH key for GitHub or GitLab
ssh-keygen -t ed25519 -C 'you@example.com'
cat ~/.ssh/id_ed25519.pub
```

```bash
# Docker, the official way
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world

# languages: use a version manager, not apt
# node   -> nvm
# python -> pyenv or python3-venv
# rust   -> rustup
```

> Do not install Node or Python versions from apt if you develop against specific versions. Version managers let each project use what it needs, and they never fight the system packages that Ubuntu itself depends on.

> Adding yourself to the `docker` group is what stops you prefixing every container command with `sudo`. It normally applies at your next login — `newgrp docker` is what makes it apply now.

---

## 11 — Point a domain at it **[server]**

*Two records and one wait. This is the step that turns an IP address into something people can use — and the one certbot depends on.*

| Type | Name | Value | Notes |
|---|---|---|---|
| A | @ | 203.0.113.10 | The root domain |
| A | www | 203.0.113.10 | Or a CNAME to the root — either works |
| AAAA | @ | 2a01:4f8::1 | Only if your VPS has IPv6 — most do |
| CNAME | blog | example.com. | Subdomains pointing at the same host |

```bash
dig +short example.com        # what the world sees
dig @1.1.1.1 example.com      # bypass your local resolver
curl -I http://example.com    # does it reach your server?
```

> Set the TTL low — 300 seconds — before you change anything, and raise it once things are stable. A mistake then costs five minutes instead of a day. Propagation is usually minutes, occasionally hours.

---

## 12 — A web server and a real certificate **[server]**

*Ten minutes from a bare server to a padlock in the address bar. Ports 80 and 443 were opened in step 05.*

```bash
sudo apt install -y nginx
systemctl status nginx

# your site's files
sudo mkdir -p /var/www/example.com/html
echo '<h1>It works.</h1>' | \
  sudo tee /var/www/example.com/html/index.html

sudo nano /etc/nginx/sites-available/example.com
sudo ln -s /etc/nginx/sites-available/example.com \
           /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

```bash
sudo apt install -y certbot python3-certbot-nginx

sudo certbot --nginx -d example.com -d www.example.com

# certbot edits the config and reloads nginx for you
sudo certbot certificates
sudo certbot renew --dry-run
systemctl list-timers | grep certbot
```

> **When certbot fails, it is almost never certbot.** Ninety-nine times out of a hundred it is DNS not pointing here yet, or port 80 blocked in one of the two firewalls — the machine's or the provider's. Fix those before you touch certbot.

> Renewal is a systemd timer certbot installs for you. `renew --dry-run` is how you confirm today that it will still work in three months — and an expiring certificate is one of the few things worth alerting on.

---

## 13 — Let the machine tell you it is unwell

*So you find out before a customer does.*

```bash
# the four numbers that matter
df -h && free -h && uptime
systemctl --failed
```

```bash
# a dashboard in ten minutes
curl -fsSL https://get.netdata.cloud/kickstart.sh | sh

# do not expose it to the internet
sudo ufw allow from YOUR.IP.ADDRESS to any port 19999
```

**Alert on what users actually feel**

- The site does not answer
- Disk over 85% full
- A certificate expires in under 14 days
- A service is in a failed state

> Layer a free external uptime check on top — UptimeRobot, Better Stack or Healthchecks.io. A check that runs on the machine cannot tell you the machine is down.

---

## Laptops — battery life and thermals **[desktop]**

*Ubuntu is good on laptops now. Three settings make it noticeably better.*

| Tool | Command | Notes |
|---|---|---|
| TLP | `sudo apt install -y tlp && sudo tlp start` | Automatic power tuning. Usually worth 30–60 minutes of battery on its own |
| powertop | `sudo powertop --auto-tune` | Finds the components keeping the CPU awake. Run it once and read the tunables tab |
| Power mode | Settings > Power > Power Saver | Built into GNOME, works with the firmware. Balanced is the sane default on AC |
| Swappiness | `sudo sysctl vm.swappiness=10` | Less aggressive swapping on a machine with 8 GB or more. Make it permanent in `/etc/sysctl.conf` |

> Check what is actually draining the battery before you tune anything: powertop shows the top offenders, and it is often a browser tab rather than the operating system.

---

## Housekeeping — the maintenance that stops the disk filling up

*Five commands, once a month. Skipping them is why people think Linux "gets slow".*

```bash
# where has the space gone?
df -h
du -h --max-depth=1 /var | sort -rh | head

# remove packages nothing needs any more
sudo apt autoremove --purge
sudo apt clean
```

```bash
# the journal grows forever by default
journalctl --disk-usage
sudo journalctl --vacuum-time=2weeks

# snaps keep old revisions
snap list --all
sudo snap set system refresh.retain=2
```

**The monthly five minutes**

- `sudo apt update && sudo apt upgrade`
- `sudo apt autoremove --purge`
- `journalctl --vacuum-time=2weeks`
- Check that the backup actually ran

---

## When something is off — common problems in the first week

*Almost all of them have a one-line answer.*

| Symptom | Most likely cause | Fix |
|---|---|---|
| Locked out right after `ufw enable` | No SSH rule existed when the firewall came up | The provider's web console; allow OpenSSH first next time |
| curl works on the server, not from outside | The provider's cloud firewall in front of it | Open the port there too — ufw cannot |
| certbot validation fails | DNS not pointing here, or port 80 blocked | Fix those two; the problem is not certbot |
| No sound, or the wrong output | Wrong default device | Settings > Sound, pick the output |
| Screen tearing or stutter | Graphics driver or Wayland | `ubuntu-drivers autoinstall`, or try Xorg |
| Wi-Fi drops on resume | Power saving on the wireless card | Disable `power_save` in NetworkManager |
| "Could not get lock" from apt | Another update is already running | Wait, or check for unattended-upgrades |
| Snap apps start slowly | First launch decompresses the snap | Use the apt version where one exists |
| Disk filling up quickly | Journal, snaps or old kernels | Vacuum the journal, `apt autoremove` |
| Fonts look wrong in one app | Missing Microsoft fonts | `sudo apt install ubuntu-restricted-extras` |

> And the general rule: read the actual error before searching for it. Ubuntu's messages are usually specific, and the first line is almost always the useful one.

---

## Keep this page — the complete command cheat sheet

*Everything from this guide, grouped by when you need it. You will not remember it all on day one — you only need to know where to look.*

**Update**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt full-upgrade
fwupdmgr refresh && fwupdmgr update
ls /var/run/reboot-required
```

**Packages**

```bash
sudo apt install -y NAME
sudo apt purge NAME
apt search KEYWORD
apt show NAME
```

**SSH**

```bash
ssh-keygen -t ed25519
ssh-copy-id user@server
sudo sshd -t
sudo systemctl reload ssh
```

**Firewall**

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp && sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

**DNS**

```bash
dig +short example.com
dig @1.1.1.1 example.com
curl -I http://example.com
```

**HTTPS**

```bash
sudo certbot --nginx -d example.com
sudo certbot certificates
sudo certbot renew --dry-run
systemctl list-timers | grep certbot
```

**Drivers**

```bash
ubuntu-drivers devices
sudo ubuntu-drivers autoinstall
nvidia-smi
lspci -nn | grep -i net
```

**Backup**

```bash
restic init --repo /mnt/backup
restic -r /mnt/backup backup /home/you
restic -r /mnt/backup snapshots
restic -r /mnt/backup restore latest
```

**Cleanup**

```bash
sudo apt autoremove --purge
sudo apt clean
journalctl --vacuum-time=2weeks
df -h && du -h -d1 /var
```
