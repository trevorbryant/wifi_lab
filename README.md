# Wireless Lab
Build a multi-Access Point wireless lab for security auditing.

## Requirements
The hardware used to build this lab is simple and minimal. A Raspberry Pi 4 starter kit, and the wireless adapters below were selected. Any supported wireless adapter may be used.

  * Knowledge and ability of managing operating systems, its software, and wireless adapters
  * [Ubuntu 20.04 LTS](https://ubuntu.com/download/raspberry-pi)
  * [Raspberry Pi 4 (4GB) Starter Set](https://smile.amazon.com/gp/product/B0854QL9L2)
  * [Panda Ultra 150Mbps Wireless N USB Adapter x2](https://smile.amazon.com/gp/product/B00762YNMG) (optional)
  * [Panda 300Mbps Wireless N USB Adapter x2](https://smile.amazon.com/gp/product/B00EQT0YK2)
  * [Panda Wireless PAU07 N600 Dual Band Wireless N USB Adapter x2](https://smile.amazon.com/gp/product/B00U2SIS0O) (for 5Ghz)

## Setup
This project does **not** use `dnsmasq`. Instead, this project makes use of the `systemd-networkd` package provided, and `hostapd` for the wireless interface station and access point configurations. This removes the complexity of disabling and removing out-of-box configurations in place of third party competing services.

As the project is currently configured, the `90-wireless.rules` will create 14 virtual interfaces where 01 through 07 are assigned to `wlan1`, and 08 through 14 are assigned to `wlan2`. This _should_ create the virtual interfaces with the adapters plugged into USB ports 1 and 2 during boot, or adding the adapters post-boot.

The physical interfaces are configured as clients. Any combination of the wireless adapters can be used, however this project configuration required four adapters.

### DHCPServer
Because this project does not use the traditional `dnsmasq`, the DHCP server is configured to be handled by `systemd-networkd`'s [DHCPServer](https://wiki.archlinux.org/index.php/Systemd-networkd#[DHCPServer]). Each access point will manage its own DHCP server. Depending on the virtual interface addresses, may or may not be assigned to connecting clients to mimic real-world scenario.

### hostapd
The access points configuration files located in `/etc/hostapd/` are configured as follow:

Configurations for `wlan1`:
 * `CTF_01` &ndash; WEP 128-bit, connected client
 * `CTF_02` &ndash; WEP, via a client (planned)
 * `CTF_03` &ndash; WEP, clientless (planned)
 * `CTF_04` &ndash; WEP, Shared Key Authentication
 * `CTF_05` &ndash; WPA-PSK, via a roaming client
 * `CTF_06` &ndash; WEP, via a roaming client
 * `CTF_07` &ndash; TBD

Configurations for `wlan2`:
 * TBD

### wpa_supplicant
The physical interfaces are used as clients.

 * `wlan0` &ndash; Client looking for `WCTF_15`
 * `wlan1` &ndash; Connected client, `CTF_04`
 * `wlan3` &ndash; Connected client, `CTF_01`
 * `wlan4` &ndash; Roaming client, `CTF_05`, `CTF_06`

## Procedures
### Initial Configuration
These step-by-step instructions are to set up the configurations to create the wireless client and access point interfaces.

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

3. Copy the following to the target locations. Example: `rsync -v --rsync-path="sudo rsync" origin user@destination`

* `ctf_*.conf` &rarr; `/etc/hostapd/`
* `ctf_*.network` &rarr; `/etc/systemd/network`
* `wpa_supplicant-wlan*.conf` &rarr; `/etc/wpa_supplicant/`
* `90-wireless.rules` &rarr; `/etc/udev/rules.d/90-wireless.rules`

4. Enable `systemd` services, and reboot.
```bash
$ sudo systemctl enable hostapd@ctf_01
$ sudo systemctl enable hostapd@ctf_04
$ sudo systemctl enable hostapd@ctf_05
$ sudo systemctl enable hostapd@ctf_06
$ sudo systemctl enable wpa_supplicant_wlan0.service
$ sudo systemctl enable wpa_supplicant_wlan3.service
$ sudo systemctl enable wpa_supplicant_wlan4.service
$ sudo reboot
```

In theory, the system will start successfully with all interfaces doing as they are configured to.

### Firewall
Enable the local firewall to limit client interaction between network interface. The ports to enable are `ssh`, `53` for UDP, and `67` for DHCP.

```bash
$ sudo ufw enable
$ sudo ufw allow ssh
$ sudo ufw allow 53
$ sudo ufw allow 67
```

### MOTD Advertisement
The `connect` script will configure interfaces `wlan0` and `wlan4` to connect (and reconnect) to target APs. This can be modified or disabled and `wlan4` instead used for an additional network, that is currently configured for 2.4/5ghz.

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
$ ExecStart=/lib/systemd/systemd-networkd-wait-online --ignore wlan1 --ignore wlan2
```
### Service start failure
As is common with `systemd`, there is a possibility that the `hostapd` services may attempt to start before the virtual interfaces are created. If this is the case, edit the `hostapd@ctf_*.service` and change `Restart=on-failure` to `Restart=always`.

## To do:
  * Network bridge
  * iptables masquerade
