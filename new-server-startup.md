This is a basic startup guide for creating a reasonably secure at-home server with a DNS connection and file backup
## Create USB
- Ventoy install
	- [ventory website](https://www.ventoy.net/en/download.html)
	- download
	- verify hash: `sha256sum ./ventoy-1.1.18-linux.tar.gz`
	- extract: `tar -xf ventoy-1.1.18-linux.tar.gz`
	- Device is probably /dev/sda. unplug usb `lsblk`, plug in usb `lsblk` your device is the one that changed
	- run `./Ventoy2Disk.sh -i /dev/sda` (`-u` to update)
- OS (I am using CachyOS, its familiar and quite good)
	- get [iso](https://cachyos.org/download/)
	- drop into usb drive
	- plug in & install through bios **pick a good password, bots try and guess it**
	- make sure ufw running: `ufw status`, and set port 22, 443 and 80 to open (`ufw status, ufw deny ufw allow ufw delete`).
	- make sure sshd is running: `systemctl status sshd` if not `systemctl enable --now`
	- note the local router IP address `ip -r`.
## ssh in from other machine
- [source](https://computingforgeeks.com/configure-ssh-server-on-arch-manjaro-endeavouros/)
- `ssh user@ip` then exit. Generate a ssh credential: `ssh-keygen`. Move public ssh credential to server: `ssh-copy-id user@ip`. May want to backup private key.
- check cert working (so you don't lock yourself out): `ssh -o BatchMode=yes user@ip `echo hi`
- On server create `/etc/ssh/sshd_config.d/10-hardening.conf` : 
```
PermitRootLogin no
PasswordAuthentication no # avoid non-stop bot password guessing
KbdInteractiveAuthentication no
MaxAuthTries 5
```
- confirm settings (so you don't lock yourself out): `sshd -T | grep -Ei "(root|pass|pubkeyauth|port)"`
- restart sshd: `systemctl restart sshd`
## Run forever
- `/etc/systemd/sleep.conf` - turn off suspend, sleep and hibernate
```
AllowSuspend=no
AllowHibernation=no
```

- `/etc/systemd/login.conf` - disable lid sensor
```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
HandleSuspendKey=ignore
HandleHibernateKey=ignore
```

- maybe: `systemctl set-default multi-user.target`
- make sure screen turns off after a bit. My ones screen is broken (hence why it is now a server)
## Double check ports
- `ss -tlnp`
- note docker will go around firewalls if you put something on 0.0.0.0
## install syncthing
- may also want to consider rsync
- both computers: `pacman -S syncthing && systemctl --user enable --now syncthing.service`
- if it fails: See section "How to set up a user service" [here](https://docs.syncthing.net/users/autostart.html#linux)
- forward the local syncthing GUI to your machine: `ssh -L 8385:localhost:8384 user@ip`.
	- your machines settings are now viewable on `http://<ip>:8384` and the servers is on `http://<ip>:8385` so you can set it up.
	- recommend: using .stignore and two backups. An encrypted-on-server folder-sync shared between multiple non-server computers. A non-encrypted-on-server folder-sync.
	- recommend: staggered file versioning.
## Install docker & friends:
- May need to turn on virtualization in the BIOS
- `pacman -S docker docker-buildx docker-compose && systemctl enable --now docker`
- Add yourself to the docker user group: `usermod -aG docker <user>`. May need to create the docker group `groupadd docker`
- log out and back in for group to take effect 
## namecheap & cloudflare in brief
- Get a domain name ([namecheap](https://www.namecheap.com/myaccount/login/))
- Set it up with cloudflare (free), click the option to prevent email spoofing.
## caddy: see note's [basic_caddy guide](./basic_caddy)
- not getting to server? `curl -4 ifconfig.me` is probably your router.
	- For me AT&T Router Passthrough:
		- video of the page
		- permanent mac:
			- I think some routers need a permanent mac address while linux tends to randomize it.
			- List networks - `nmcli -o -p con`
			- Set permanent mac `nmcli -p con mod <wifi-name> 802-11-wireless.cloned-mac-address permanent`
			- `systemctl restart NetworkManager`
- Once this is working you should be able to ssh in using the url.
- Will go more into docker and caddy later.
## Additional Security
- I have seen recommendations on arch forums that all updates should be run manually. That seems a bit much.
- With [black october](https://en.wikipedia.org/wiki/Black_October_(information_security)) and other vulnerabilities found by AI, I recommend automatic updateing to get patches quickly.
- [Source](https://www.reddit.com/r/archlinux/comments/o87muy/one_year_of_unattended_updates_yielded_9996/)
- create script: `/root/bin/auto-update.sh` (then `chmod +x /root/bin/auto-update.sh`):
```bash
#!/bin/bash
date >> /root/auto-update.log
out=$(pacman -Syu --noconfirm 2>&1)
echo $out >> /root/auto-update.log
# reboot if the word reboot occurs in the results
grep -qi reboot <<< "$out" && reboot
```
- create service: `/etc/systemd/system/auto-update.service`
```
[Unit]
Description="auto update with log"

[Service]
ExecStart=/root/bin/auto-update.sh
```
- create timer: `/etc/systemd/system/auto-update.timer`
```
[Unit]
Description=Auto update written by luke

[Timer]
# Seem like good times
OnCalendar=Mon,Thu 05:23:13
Unit=auto-update.service

[Install]
WantedBy=timers.target
[root@CachyServer system]
```
- then:
```bash
systemctl daemon-reload
systemctl enable --now auto-update.timer
```
- finally: keep an eye out for that log file!