# Initramfs-WiFi

## Description

### WiFi support for the initramfs

This package provides WiFi support for the initramfs in the early stages of the boot process. It includes necessary dependencies such as wpa_supplicant, wireless kernel modules, and associated firmware files into the initrd so that the kernel can initialize WiFi hardware in the early phase of the boot process. The WiFi support integrates seamlessly with Dropbear and other network-capable initramfs tools.

## Installation

Before the package is installed for the first time, it is recommended to temporarily deactivate the update of the initramf as the WiFi configuration must be carried out first. This saves a few minutes of time that would otherwise be needed to update the initramf. Set `update_initramfs=no` in `/etc/initramfs-tools/update-initramfs.conf` to do that.

Download the Debian package and run this commands:

```bash
sudo dpkg -i initramfs-wifi_1.0_all.deb || sudo apt-get -f install
```

## WiFi Configuration

There are three configuration files in `/etc/initramfs-wifi`:

* `initramfs.conf` - This file contains the variable `WIRELESS`, which is used to determine which wireless modules should be integrated into the initramfs. Possible values are:
  * `all` - Add all wireless kernel modules (Maximizes hardware compatibility, but also causes the initramfs to become bigger)
  * `loaded` - Add currently loaded wireless kernel modules (`loaded` is the default value)
  * `list` - Add modules from the file `/etc/initramfs-wifi/modules.conf` (Only wireless kernel modules are taken into account)
  * `none` - Do not include any wireless kernel modules
* `modules.conf` - The wireless kernel modules are read from this file if the `WIRELESS` variable in `initramfs.conf` has the value `list`, otherwise this file has no effect.
* `wpa_supplicant.conf` - This file contains the WiFi name and passphrase as well as some other configuration options:
  * `ssid` - The name of the WiFi
  * `psk` - The pre shared key (passphrase) of the WiFi
  * `country` - The country code (optional)
  * `ap_scan` - See the wpa_supplicant man page for details.
  * `scan_ssid` - See the wpa_supplicant man page for details.

All you have to do is to enter your WiFi credentials in `wpa_supplicant.conf`. If you want to maximize hardware compatibility you can also set `WIRELESS=all` in `initramfs.conf` and install the package `firmware-misc-nonfree`.

Changes in the configuration files will not take effect until the initramfs is updated, which means that you need to run `update-initramfs`, but continue reading before you do it.

## Testing the WiFi Configuration

If you want, you can first verify whether your `wpa_supplicant.conf` file really works. To do this you must first deactivate the `NetworkManager.service`, the `wpa_supplicant.service` and unblock the WiFi to prevent conflicts:

```bash
systemctl stop wpa_supplicant.service NetworkManager.service && rfkill unblock wlan
```

Now you can test the connection:

```bash
wpa_supplicant -c /etc/initramfs-wifi/wpa_supplicant.conf -i wlan0
```

It is also possible to pipe the configuration via STDIN. For the sake of completeness, here is an example in which you can also see how a typical wpa_supplicant configuration file looks like. It is usually enough to specify the `ssid` and the `psk` (pre-shared key):

```bash
cat << CONF | wpa_supplicant -c /dev/stdin -i wlan0
ctrl_interface=/run/wpa_supplicant
#ap_scan=0
#country=

network={
  #scan_ssid=1
  ssid="easylink"
  psk="12345678"
}
CONF
```

If everything is OK you should see `CTRL-EVENT-CONNECTED` in the output:

```text
Successfully initialized wpa_supplicant
wlan0: SME: Trying to authenticate with ff:ee:dd:cc:bb:aa (SSID='easylink' freq=2462 MHz)
wlan0: Trying to associate with ff:ee:dd:cc:bb:aa (SSID='easylink' freq=2462 MHz)
wlan0: Associated with ff:ee:dd:cc:bb:aa
wlan0: CTRL-EVENT-SUBNET-STATUS-UPDATE status=0
wlan0: WPA: Key negotiation completed with ff:ee:dd:cc:bb:aa [PTK=CCMP GTK=CCMP]
wlan0: CTRL-EVENT-CONNECTED - Connection to ff:ee:dd:cc:bb:aa completed [id=0 id_str=]
wlan0: CTRL-EVENT-REGDOM-CHANGE init=COUNTRY_IE type=COUNTRY alpha2=DE
```
Press `CTRL-C` to terminate wpa_supplicant and start the previously stopped services:

```bash
systemctl start wpa_supplicant.service NetworkManager.service
```

## IP Configuration

The configuration of the IP address and the selection of the network interface is done via [kernel cmdline parameters](https://www.kernel.org/doc/Documentation/filesystems/nfs/nfsroot.txt).

It is highly recommended to explicitly specify the name of the network interface to be used, because otherwise *the device that received the first reply* will be used, and there is no guarantee that this will be the wireless interface.

A DHCP configuration looks like this:

```text
ip=:::::wlan0:dhcp
```

And a static configuration could look like this:

```text
ip=192.168.1.100::192.168.1.1:255.255.255.0:your-hostname:wlan0:static:8.8.8.8:8.8.4.4
```

## Testing the IP Configuration

If you want, you can test and verify that the ip configuration works. Under the hood a tool with the name [`ipconfig`](https://git.kernel.org/pub/scm/libs/klibc/klibc.git/plain/usr/kinit/ipconfig/README.ipconfig) is used by the initramfs. To carry out the test, the WiFi hotspot must be connected first (see above). Once connected, run the following command in another terminal window:

```bash
/usr/lib/klibc/bin/ipconfig :::::wlan0:dhcp
```

The output should look something like this:

```text
IP-Config: wlan0 hardware address aa:bb:cc:dd:ee:ff mtu 1500 DHCP
IP-Config: wlan0 guessed broadcast address 192.168.1.255
IP-Config: wlan0 complete (dhcp from 192.168.1.1):
 address: 192.168.1.100   broadcast: 192.168.1.255   netmask: 255.255.255.0
 gateway: 192.168.1.1     dns0     : 192.168.1.1     dns1   : 0.0.0.0
 rootserver: 192.168.1.1  rootpath: 
 filename  : 
```

This text is displayed on the screen as a boot message when booting, provided the log output settings are configured accordingly. I won't go into that any further here. It's important to know is that in case of DHCP this text tells you the ip to be used if you want to connect to that computer via SSH. It is therefore advisable to ensure that boot messages are output.

If boot messages are not available, but you still want to find out the IP assigned via DHCP, you can try the commands `arp -a` and `ip neigh show`, or in extreme cases, simply scan your entire network range with `netdiscover`.

When you run the ip command you should see that the configuration has been applied. Here some commands to verify that the wireless connection works.

First check the ip address:

```bash
ip -4 -br addr show dev wlan0
```

The output should look something like this:

```text
wlan0            UP             192.168.1.100/24 
```

Then check the routes:

```bash
ip -4 route show dev wlan0
```

The output should look something like this:

```text
default via 192.168.1.1 
192.168.1.0/24 proto kernel scope link src 192.168.1.100 
```

Then ping a host in the internet:

```bash
ping -n -c 1 -w 5 -I wlan0 1.1.1.1
```

The output should look something like this:

```text
PING 1.1.1.1 (1.1.1.1) from 192.168.1.100 wlan0: 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=58 time=27.3 ms
```

## WiFi Interface Configuration

By default, the initramfs-wifi premount script uses the first wireless interface it finds.

However, to explicitly set the name of the interface, you can specify the following kernel cmdline parameter:

```text
initramfs-wifi=wlan0
```

This kernel parameter can also be used to disable initramfs-wifi functionality:

```text
initramfs-wifi=no
```

## Changing the kernel cmdline

How to change the kernel cmdline depends on your hardware and/or software. If you are using a Raspberry Pi you need to change the file `cmdline.txt` in your `/boot` directory. If you are using grub as bootloader you need to change the kernel parameters in the variable `GRUB_CMDLINE_LINUX` in the file `/etc/default/grub`. The part that you need to add in case of DHCP is:

```text
ip=:::::wlan0:dhcp initramfs-wifi=wlan0
```

If you are using grub, then run `update-grub` afterwards.

## Additional kernel cmdline parameters

There are a few other additional kernel parameters that can be used for debugging:

* `wpa_supplicant.ap_scan` - Overrides the `ap_scan` value in the `wpa_supplicant.conf` file within the initramfs.
* `wpa_supplicant.country` - Overrides the `country` value in the `wpa_supplicant.conf` file within the initramfs.
* `wpa_supplicant.scan_ssid` - Overrides the `scan_ssid` value in the `wpa_supplicant.conf` file within the initramfs.
* `wpa_supplicant.ssid` - Overrides the `ssid` value in the `wpa_supplicant.conf` file within the initramfs.
* `wpa_supplicant.psk` - Overrides the `psk` value in the `wpa_supplicant.conf` file within the initramfs.
* `wpa_supplicant.debug` - Enable debugging in wpa_supplicant if set to `yes` or `1` or `2` or `3`
* `wpa_supplicant.dumpconf` - Dump the wpa_supplicant config if set to `yes` or `1`
* `wpa_supplicant.dumplogs` - Dump the wpa_supplicant logs if set to `yes` or `1`
* `wpa_supplicant.interface` - Determine the network interface to be used in absence of the `initramfs-wifi` kernel parameter
* `wpa_supplicant.timeout` - Determine how many seconds to wait before giving up connecting to the Wi-Fi hotspot. The default value is `30` seconds.

## Update the initramfs

Now that everything has been thoroughly explained, configured and tested, we can update the initramfs. Set `update_initramfs=yes` in `/etc/initramfs-tools/update-initramfs.conf` and then run this command:

```bash
update-initramfs -v -c -k all
```

The `-v` argument causes a verbose output so that you can see what happens under the hood.

## Possible missing firmware

If you see the warning `Possible missing firmware` during the update, you can ignore it:

```text
W: Possible missing firmware /lib/firmware/.../... for module ...
```

The reason for this is that the initramfs update script determines the firmware files to be copied by running `modinfo modulename` which returns a list of filenames and in some cases filename patterns that contain wildcards like `*` and `?`. For example the WiFi of a Raspberry Pi 3b depends on the kernel module `brcmfmac`. The command `modinfo brcmfmac` outputs these lines:

```text
...
firmware:       brcm/brcmfmac*-sdio.*.bin
firmware:       brcm/brcmfmac*-sdio.*.txt
...
```

The problem is that the script behind the `update-initramfs` command interprets these entries as filenames and not as filename patterns and then "thinks" that some firmware files are missing. All other scripts I have seen on the subject of WiFi in initramfs do not take this into account either. but initramfs-wifi does, it can copy these firmware files, and only then it will work in all cases. The `brcmfmac` module of the Raspberry Pi 3b for example would not work without the NVRAM config files that matches the filename pattern above. Unfortunately, the warning message remains even if everything was copied correctly.

## Usage

Restart and you should be able to connect to dropbear via WiFi.

If you don't have Dropbear installed and there is no other application that can do anything with a Wifi in the initramfs, then of course it makes no sense to install this package.

## Contributing

Contributions are welcome! Feel free to fork this repository and submit pull requests.

## License

This project is licensed under the GPLv3 License. See the COPYING file for details.

## Author

The Initramfs-WiFi Debian package was created by m4dm4x1337.

## Donations 🥺

 ❤️ Please donate ❤️

![QR code for donations](https://raw.githubusercontent.com/m4dm4x1337/tor-router-gnome/master/tor-router-gnome/usr/share/pixmaps/tor-router-gnome-donation.png)

This project is open source, and the only income comes from the donators. If you like the project, please donate, thank you!

[bitcoin:bc1q9ha0l0tt7dghcpgext8jppejandefeshcukpxx](bitcoin:bc1q9ha0l0tt7dghcpgext8jppejandefeshcukpxx)
