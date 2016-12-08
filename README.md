Raspberry Pi web browser kiosk mode
===================================

## Install OS

Download and install [Minibian](https://minibianpi.wordpress.com/). Follow these [instructions](https://www.raspberrypi.org/documentation/installation/installing-images/).

## Update

Connect your ethernet cable, start your pi, login (user *root*, password *raspberry*) and update your system.

	apt-get update
	apt-get -y install sudo nano psmisc fdisk parted
	apt-get -y upgrade
	apt-get clean

## Extend root filesystem

Follow these [instructions](https://www.raspberrypi.org/forums/viewtopic.php?f=51&t=45265).

## Wireless

If you plan to use wireless network install wifi drivers (you can skip this step):

	apt-get update
	apt-get install -y firmware-ralink firmware-realtek wireless-tools wireless-regdb wpasupplicant iw crda

Set `wlan0` by adding these lines to `/etc/network/interfaces`:

	allow-hotplug wlan0
	iface wlan0 inet manual
	wpa-roam /etc/wpa_supplicant/wpa_supplicant.conf
	iface default inet dhcp

Create `/etc/wpa_supplicant/wpa_supplicant.conf`:

	country=HR
	network={
		ssid="myssidname"
		psk="mypresharedkey"
	}

We need to disable *wait for network* at boot by creating file (with subdirs) `/etc/systemd/system/networking.service.d/reduce-timeout.conf`:

	[Service]
	TimeoutStartSec=1

## User

Add new user *pi*:

	adduser pi

Add user *pi* to sudoers by executing `visudo` and add these line to end of file:

	pi	ALL=NOPASSWD:	ALL

Change *root* password:

	passwd


## Locale

Append this lines to `/home/pi/.profile`:

	export LC_CTYPE=en_GB.UTF-8
	export LC_ALL=en_GB.UTF-8

Set timezone:

	sudo dpkg-reconfigure tzdata

## SSH

Disable *root* SSH login by setting `PermitRootLogin` in `/etc/ssh/sshd_config` file:

	PermitRootLogin no

Restart SSH service:

	/etc/init.d/ssh restart

From now on you can use SSH to setup your system (user *pi* with previously set password).

## Autologin

Create `/etc/systemd/system/getty@tty1.service.d/autologin.conf` file with content:

	[Service]
	ExecStart=
	ExecStart=-/sbin/agetty --autologin pi --noclear %I 38400 linux

Enable with:

	sudo systemctl enable getty@tty1.service

## Window manager

	sudo apt-get install -y xinit x11-xserver-utils matchbox unclutter

## KWeb browser

	cd /tmp
	wget http://steinerdatenbank.de/software/kweb-1.7.5.tar.gz
	tar -xzf kweb-1.7.5.tar.gz
	cd kweb-1.7.5
	./debinstall

## Set StartX

Allow *anybody* to run X server:

	sudo dpkg-reconfigure x11-common

Append this line to your `/etc/rc.local`:

	su - pi -c 'startx' &

Create `/home/pi/.xinitrc`:

	#!/bin/sh
	while true; do
		echo "Clean up previously running apps"
		killall -TERM kweb 2>/dev/null;
		killall -TERM kweb3 2>/dev/null;
		killall -TERM matchbox-window-manager 2>/dev/null;
		sleep 2;
		killall -9 kweb 2>/dev/null;
		killall -9 kweb3 2>/dev/null;
		killall -9 matchbox-window-manager 2>/dev/null;

		echo "Disable DPMS / Screen blanking"
		xset -dpms
		xset s off

		echo "Hide the cursor"
		unclutter -noevents -grab &

		echo "Start the window manager"
		matchbox-window-manager -use_titlebar no -use_cursor no &

		echo "Start the browser"
		kweb3 -KFJHCUA+-zbhrqfpoklgtjeduwxy http://path.to.your/web.page
	done;

## Finally

Reboot your pi and it should start X, launch KWeb browser and be ready with your chosen web-page in kiosk-mode!
