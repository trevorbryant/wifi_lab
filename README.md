# Wireless Lab
Build a multi-Access Point wireless lab for security auditing.

## Requirements
The hardware used to build this lab is simple and minimal. A Raspberry Pi 4 starter kit, and the wireless adapters below were selected. Any supported wireless adapter may be used.

  * Knowledge and ability of managing operating systems, its software, and wireless adapters
  * [Ubuntu 20.04 LTS](https://ubuntu.com/download/raspberry-pi)
  * [Raspberry Pi 4 (4GB) Starter Set](https://smile.amazon.com/gp/product/B0854QL9L2)
  * [Panda Ultra 150Mbps Wireless N USB Adapter x2](https://smile.amazon.com/gp/product/B00762YNMG)
  * [Panda 300Mbps Wireless N USB Adapter x2](https://smile.amazon.com/gp/product/B00EQT0YK2)
  * [Panda Wireless PAU07 N600 Dual Band Wireless N USB Adapter x2](https://smile.amazon.com/gp/product/B00U2SIS0O) (for 5Ghz)

## Host
This project does **not** use `dnsmasq` or `dhcpcd`. Instead, this project makes use of the `systemd-networkd` package provided, and `hostapd` for the wireless interface station and access point configurations. This removes the complexity of disabling and removing out-of-box configurations in place of third party competing services.

As the project is currently configured, the configurations for `hostapd` will create the virtual interfaces where CTF scenarios 01 through 07 are assigned to `wlan1`, and 08 through 14 can be assigned to `wlan2`. This will create the virtual interfaces for the adapters plugged into USB ports 1 and 2. The on-board wireless has a `udev` rule to force it to be `wlan0` avoiding conflict with the other interfaces.

### DHCPServer
Because this project does not use `dnsmasq`, the DHCP server is managed by `systemd-networkd`'s [DHCPServer](https://wiki.archlinux.org/index.php/Systemd-networkd#[DHCPServer]) configuration. Each access point can manage its own DHCP server, though DHCP is currently configured per interface. The clients that connect will receive an an IPv4 address when `DHCP=ipv4` is set. Any other clients outside of this lab will receive an IPv4 address as anticipated.

### hostapd
The access points configuration files located in `/etc/hostapd/` are configured as follow:

Configurations for `wlan1`:
 * `CTF_01` &ndash; WEP 128-bit, connected client
 * `CTF_02` &ndash; WEP, via a client (planned)
 * `CTF_03` &ndash; WEP, clientless
 * `CTF_04` &ndash; WEP, Shared Key Authentication
 * `CTF_05` &ndash; WPA-PSK, via a roaming client
 * `CTF_06` &ndash; WEP, via a roaming client
 * `CTF_07` &ndash; TBD

Configurations for `wlan2`:
 * NA

### wpa_supplicant
The onboard physical interface is configured to be a client looking for an AP.

 * `wlan0` &ndash; Client looking for `WCTF_15`

## Client
A separate device is to be used as the clients. The client device(s) can be anything that has wifi and cannot to an AP. The assumption here is another raspberry pi. I haven't figured out if I can create virtual clients yet.

### wpa_supplicant
Here, simple `wpa_supplicant` configurations are used as clients connecting to the respective APs.

* `wlan0` &ndash; Connected client, `CTF_04`
* `wlan1` &ndash; Connected client, `CTF_01`
* `wlan2` &ndash; Roaming client, `CTF_05`, `CTF_06`

## Procedures
### Host Setup
These step-by-step instructions are to set up the configurations to create the host environment.

1. After initial login, set timezone, hostname, and update packages.
```bash
$ sudo hostnamectl set-hostname four
$ sudo timedatectl set-timezone America/New_York
$ sudo apt update && sudo apt -y dist-upgrade
```

2. Install the necessary packages, and any others desired, and reboot the system.
```bash
$ sudo apt -y install hostapd rfkill mlocate net-tools iw netfilter-persistent iptables-persistent
$ sudo reboot
```

3. Copy the following to the target locations.

* `wlan*.conf` &rarr; `/etc/hostapd/`
* `wlan*.network` &rarr; `/etc/systemd/network`
* `wpa_supplicant-wlan*.conf` &rarr; `/etc/wpa_supplicant/`
* `71-brcmfmac.rules` &rarr; `/etc/udev/rules.d/`

4. Enable `systemd` services.
```bash
$ sudo systemctl enable hostapd@wlan1.service
$ sudo systemctl enable wpa_supplicant@wlan0.service
```

5. Enable the local firewall. The ports to enable are `ssh`, `53` for UDP, and `67` for DHCP.
```bash
$ sudo ufw enable
$ sudo ufw allow ssh
$ sudo ufw allow 53
$ sudo ufw allow 67
```

And reboot.

In theory, the system will start successfully with all interfaces doing as they are configured to.

### Client Setup
Setting up the client is just as simple.

1. After initial login, set timezone, hostname, and update packages.
```bash
$ sudo hostnamectl set-hostname wasp
$ sudo timedatectl set-timezone America/New_York
$ sudo apt update && sudo apt -y dist-upgrade
$ sudo reboot
```

2. Copy the following to the target locations.

* `wpa_supplicant-wlan*.conf` &rarr; `/etc/wpa_supplicant/`

3. Enable and start the `systemd` services.
```bash
$ sudo systemctl enable wpa_supplicant@wlan0.service
$ sudo systemctl enable wpa_supplicant@wlan1.service
$ sudo systemctl enable wpa_supplicant@wlan2.service
$ sudo systemctl start wpa_supplicant@wlan0.service
$ sudo systemctl start wpa_supplicant@wlan1.service
$ sudo systemctl start wpa_supplicant@wlan2.service
```

4. Enable the local firewall. The ports to enable are `ssh`.
```bash
$ sudo ufw enable
$ sudo ufw allow ssh
```

An additional reboot is not needed.

### MOTD Advertisement (Optional)
Disable [MOTD advertisement](https://bugs.launchpad.net/ubuntu/+source/base-files/+bug/1701068).
```bash
$ sudo sed -i 's/ENABLED=1/ENABLED=0/' /etc/default/motd-news
```

## Troubleshoot
### Network boot delays
If experiencing an unusually longer boot time, review `/var/log/syslog` for any `systemd-networkd-wait-online` warnings or errors causing the waiting for network delays.

Check the status with `$ sudo systemctl status systemd-networkd-wait-online.service` to identify the cause of delays. If a specific network interface is causing the delays, you may modify the `systemd-networkd-wait-online.service`to the example below.

The server can be disabled entirely, though that is recommended if no other option is successful.

```bash
$ ExecStart=/lib/systemd/systemd-networkd-wait-online --timeout 1
```

### Service start failure
As is common with `systemd`, there is a possibility that the `hostapd` services may attempt to start before the virtual interfaces are created. If this is the case, edit the `hostapd@ctf_*.service` and change `Restart=on-failure` to `Restart=always`.
