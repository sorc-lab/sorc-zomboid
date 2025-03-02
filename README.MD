# SORC_ Project Zomboid Dedicated Server

## Operating System
Ubuntu 24.04.2 LTS with OpenSSH enabled through installation setup, or done manually.

## Basic Security Configuration for Internal Network Access

### Install Updates and Upgrade to Latest Ubuntu LTS Version
Repeat these commands as necessary until latest version has been installed:
```
$ sudo apt update -y && sudo apt upgrade -y
$ sudo do-release-upgrade
$ sudo reboot

# rinse and repeat until at latest version
```

### Free Disk Space in '/boot' for Kernel Upgrades
Run 'apt autoremove' to run an auto remove of old kernels:
```
$ sudo apt autoremove
```

Set 'COMPRESS' to use 'xz':
```
$ sudo vim /etc/initramfs-tools/initramfs.conf

# Edit the COMPRESS field:
COMPRESS=xz
```

Manually remove old kernels:
⚠️**NOTE: Take note of currently installed kernel.**
```
$ uname -r
```
List installed kernels:
```
$ dpkg --list | grep linux-image
```
Manually remove old kernels:
```
$ sudo apt remove --purge linux-image-<old-version>
```
⚠️**NOTE: DO NOT remove the currently installed kernel listed in previous `uname -r` command!**

After removing old kernels, update GRUB:
```
$ sudo update-grub
```
Regenerate the initramfs to clear up space:
```
$ sudo update-initramfs -u
```
Check space usage in '/boot':
```
$ sudo du -sh /boot/*
```
⚠️**NOTE: You can also remove any old initrd.img-* or vmlinuz-* files that don't match your current kernel via `sudo rm -f /boot/initrd.img-<old-version>-generic` etc.**

Clear old package versions:
```
$ sudo apt clean
```


### Setup Static IP Address
- Find default gateway via `ip route` command, e.g. `default via 192.168.64.1 dev enp3s1`
- Identify network interface via `ip a` command, e.g. `eth0`, `eno1`, or `enp3s1`, etc. Find by locating current IP address in list of network interfaces to locate server's interface and take note of it:
```
eno1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.64.231  netmask 255.255.255.0  broadcast 192.168.64.255
        ...
```
⚠️**NOTE: Everything mentioned above will be different for your specific hardware and configurations.**
- Configure a static IP address using `Netplan`:
```
$ sudo vim /etc/netplan/01-netcfg.yaml
```
- Write the following configuration to file (adjusted to desired static IP):
```
network:
  version: 2
  renderer: networkd
  ethernets:
    eno1:
      dhcp4: no
      addresses:
        - 192.168.64.100/24
      gateway4: 192.168.64.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```
- Save and apply changes (e.g. will update dynamic `192.168.64.231` to `192.168.64.100` static IP):
```
$ sudo netplan apply
```
⚠️**NOTE: This command can result in a failure message but should fall back to using 'systemd-networkd.service' to restart the network daemon. Double check with the 'ip a' command to verify the IP address changed.**

### Setup SSH with Public and Private Keys for Remote Access
On local machine that will remote connect to sever, generate ed25519 SSH Key Pair:
```
$ ssh-keygen -t ed25519 -C "your_email@example.com"
```
- When prompted, name the file anything you want, e.g.: `id_ed25519_sorc-zomboid`
⚠️**NOTE: It will ask for a passphrase, but this is optional.**
- Copy or move both the private (`id_ed25519_sorc-zomboid`) and public keys (`id_ed25519_sorc-zomboid.pub`) into a `.ssh` directory in your home directory `~/.ssh` or `C:\Users\<your_user>\.ssh`
- On Linux, set permissions on private key file so only your user can read ():
```
$ chmod 600 ~/.ssh/id_ed25519_sorc-zomboid
```
- ⚠️**NOTE: On Windows, you can set permissions to make sure only your user can access, if sharing PC with other individuals.**

On Linux server, create a `~/.ssh` directory with permissions:
```
$ mkdir -p ~/.ssh
$ chmod 700 ~/.ssh
```

From remote PC, copy public key to Linux server:
```
scp -rp C:\Users\<your_windows_user>\.ssh\id_ed25519_sorc-zomboid.pub <linux_user>@<server_ip>:~/.ssh/.
```
⚠️**NOTE: You will be asked if you trust the host, say yes.**
⚠️**NOTE: You will be prompted for the password for the Linux user. Password prompt will be disabled in later steps.**

On Linux server, write public key(s) into an `authorized_keys` file and set permissions:
```
$ touch ~/.ssh/authorized_keys
$ cat ~/.ssh/id_ed25519_sorc-zomboid.pub >> ~/.ssh/authorized_keys
$ rm ~/.ssh/id_ed25519_sorc-zomboid.pub
$ chmod 600 ~/.ssh/authorized_keys
$ chown your_user:your_user ~/.ssh ~/.ssh/authorized_keys
```

On Linux server, secure SSH configuration and disable password login:
```
$ sudo vim /etc/ssh/sshd_config

# Edit or add the following:
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
MaxAuthTries 3

# NOTE: Some lines above exist. Search vim via '/'. Some lines will need to be added, if not existing.
```

Restart SSH daemon:
```
sudo systemctl restart sshd
```

On local machine, add SSH private key(s) to Windows `ssh-agent`:
```
Set-Service -Name ssh-agent -StartupType Automatic
Start-Service ssh-agent

ssh-add C:\Users\your_user\.ssh\id_ed25519_sorc-zomboid
```
⚠️**NOTE: Run commands via Windows Terminal in Admin mode.**

Finally, connect via: `ssh your_user@192.168.64.100`.


### Configure UFW for SSH Access
```
$ sudo ufw allow 22/tcp
$ sudo ufw enable
$ sudo ufw status
```

### Install and Configure Fail2Ban





# 👇 Original Docs from Forked Repo

---

# Project Zomboid dedicated server container

This Dockerfile downloads the Project Zombie dedicated server using SteamCMD. Will allow you to deploy with ease a server in Kubernetes, Docker...

The ready to use image is available here:

https://hub.docker.com/r/danixu86/project-zomboid-dedicated-server

**WARNING:** Running the image on Windows using WSL2 in Docker Desktop will make the server startup times significantly slower. This is fine unless you are running alot of mods in which case server startup times may vary from 15 - 40 minutes. The best way to solve this problem is to run your container on a Linux based system or in a Linux VM.

## Environment variables

This dockerfile converts some env variables to arguments in the server command, allowing to configure it. The image supports the following environment variables:

`ADMINUSERNAME:` Sets the administrator username, otherwise `admin` will be used.

`ADMINPASSWORD:` Sets or changes the admin password. Is mandatory at first startup or will fail. I recommend to remove it once the server is started because the server logs every argument in clear text (if doesn't cares, you can leave it).

`PASSWORD:` Sets or changes server password. Is optional.

`CACHEDIR:` Set the folder where the data will be stored. By default the server stores the data in the Zomboid folder in home directory (/home/steam/Zomboid). Is recommended to store this data into a persistent volume to keep the server state.

`DEBUG:` Enable the debug mode in server

`FORCEUPDATE:` Force a server update on every start. This process can be slow if the image is old.

`IP:` Set the interface IP where the server will listen. By default all the interfaces.

`MEMORY:` Amount of memory to use in the server JVM (units can be used, for example 2048m). By default 8096m.

`MODFOLDERS`: Mod folders to use to load the game mods. Allowed options are workshop, steam, and mods. Options must be separated by comma, for example: workshop,steam,mods

`NOSTEAM:` Starts the server in NoSteam mode, allowing to non steam users to connect.

`PORT:` Set the server port. By default 16261

`SERVERNAME:` Set the server name, which allows you to have multiple server versions. Changing the name will create a new server data, so current progress will not be available. Also will not be lost, just reverting the server name change will load it again. **NOTE: Don't use spaces or special characters or admin user will fail**

`SERVERPRESET:` Set the new servers default preset. If not set, the default is apocalypse. Allowed options are Apocalypse, Beginner, Builder, FirstWeek, SixMonthsLater, Survival and Survivor.

`SERVERPRESETREPLACE:` Replace the server preset by the one seted by the SERVERPRESET variable. By default false to avoid to replace custom configurations once the server has started.

`SOFTRESET:` Performs a soft reset in the server. More info in the Soft Reset section.

`STEAMPORT1 & STEAMPORT2:` Sets the two additional ports required for steam to work. I guess that those ports are UDP, but it can be also TCP.

`STEAMVAC:` Enables or disables de SteamVac protection in server

`WORKSHOP_IDS:` Semicolon separated list of Workshop IDs to install to the server

`MOD_IDS:` Semicolon separated list of Mod IDs to install to the server

`LANG:` This env variable will set the container locales, which will be used by the pz server to set the language. Example (en_EN.UTF-8, es_ES.UTF-8...)

Mods will install on second server start

Maps are added on third server start, after mods are added
## Soft Reset

Soft reset will perform the following tasks:

* Removes items from the ground.
* Removes items from containers and replaces them with new loot. This includes player-made containers.
* Removes corpses and zombified players.
* Resets zombies.
* Removes blood splatter.
* Resets building alarms.
* Resets the game clock. I’m not positive if this resets to day 1, but the original blog post implies that it does. So this would bring water and electricity back. Assuming your server settings have them * available at day 1.
* Player-made buildings won’t be deleted.
* Player inventories won’t reset.

Removing the following files in the Server data folder (ToDo: I have to search the equivalent files in modern version):

* all zombie_X_Y.bin files
* map_t.bin
* map_meta.bin
* reanimated.bin

Currently is not working correctly and will cause the server to crash when it is active. Also the above points are not performed so nothign will change in the server.

## Required ports

The required ports are docummented in the [Official wiki](https://pzwiki.net/wiki/Dedicated_Server#Forwarding_Required_Ports)

For now the ports are the following, but it can change.

### Steam clients ports

This ports are used by the Steam clients to connect with the server, so it must be opened in firewall and configured in NAT.

* 8766 UDP
* 8767 UDP
* 16261 UDP (configurable)

### Non Steam clients ports

This ports are used by the No Steam clients to connect with the server, so it must be opened in firewall and configured in NAT.

* 8766 UDP
* 8767 UDP
* 16261 UDP (configurable)
* 16262 - 16272 TCP (range depends of the above port and the clients slots)

### Notes

* The 16261 port can be changed using the PORT environment variable. This will affect at the TCP ports range, changing it to the next port. For example, if PORT is set to 12234, the TCP range will be 12234 - 12244.
* The TCP ports range is used by the clients, so every client slot will need a port. If your server plans to host 50 clients, then you will need to open 50 ports in firewall/nat... (16262 - 16312).
* The Wiki says that Non Steam ports is supposed to be for non steam clients, but I am using the steam client and those ports are used too.

## How to use the image

To setup the server you can use the docker-compose program:

* Copy the .env.template to .env and edit it with the settings you want. The only mandatory variable is ADMINPASSWORD and just the first time the server is executed (once is started at least once, it can be removed). The other variables are optional.
* Run the `docker-compose up -d` command to start the server.

## Scripts folder

In the scripts folders there are three scripts:

* entry.sh: The script used in the image to start the server
* search_folder.sh: Script to search through mod folders for any maps 
* update_forum.sh: Check the posts in the forum for updates and create a new image if a new version exists. This script was made because looks like the official page doesn't update the version as soon it is out.
* uodate_web.sh: Check the version in the official page and create a new image if a new version exists
