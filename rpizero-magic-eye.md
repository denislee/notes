# burn image on sd for rpi zero

```
sudo dd if=/home/dns/Downloads/2024-10-22-raspios-bullseye-armhf-lite.img of=/dev/mmcblk0 bs=4M status=progress
```

# enable USB OTG and Networking

Open the file config.txt in the boot partition and add the following line to enable OTG mode:

```
dtoverlay=dwc2
```

Edit cmdline.txt (also in the boot partition). Insert `modules-load=dwc2,g_ether` right after rootwait, leaving the rest of the line untouched. For example:

```
console=serial0,115200 console=tty1 root=PARTUUID=xxxx-xxxx rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait modules-load=dwc2,g_ether
```

# enable SSH for Headless Access

To enable SSH, create an empty file named ssh (no extension) in the boot partition. This allows SSH access once the Pi boots.


# change password 

```
python3 -c 'import crypt; print(crypt.crypt("newpassword", crypt.mksalt(crypt.METHOD_SHA512)))'
```

Replace "newpassword" with your desired password. This will generate a hashed password string you can copy and paste into the shadow file. It will look something like this:

```
pi:$6$someRandomHash$EncryptedPasswordString:20018:0:99999:7:::
```

# ssh config

edit `etc/ssh/sshd_config`

```
PasswordAuthentication yes
PermitRootLogin prohibit-password
```


# set a Static IP Address (Optional but Recommended)

```
sudo vim /etc/dhcpcd.conf
```

Add the following at the end:

```
interface usb0
static ip_address=192.168.2.2/24
static routers=192.168.2.1
static domain_name_servers=8.8.8.8
```

# optional: Internet Sharing

If you want the Raspberry Pi to access the internet through the host, enable IP forwarding and set up NAT:

On the host:

```
sudo iptables -t nat -A POSTROUTING -o <your_host_interface> -j MASQUERADE
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

Replace <your_host_interface> with your host's primary internet interface (e.g., wlan0 or eth0).

# bridge

shell script

```
#!/bin/bash

export DEVICE=$(ip a | sed -n '/enx[0-9a-f]\{12\}/ {s/.*\(enx[0-9a-f]\{12\}\).*/\1/; p; q}')

echo $DEVICE

sudo nmcli dev set $DEVICE managed no
sudo ip addr add 192.168.2.1/24 dev $DEVICE
sudo ip link set $DEVICE down
sudo ip link set $DEVICE up

sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -i $DEVICE -o wlan0 -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o $DEVICE -m state --state RELATED,ESTABLISHED -j ACCEPT

# for rtsp
sudo iptables -t nat -A PREROUTING -p tcp --dport 8555 -j DNAT --to-destination 192.168.2.2:8555
sudo iptables -t nat -A POSTROUTING -p tcp --dport 8555 -j MASQUERADE
sudo iptables -t nat -A PREROUTING -p tcp --dport 8554 -j DNAT --to-destination 192.168.2.2:8554
sudo iptables -t nat -A POSTROUTING -p tcp --dport 8554 -j MASQUERADE

sudo iptables -t nat -A PREROUTING -p udp --dport 1024:65535 -j DNAT --to-destination 192.168.2.2
sudo iptables -t nat -A POSTROUTING -p udp --dport 1024:65535 -j MASQUERADE


# check if is ready
ping 192.168.2.2
```

# enable camera

```
sudo raspi-config
```

Interface Options > Legacy Camera > Enable

Reboot.

# v4l2rtspserver

## install dependencies

You need to install the required dependencies for v4l2rtspserver. First, make sure your Raspberry Pi system is up-to-date:

```
sudo apt update -y && sudo apt upgrade -y && sudo apt dist-upgrade
```

Then, install the necessary packages:

```
sudo apt install build-essential cmake git libv4l-dev vim
```

## Install v4l2rtspserver

To install v4l2rtspserver, you'll need to compile it from source. Follow these steps:

Clone the repository:

```
git clone https://github.com/mpromonet/v4l2rtspserver.git
```

Navigate into the repository:

```
cd v4l2rtspserver
```

Compile the server:

```
cmake . ; \
make ; \
sudo make install
```

# make it a service

## set up autostart

Create a systemd service:

```
sudo vim /etc/systemd/system/v4l2rtspserver.service
```

Add the following content:

```
[Unit]
Description=RTSP Server
After=network.target

[Service]
ExecStart=/usr/local/bin/v4l2rtspserver /dev/video0
Restart=always

[Install]
WantedBy=multi-user.target

Enable and Start the Service:

sudo systemctl enable v4l2rtspserver
sudo systemctl start v4l2rtspserver
```

# test if camera is working

```
vcgencmd get_camera
```
