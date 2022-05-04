Add dtoverlay=dwc2 to the /boot/firmware/config.txt\
Add modules-load=dwc2 to the end of /boot/firmware/cmdline.txt\
Add libcomposite to /etc/modules\
Add denyinterfaces usb0 to /etc/dhcpcd.conf\
Install dnsmasq with sudo apt install dnsmasq\
Create /etc/dnsmasq.d/usb with following content
```
interface=usb0
dhcp-range=10.55.0.2,10.55.0.6,255.255.255.248,1h
dhcp-option=3
leasefile-ro
port=0
```
Edit available /etc/netplan/* config file
```
network:
  version: 2
  renderer: networkd
  wifis:
    wlan0:
      dhcp4: yes
      dhcp6: no
      access-points:
        "my-ssid1":
          password: "secret-preshared-key1"
    wlan0:
      dhcp4: yes
      dhcp6: no
      access-points:
        "my-ssid2":
          password: "secret-preshared-key2"
  ethernets:
    eth0:
      dhcp4: true
    usb0:
      dhcp4: false
      optional: true
      addresses: [10.55.0.1/29]
```
Create /root/usb.sh
```shell
#!/bin/bash
cd /sys/kernel/config/usb_gadget/
mkdir -p pi4
cd pi4
echo 0x1d6b > idVendor # Linux Foundation
echo 0x0104 > idProduct # Multifunction Composite Gadget
echo 0x0100 > bcdDevice # v1.0.0
echo 0x0200 > bcdUSB # USB2
echo 0xEF > bDeviceClass
echo 0x02 > bDeviceSubClass
echo 0x01 > bDeviceProtocol
mkdir -p strings/0x409
echo "fedcba9876543211" > strings/0x409/serialnumber
echo "Jan rpi" > strings/0x409/manufacturer
echo "PI4 USB Device" > strings/0x409/product
mkdir -p configs/c.1/strings/0x409
echo "Config 1: ECM network" > configs/c.1/strings/0x409/configuration
echo 250 > configs/c.1/MaxPower
mkdir -p functions/ecm.usb0
HOST="00:dc:c8:f7:75:14" # "HostPC"
SELF="00:dd:dc:eb:6d:a1" # "BadUSB"
echo $HOST > functions/ecm.usb0/host_addr
echo $SELF > functions/ecm.usb0/dev_addr
ln -s functions/ecm.usb0 configs/c.1/
udevadm settle -t 5 || :
ls /sys/class/udc > UDC
systemctl restart sshd
```
Make /root/usb.sh executable with chmod +x /root/usb.sh
Create rc-local.service
```
sudo vim /etc/systemd/system/rc-local.service
```
Copy in following text
```
[Unit]
Description=/etc/rc.local Compatibility
ConditionPathExists=/etc/rc.local

[Service]
Type=forking
ExecStart=/etc/rc.local start
TimeoutSec=0
StandardOutput=tty
RemainAfterExit=yes
SysVStartPriority=99

[Install]
WantedBy=multi-user.target
```
Create /etc/rc.local file with following content
```
#!/bin/bash
/root/usb.sh
exit 0
```
Make it executable with 
```
sudo chmod +x /etc/rc.local
```
Enable rc-local
```shell
sudo systemctl enable --now rc-local
```
