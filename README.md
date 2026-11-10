# initramfs-wifi

## Description

### Wi-Fi support in the initramfs

This package provides Wi-Fi connectivity during the early stages of the boot
process. It copies `wpa_supplicant`, wireless kernel modules, and associated
firmware into the initramfs so that the kernel can initialize Wi-Fi hardware
before the root file system is available. It integrates with Dropbear and other
network-capable initramfs tools.

## Installation

Download the Debian package and run the following command:

```bash
sudo dpkg -i initramfs-wifi_1.1_all.deb || sudo apt-get -f install
```

Commands in this guide that change the network configuration or rebuild the
initramfs require root privileges. Run them from a root shell or use `sudo` as
shown where appropriate.

## Wi-Fi Configuration

The `/etc/initramfs-wifi` directory contains three configuration files:

* `initramfs.conf` contains the `WIRELESS` variable, which determines which
  wireless kernel modules are included in the initramfs. The supported values
  are:
  * `all` - Include all wireless kernel modules. This maximizes hardware
    compatibility but produces a larger initramfs.
  * `loaded` - Include the currently loaded wireless kernel modules.
  * `list` - Include the modules listed in
    `/etc/initramfs-wifi/modules.conf`. Only wireless kernel modules are used.
  * `none` - Do not include any wireless kernel modules.
* `modules.conf` lists the wireless kernel modules to include when `WIRELESS`
  is set to `list`. It has no effect in the other modes.
* `wpa_supplicant.conf` contains the Wi-Fi credentials and other connection
  settings:
  * `ssid` - The Wi-Fi network name.
  * `psk` - The Wi-Fi pre-shared key (passphrase).
  * `country` - The optional country code.
  * `ap_scan` - See the `wpa_supplicant` manual page for details.
  * `scan_ssid` - See the `wpa_supplicant` manual page for details.

The default is `WIRELESS=all`, so wireless kernel modules are included
automatically unless you explicitly select another mode. This setting controls
only which modules are copied into the initramfs. To use Wi-Fi during early
boot, enter your credentials in `wpa_supplicant.conf`, enable the premount
script with the `initramfs-wifi=` kernel command-line parameter, and rebuild
the initramfs after changing the configuration. Depending on your hardware,
you may also need to install the `firmware-misc-nonfree` package.

Configuration changes do not take effect until the initramfs is rebuilt. The
required command is described below, but you may want to test the configuration
first.

## Testing the Wi-Fi Configuration

You can test `wpa_supplicant.conf` before rebuilding the initramfs. The
following test is intended to be run locally on a desktop system such as Ubuntu
with GNOME, where NetworkManager manages the Wi-Fi connection. Do not run it
through SSH or another remote connection that depends on the interface being
tested, because stopping NetworkManager may disconnect you from the system.

On systems that do not use NetworkManager, stop the service that currently
manages the wireless interface instead and adapt the commands accordingly.
On a NetworkManager-based desktop, stop NetworkManager and the system
`wpa_supplicant` service, then unblock wireless devices to prevent conflicts:

```bash
sudo systemctl stop wpa_supplicant.service NetworkManager.service
sudo rfkill unblock wlan
```

Now test the connection:

```bash
sudo wpa_supplicant -c /etc/initramfs-wifi/wpa_supplicant.conf -i wlan0
```

You can also pass the configuration through standard input. The following
example shows what a typical `wpa_supplicant` configuration looks like. In most
cases, specifying the `ssid` and `psk` is sufficient:

```bash
cat << CONF | sudo wpa_supplicant -c /dev/stdin -i wlan0
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

If the connection succeeds, the output should contain
`CTRL-EVENT-CONNECTED`:

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

Press `CTRL-C` to terminate `wpa_supplicant`, then restart the services that
were stopped for the test:

```bash
sudo systemctl start wpa_supplicant.service NetworkManager.service
```

## IP Configuration

The IP address and network interface are configured through
[kernel command-line parameters](https://www.kernel.org/doc/Documentation/filesystems/nfs/nfsroot.txt).

Explicitly specifying the network interface is strongly recommended. Without
an interface name, the first interface to receive a reply is used, and there
is no guarantee that it will be the wireless interface.

A DHCP configuration looks like this:

```text
ip=:::::wlan0:dhcp
```

A static configuration could look like this:

```text
ip=192.168.1.100::192.168.1.1:255.255.255.0:your-hostname:wlan0:static:8.8.8.8:8.8.4.4
```

## Testing the IP Configuration

The initramfs uses
[`ipconfig`](https://git.kernel.org/pub/scm/libs/klibc/klibc.git/plain/usr/kinit/ipconfig/README.ipconfig)
to configure the network. To test the IP configuration, first connect to the
Wi-Fi network as described above. Then run the following command in another
terminal:

```bash
sudo /usr/lib/klibc/bin/ipconfig :::::wlan0:dhcp
```

The output should look similar to this:

```text
IP-Config: wlan0 hardware address aa:bb:cc:dd:ee:ff mtu 1500 DHCP
IP-Config: wlan0 guessed broadcast address 192.168.1.255
IP-Config: wlan0 complete (dhcp from 192.168.1.1):
 address: 192.168.1.100   broadcast: 192.168.1.255   netmask: 255.255.255.0
 gateway: 192.168.1.1     dns0     : 192.168.1.1     dns1   : 0.0.0.0
 rootserver: 192.168.1.1  rootpath:
 filename  :
```

During boot, this information appears as a boot message if the logging settings
allow it. With DHCP, the message shows the assigned IP address, which you need
to connect to the computer through SSH. Configure the system to display boot
messages if you need this information during early boot.

If boot messages are unavailable, you can try `arp -a` or `ip neigh show` to
find the address assigned through DHCP. As a last resort, scan the local network
range with `netdiscover`.

After running `ipconfig`, use the following commands to verify the network
configuration and connectivity.

First, check the IP address:

```bash
ip -4 -br addr show dev wlan0
```

The output should look similar to this:

```text
wlan0            UP             192.168.1.100/24
```

Then check the routes:

```bash
ip -4 route show dev wlan0
```

The output should look similar to this:

```text
default via 192.168.1.1
192.168.1.0/24 proto kernel scope link src 192.168.1.100
```

Finally, ping a host on the Internet:

```bash
ping -n -c 1 -w 5 -I wlan0 1.1.1.1
```

The output should look similar to this:

```text
PING 1.1.1.1 (1.1.1.1) from 192.168.1.100 wlan0: 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=58 time=27.3 ms
```

## Wi-Fi Interface Configuration

The initramfs-wifi premount script is disabled by default. It starts only when
the `initramfs-wifi=` kernel command-line parameter contains an enabling value
or an interface name.

Use one of the following values to enable initramfs-wifi with automatic
interface detection:

```text
initramfs-wifi=yes
initramfs-wifi=enabled
initramfs-wifi=1
```

Unless `wpa_supplicant.interface=` is also specified, the script uses the first
wireless interface it finds.

To select an interface directly, assign its name to `initramfs-wifi=`:

```text
initramfs-wifi=wlan0
```

Any value other than the documented enabling and disabling values is interpreted
as an interface name. When `initramfs-wifi=` specifies an interface,
`wpa_supplicant.interface=` is ignored.

Use one of the following values to disable initramfs-wifi explicitly:

```text
initramfs-wifi=
initramfs-wifi=no
initramfs-wifi=disabled
initramfs-wifi=0
```

If no wireless interface is immediately available, the script waits up to five
seconds before giving up.

## Changing the Kernel Command Line

The method used to change the kernel command line depends on the hardware and
boot loader. On a Raspberry Pi, edit `cmdline.txt` in the `/boot` directory. If
you use GRUB, add the parameters to `GRUB_CMDLINE_LINUX` in `/etc/default/grub`.
For example, a DHCP configuration can look like this:

```text
ip=:::::wlan0:dhcp initramfs-wifi=wlan0
```

To let initramfs-wifi detect the wireless interface automatically, use:

```text
ip=:::::wlan0:dhcp initramfs-wifi=yes
```

To specify the interface separately, use:

```text
ip=:::::wlan0:dhcp initramfs-wifi=yes wpa_supplicant.interface=wlan0
```

After changing `/etc/default/grub`, update the GRUB configuration:

```bash
sudo update-grub
```

## Additional Kernel Command-Line Parameters

The following parameters override settings from `wpa_supplicant.conf` or
control debugging and timeouts:

* `wpa_supplicant.ap_scan` - Overrides the `ap_scan` value in the
  `wpa_supplicant.conf` file in the initramfs.
* `wpa_supplicant.country` - Overrides the `country` value in the
  `wpa_supplicant.conf` file in the initramfs.
* `wpa_supplicant.scan_ssid` - Overrides the `scan_ssid` value in the
  `wpa_supplicant.conf` file in the initramfs.
* `wpa_supplicant.ssid` - Overrides the `ssid` value in the
  `wpa_supplicant.conf` file in the initramfs.
* `wpa_supplicant.psk` - Overrides the `psk` value in the
  `wpa_supplicant.conf` file in the initramfs.
* `wpa_supplicant.debug` - Enables `wpa_supplicant` debugging when set to `yes`,
  `1`, `2`, or `3`.
* `wpa_supplicant.dumpconf` - Dumps the `wpa_supplicant` configuration when set
  to `yes` or `1`.
* `wpa_supplicant.dumplogs` - Dumps the `wpa_supplicant` logs when set to `yes`
  or `1`.
* `wpa_supplicant.interface` - Selects the wireless interface when
  `initramfs-wifi=` enables the script without specifying an interface name.
  This parameter does not enable initramfs-wifi by itself.
* `wpa_supplicant.timeout` - Sets the number of seconds to wait for a connection.
  The default is `30` seconds.

## Updating the Initramfs

After configuring and testing the connection, enable initramfs updates in
`/etc/initramfs-tools/update-initramfs.conf`:

```text
update_initramfs=yes
```

Then rebuild the initramfs:

```bash
sudo update-initramfs -v -c -k all
```

The `-v` option enables verbose output so that you can see which files and
modules are added.

## Possible Missing Firmware

Adding wireless drivers to the initramfs may produce warnings such as:

```text
W: Possible missing firmware /lib/firmware/.../... for module ...
```

These warnings are usually caused by a limitation in the initramfs scripts.
The scripts obtain the required firmware paths by running `modinfo` for each
module. Some modules report filename patterns containing wildcards such as `*`
and `?` instead of literal filenames. For example, the `brcmfmac` module used by
the Wi-Fi adapter in a Raspberry Pi 3B reports entries similar to these:

```text
...
firmware:       brcm/brcmfmac*-sdio.*.bin
firmware:       brcm/brcmfmac*-sdio.*.txt
...
```

The standard initramfs scripts may interpret these entries as literal filenames
and report them as missing. Many other scripts that add Wi-Fi support to an
initramfs do not account for these patterns either. The initramfs-wifi hook
expands the patterns and copies the matching firmware files whenever possible.
This is important for modules such as `brcmfmac`, which may require matching
NVRAM configuration files. The original warning may remain even when the hook
has copied all matching files correctly.

Warnings that appear only after wireless modules are added can therefore
usually be ignored after confirming that the required firmware was copied.
Warnings for unrelated modules or hardware should still be investigated.

## Usage

Restart the system. You should then be able to connect to Dropbear through
Wi-Fi during early boot.

If neither Dropbear nor another initramfs application uses the network, this
package provides no practical benefit.

## Contributing

Contributions are welcome. Feel free to fork this repository and submit a pull
request.

## License

This project is licensed under the GNU General Public License version 3 or
later. See the `COPYING` file for details.

## Author

The initramfs-wifi Debian package was created by m4dm4x1337.

## Donations 🥺

❤️ Please donate ❤️

![QR code for donations](https://raw.githubusercontent.com/m4dm4x1337/tor-router-gnome/master/tor-router-gnome/usr/share/pixmaps/tor-router-gnome-donation.png)

This project is open source and supported by donations. If you find it useful,
please consider contributing to its continued development.

[bitcoin:bc1q9ha0l0tt7dghcpgext8jppejandefeshcukpxx](bitcoin:bc1q9ha0l0tt7dghcpgext8jppejandefeshcukpxx)
