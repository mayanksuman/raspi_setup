## Enable ssh without using a screen 

For enabling ssh without a screen make an empty file in `boot` partition of raspberry pi SD card with a name `ssh`. This can be done by touch command. Navigate to `boot` partition of the SD card and issue following command:
```
touch ssh
```
The default user is `pi` and default password is `raspberry`.

## Finding your raspberry pi on network

To login into raspberry pi using ssh, we need its IP address. If you are connected to a DHCP server and you know the IP range and its subnetmask then you can have static IP on raspberry pi. For the purpose you need to edit `etc/dhcpcd.conf` on `root` partition of raspberry SD card. Mount the required partition on PC and edit the file to add following to it:
```
interface eth0
static ip_address=10.103.15.100/24
static routers=10.103.15.2		# For internet to work
static domain_name_servers=10.103.15.1 8.8.8.8 fd51:42f8:caae:d92e::1 # For internet to work
```
change the `ip_address` to whatever you need, then the raspberry can be sshed using `ssh pi@ip_address`.

However, if you are not connected to a dhcp server, then connect the raspberry pi to your PC using LAN cable. Select a LAN profile (in network manager) that is either `Link-Only` or has `shared with other computer` settings. Check the braodcast address (assuming it to be 10.103.15.255/24) for Ethernt on PC by issuing `ip addr` command. Using `nmap` issue following command 
```
nmap -n -sP 10.103.15.255/24
```
Ideally, two computer should respond --  One is your PC other is raspberry. Use the raspberry ip to login into the machine. Further in case if you do not wish to use DHCP in future too, then you can setup zero conf network on raspberry pi and PC using following commands:
```bash
sudo -E apt install -y avahi-daemon
sudo service avahi-daemon start
sudo systemctl enable avahi-daemon
```
Now if PC and raspberry pi are directly connected with LAN cable. You can login into raspberry with `ssh pi@<raspberry_hostname>.local`. By default `<raspberry_hostname>` is `raspberry`.

## Changing the default account

The default username in raspberry pi 4 is `pi` with password `raspberry`. For deploying the machine for production use, the username and password need to be changed. For that we need to enable root login and login as root using following command

```bash
sudo passwd root		#Set password of the root user and enable it.
logout				#Loguout the current session and relogin as root
```

Inside root login change the username (and  and home directory of the user using following commands

```bash
usermod -l <new_user_name> pi
usermod -m -d /home/<new_user_name> <new_user_name>
groupmod -n <new_group_name> pi
logout
```

Login into changed user and change the password using `passwd` command. Check that user is still a sudo user by running `sudo apt update`. If everything is fine then disable root login using `sudo passwd -l root`.

## Change misc. options in raspi-config

Change the hostname, user autologin, country, timezone, keyboard layout, locale, enable ssh, and reduce graphics memory in raspi-config.
```bash
sudo raspi-config
```

## Installing NTFS and exFAT support

```bash
sudo -E apt install -y ntfs-3g exfat-fuse
```

## Install git and other essential software

Install git 
```
sudo -E apt install -y git
```
Install samba and docker (optional)
```
sudo -E apt install -y samba docker.io docker-compose
```

## Install htpdate

It is optional if your raspberry pi is not able to connect to internet directly (i.e. it is connected to internet through proxy). In that case time may not get updated automatically from ntp servers, so we require htpdate. This program update the system time based on http/https request that most proxy allow.
```
sudo -E apt install -y htpdate
```
Then find the relevant line in `/etc/default/htpdate` and change it to
```
HTP_PROXY="-P <proxy_address>:<proxy_port>"
HTP_OPTIONS="-D -s -t -l"
```
Change `<proxy_address>` and `<proxy_port>` to match your proxy setting.

## Automatically mount external drives

If you wish to set up raspberry pi as network attached storage (NAS), then the external attached storage device need to be mounted automatically on boot. For that make a folder `DATA` in `\media`.
```
sudo mkdir -p \media\DATA
```
Attach the external storage device to raspberry pi and get its `UUID` by issuing the command `sudo lsblk -f`. Add the following line at the end of `\etc\fstab`: (assuming disk has ntfs file system)
```
UUID=uuid_of_disk_drive /media/DATA ntfs defaults,rw,auto,users    0   0
```
Check the settings by issuing `sudo mount -a`. Now the disk will be mounted automatically from next boot.

## Enable firewall (Optional; Enhances security)

Consider adding firewall support to raspberry pi if it is connected to network as being a small device, it can be easily DNDed. For adding firewall support install `nftables` in raspberry
```
sudo -E apt install -y nftables
```

Make a file `/etc/nftables.firewall.rules` with following content:
```
flush ruleset

table inet filter {
	chain input {
		type filter hook input priority 0; policy drop;

		# established/related connections
		ct state established,related accept

		# invalid connections
		ct state invalid drop
		
		# loopback interface
		iif lo accept

		# no ping floods:
		ip protocol icmp icmp type echo-request limit rate over 10/second burst 4 packets  drop
		ip6 nexthdr icmpv6 icmpv6 type echo-request limit rate over 10/second burst 4 packets drop

		ct state established,related accept

		# ICMP & IGMP
		ip6 nexthdr icmpv6 icmpv6 type { destination-unreachable, packet-too-big, time-exceeded, parameter-problem, mld-listener-query, mld-listener-report, mld-listener-reduction, nd-router-solicit, nd-router-advert, nd-neighbor-solicit, nd-neighbor-advert, ind-neighbor-solicit, ind-neighbor-advert, mld2-listener-report, echo-request } accept
		ip protocol icmp icmp type { destination-unreachable, router-solicitation, router-advertisement, time-exceeded, parameter-problem, echo-request } accept
		ip protocol igmp accept

		# HTTP (ports 80 & 443)
		tcp dport { http, https } accept

		# SSH (port 22)
		tcp dport ssh accept

		# avoid brute force on ssh:
		tcp dport ssh ct state new limit rate 10/minute accept

		# Log more than 15 connection attempts
		limit rate 15/minute burst 5 packets counter log prefix "nftables denied: " level debug

		# drop all other
		counter drop
	}

	chain forward {
		type filter hook forward priority 0; policy drop;
	}

	chain output {
		type filter hook output priority 0; policy accept;
	}

}
```
To load the firewall settings at startup, make a file `/etc/network/if-pre-up.d/firewall` and following:
```sh
#!/bin/sh
/usr/sbin/nft -f /etc/nftables.firewall.rules
```
Further, make the file `/etc/network/if-pre-up.d/firewall` executable by issuing `sudo chmod +x /etc/network/if-pre-up.d/firewall`. Reboot the system, firewall is loaded after that.

## Making Raspberry readonly

Raspberry without X11 can be run as readonly system. Here we will make raspberry pi 4 readonly by using UnionFS. Note: Before making raspberry readonly install all required softwares. Tip: Install docker before making raspberry pi readonly as it greatly reduce the effort needed to add software afterwards.

At first we need to disable swap by following commands.
```bash
dphys-swapfile swapoff
dphys-swapfile uninstall
update-rc.d dphys-swapfile disable
```
Install `UnionFS` by using command `sudo apt install unionfs-fuse`. Further create a mounting script `mount_unionfs` in `/usr/local/bin/` having following content:
```sh
#!/bin/sh

[ -z “$1” ] && exit 1 || DIR=$1
ROOT_MOUNT=$(awk '$2=="/" {print substr($4,1,2)}' < /etc/fstab)
if [ $ROOT_MOUNT = "rw" ]
then
  /bin/mount --bind ${DIR}_org ${DIR}
else
  /bin/mount -t tmpfs ramdisk ${DIR}_rw
  /usr/bin/unionfs-fuse -o cow,allow_other,nosuid,dev,nonempty ${DIR}_rw=RW:${DIR}_org=RO ${DIR}
fi
```
Make the script executable by running `sudo chmod +x /usr/local/bin/mount_unionfs`. Add the following line in `/etc/fstab`:
```
mount_unionfs   /var            fuse    defaults          0       0
none            /tmp            tmpfs   defaults          0       0
```
Add `ro` option to the mount options for both `root` and `boot` partition in `\etc\fstab` file (usually it is first two lines). Now, prepare the directories using following commands:
```
mv /var /var_org
mkdir /var /var_rw
```
Some files in `/etc` folder like `resolv.conf` also need to symlinked to writable location. For that run following commands:
```
sudo cp /etc/resolv.conf /etc/resolv.conf.bak
sudo rm /etc/resolv.conf
touch /tmp/resolv.conf
sudo ln -s /tmp/resolv.conf /etc/resolv.conf
```
Change the file `/etc/cron.hourly/fake-hwclock`, so that it mount the system writable before update the saved date and then turn the system readonly.
```
#!/bin/sh
#
# Simple cron script - save the current clock periodically in case of
# a power failure or other crash

if (command -v fake-hwclock >/dev/null 2>&1) ; then
  mount -o remount,rw /
  fake-hwclock save
  mount -o remount,ro /
fi
```
In the end add the command `fastboot noswap ro` to `/boot/cmdline.txt`

## Enabling Watchdog
Enable watchdog module and install the watchdog software:
```
modprobe bcm2708_wdog
apt-get install watchdog
```
Edit the file `/etc/watchdog.conf` and add the following lines at the end of the file
```
watchdog-device  = /dev/watchdog
max-load-15      = 25
watchdog-timeout = 10
```
Edit the file `/lib/systemd/system/watchdog.service` and in section [Install]  add the following
```
[Install]
WantedBy=multi-user.target
```
Enable the watchdog by command `sudo systemctl enable watchdog`


In addition to the watchdog, you should set up reboot after a kernel panic. This is done by editing `/etc/sysctl.conf`. Add this line `kernel.panic = 10` at the end of file.This will cause to wait 10 seconds after a kernel panic, then automatically safely reboot the box.

## Switching from Read-Only mode to Read-Write and vice-versa
Add following to `~/.bashrc`:
```
# set variable identifying the filesystem you work in (used in the prompt below)
set_bash_prompt(){
    fs_mode=$(mount | sed -n -e "s/^\/dev\/.* on \/ .*(\(r[w|o]\).*/\1/p")
    PS1='\[\033[01;32m\]\u@\h${fs_mode:+($fs_mode)}\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
}

alias ro='sudo mount -o remount,ro / ; sudo mount -o remount,ro /boot'
alias rw='sudo mount -o remount,rw / ; sudo mount -o remount,rw /boot'

# setup fancy prompt"
PROMPT_COMMAND=set_bash_prompt
```
Now one can use `rw` to make the system writable and `ro` to make the system readonly. Please note that you should use `rw` for making small changes in `\home` or `\etc\`. If you do want to install new software consult the subsequnet section.

## Installing sofware on readonly raspberry (Advanced: Read it very carefully before doing)
Preferred way is to use docker. Please consult docker documentation for that. However, if you do want to install system softwares then it is tricky. During making raspberry pi readonly we changed the way `\var` and `\tmp` is mounted, we need to reverse that before we can install anything. 

Take out the SD card from raspberry pi and mount it in PC. Edit the file `etc/fstab` from the `root` partition of SD card. Comment out the line intended for `\var` and `\tmp`. Further remove the `ro` option from the mount option of root and boot partition. 
Open `cmdline.txt` in boot partition of SD card and remove `ro` from the end.
Issue following commands in succession to restore `var` in root partition of SD card:
```bash
sudo rm -rf var
sudo mv var_org var
```
Take out the SD card safely from PC and put it into raspberry pi. Turn on the raspberry pi and ssh into it. Ensure the raspberry pi is in `rw` mode (should be visible on command prompt). If yes install the softwares needed. 

After that turn off raspberry pi and take out SD card. Mount SD card on PC and undo all the cahnges done in `etc/fstab` of root partition and `cmdline.txt` in boot partition. Further issue following command in to restore the earlier `var_org` folder in root partition of SD card:
```bash
sudo mv var var_org
sudo mkdir var
```
 
## Taking backups of raspberry pi SD card

I usually take backup of raspberry pi after I have made some change. For taking backup, take out SD card from a turned off raspberry pi and put it in PC. Find the device file for SD card by issuing `lsblk` command in terminal (It usually look like `\dev\sdb`. Note there is no number at the end). Go to the folder where you intend to save the backup and take the backup using following command:
```
sudo dd if=sd_card_device_file of=backup_file_name bs=10M status=progress;sync
```
In case your SD card is corrupted, you can restore the backup using following command:
```
sudo dd if=backup_file_name of=sd_card_device_file bs=10M status=progress;sync
```

## Troubleshooting

For troubleshooting the logs in `\var\log` can be checked. 
FOr troubleshooting any service following commands can be used:
```
sudo systemctl status service_name
sudo journalctl 
sudo journalctl -xe 
```

Best of luck with your raspberry pi
