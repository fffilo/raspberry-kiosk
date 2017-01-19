Raspberry Pi web browser kiosk mode
===================================

## Install OS

Download and install [Minibian](https://minibianpi.wordpress.com/). Follow these [instructions](https://www.raspberrypi.org/documentation/installation/installing-images/).

## Update

Connect your ethernet cable, start your pi, login (user *root*, password *raspberry*) and update your system.

	apt-get update
	apt-get -y install --no-install-recommends sudo psmisc bash-completion nano
	apt-get -y upgrade
	apt-get -y dist-upgrade
	apt-get clean

## Extend root filesystem

Follow these [instructions](https://www.raspberrypi.org/forums/viewtopic.php?f=51&t=45265).

## Wireless

If you plan to use wireless network install wifi drivers (you can skip this step):

	apt-get -y install --no-install-recommends firmware-ralink firmware-realtek wireless-tools wireless-regdb wpasupplicant iw crda

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

Add user *pi* to *sudoers* by executing `visudo` and add these line to end of file:

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

	sudo apt-get -y install --no-install-recommends xinit x11-xserver-utils xserver-xorg matchbox unclutter xautomation

## Web browsers

##### Chromium

	sudo apt-get -y install --no-install-recommends chromium-browser

##### IceWeasel

	sudo apt-get -y install --no-install-recommends iceweasel

##### Epiphany Web Browser

	sudo apt-get -y install --no-install-recommends epiphany-browser

##### Midori

	sudo apt-get -y install --no-install-recommends midori

##### Kweb Suite (Minimal Kiosk Browser)

	cd /tmp
	wget http://steinerdatenbank.de/software/kweb-1.7.5.tar.gz
	tar -xzf kweb-1.7.5.tar.gz
	cd kweb-1.7.5
	./debinstall

## CEC

If you wish to control your pi with remote control install CEC client:

	sudo apt-get -y install --no-install-recommends cec-utils
	cd /tmp
	git clone https://github.com/fffilo/cec-bind.git
	chmod +x cec-bind/src/cec-bind.sh
	sudo mv cec-bind/src/cec-bind.sh /usr/local/bin/cec-bind
	mv cec-bind/src/config.map ~/.cec-bind

Adjust your `/home/pi/.cec-bind` keymap if necessary (for more info read [this](https://github.com/fffilo/cec-bind)).

## Set StartX

Add user *pi* to *video* group:

	sudo usermod -aG video pi

Allow *anybody* to run X server:

	sudo dpkg-reconfigure x11-common

Autostart X by adding this line to your `/home/pi/.profile`:

	[[ -z $DISPLAY && $XDG_VTNR -eq 1 ]] && exec startx

Create `/home/pi/.xinitrc`:

	#!/bin/sh

	# defaults
	export KIOSK_OSD_NAME="WEB KIOSK"
	export KIOSK_URL="https://www.google.com"

	while true; do
		if [ -f ${HOME}/.kiosk ]; then
			echo "Load config"
			eval "`cat ${HOME}/.kiosk`"
		fi

		echo "Gracefully clean up previously running apps"
		killall -TERM chromium-browser 2>/dev/null;
		killall -TERM midori 2>/dev/null;
		killall -TERM iceweasel 2>/dev/null;
		killall -TERM epiphany-browser 2>/dev/null;
		killall -TERM kweb3 2>/dev/null;
		killall -TERM kweb 2>/dev/null;
		killall -TERM cec-client 2>/dev/null;
		killall -TERM matchbox-window-manager 2>/dev/null;
		sleep 2;

		echo "Harshly clean up previously running apps"
		killall -9 chromium-browser 2>/dev/null;
		killall -9 midori 2>/dev/null;
		killall -9 iceweasel 2>/dev/null;
		killall -9 epiphany-browser 2>/dev/null;
		killall -9 kweb3 2>/dev/null;
		killall -9 kweb 2>/dev/null;
		killall -9 cec-client 2>/dev/null;
		killall -9 matchbox-window-manager 2>/dev/null;

		echo "Clean out existing profile information"
		rm -rf ${HOME}/.cache;
		rm -rf ${HOME}/.config;
		rm -rf ${HOME}/.pki;

		echo "Disable DPMS / Screen blanking"
		xset -dpms
		xset s off

		echo "Hide the cursor"
		unclutter -noevents -grab &

		echo "Start CEC client"
		cec-bind --osd-name="${KIOSK_OSD_NAME}" &

		echo "Start the window manager"
		matchbox-window-manager -use_titlebar no -use_cursor no &

		#echo "Start Chromium Browser - chromium-browser"
		#sed -i 's/"exited_cleanly": false/"exited_cleanly": true/' ${HOME}/.config/chromium Default/Preferences
		#chromium-browser --noerrdialogs --kiosk "${KIOSK_URL}" --incognito --disable-translate

		#echo "Start Midori Browser - midori"
		#midori -e Fullscreen -a "${KIOSK_URL}"

		#echo "Start IceWeasel Browser - iceweasel"
		#sleep 15s && xte "key F11" -x:0 &
		#iceweasel "${KIOSK_URL}"

		#echo "Start Epiphany Browser - epiphany-browser"
		#sleep 15s && xte "key F11" -x:0 &
		#epiphany-browser -a -i --profile ~/.config "${KIOSK_URL}"

		echo "Start Kweb Suite (Minimal Kiosk Browser) - kweb3"
		kweb3 -KFJHCUA+-zbhrqfpoklgtjeduwxy "${KIOSK_URL}"

		#echo "Start Kweb Suite (Minimal Kiosk Browser) - kweb"
		#kweb -KFJHCUA+-zbhrqfpoklgtjeduwxy "${KIOSK_URL}"
	done;

Set kiosk configuration in `/home/pi/.kiosk`:

	export KIOSK_OSD_NAME="WEB KIOSK"
	export KIOSK_URL="https://www.google.com"

## Optimization

For more info see this [link](http://elinux.org/RPiconfig) (to do).

## Finally

Reboot your pi and it should start X, launch web browser and be ready with your chosen web-page in kiosk-mode!
