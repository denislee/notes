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

# v4l2rtspserver

1. Install Dependencies

You need to install the required dependencies for v4l2rtspserver. First, make sure your Raspberry Pi system is up-to-date:

```
sudo apt update -y && sudo apt upgrade -y && sudo apt dist-upgrade
```

Then, install the necessary packages:

sudo apt install build-essential cmake git libv4l-dev

2. Install v4l2rtspserver

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
cmake .
make
sudo make install
```

# make it a service

## Set Up Autostart

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
