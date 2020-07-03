# Wireless Lab
Build a multi-Access Point wireless lab for wireless auditing or other testing.

## Setup
This project does not use `dnsmasq`. Instead, it makes use of the `systemd-networkd` package provided natively, and `hostapd` for the wireless interface configurations. This removes the complexity of disabling and removing out-of-box configurations in place of competing services.

As the project is currently configured, `wlan0` remains as a client while wlan's one through three are configured as managed access points for the 2.4 frequency.

The `wlan4` is currently configured as a client, but can become a managed interface with 5ghz when modifying the `hostapd.service` file to include `wlan4.conf` at service start.

In short, the summary of configuration is to recreate world-world scenarios.
  * `wlan0` connects to `wlan3` (CTF_03) as a client
  * `wlan4` connects to `wlan2`(CTF_02) as a client
  * `wlan1` is a WEP (CTF_01) without clients access point
  * `wlan2` is a WEP-SKA (CTF_02) with clients access point
  * `wlan3` is a WPA-PSK (CTF_03) with client access point

## Requirements
The hardware used to build this lab is simple and minimal. A Raspberry Pi 4 starter kit, and the wireless adapters below were selected. Any supported wireless adapter may be used.

  * Knowledge and ability of managing operating systems, its software, and wireless adapters
  * [Ubuntu 20.04 LTS](https://ubuntu.com/download/raspberry-pi)
  * [Raspberry Pi 4 (4GB) Starter Set](https://smile.amazon.com/gp/product/B0854QL9L2)
  * [Panda Ultra 150Mbps Wireless N USB Adapter x2](https://smile.amazon.com/gp/product/B00762YNMG)
  * [Panda 300Mbps Wireless N USB Adapter x2](https://smile.amazon.com/gp/product/B00EQT0YK2)
  * [Panda Wireless PAU07 N600 Dual Band Wireless N USB Adapter x2](https://smile.amazon.com/gp/product/B00U2SIS0O)

## Instructions
  1. After initial login, set timezone, hostname, and update packages.
  ```bash
  $ sudo hostnamectl set-hostname four
  $ sudo timedatectl set-timezone America/New_York
  $ sudo apt update && sudo apt -y dist-upgrade
  ```

  2. Install the necessary packages, and any others desired, and reboot the system.
  ```bash
  $ sudo apt -y install hostapd rfkill mlocate net-tools netfilter-persistent iptables-persistent
  $ sudo reboot
  ```

  3. Copy the following to the target locations. Example: `rsync -v --rsync-path="sudo rsync" origin user@destination`

    * `dhcpcd.conf` to `/etc/dhcpcd.conf`
    * `wlan*.conf` to `/etc/hostapd/`
    * `wlan*.network` to `/etc/systemd/network`
    * `wpa_wlan*.conf` to `/etc/wpa_supplicant/`
    * `*.service` to `/etc/systemd/system/`
    * `connect` to `/root/connect`

  4. Enable services to run at boot, and reboot.
  ```bash
  $ sudo systemctl enable connect
  $ sudo systemctl enable hostapd
  $ sudo reboot
  ```

## Extra
The `connect` script will configure interfaces `wlan0` and `wlan4` to connect (and reconnect) to target APs. This can be modified or disabled and `wlan4` instead used for an additional network, that is currently configured for 2.4/5ghz.

Disable [MOTD advertisement](https://bugs.launchpad.net/ubuntu/+source/base-files/+bug/1701068).
```bash
$ sudo sed -i 's/ENABLED=1/ENABLED=0/' /etc/default/motd-news
```

## Troubleshoot
If experiencing an unusually longer boot time and interfaces `wlan0` and `wlan4` are not connected as clients, then disable `connect` service from systemd and instead use the (now deprecated) `rc.local` service.

## To do:
  * Set up network bridge
  * Set up iptables masquerade
