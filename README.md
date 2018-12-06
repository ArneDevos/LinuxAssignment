# LinuxAssignment
Final Assignment for the elective course Linux

## Start Pi
# Install lxc

$ sudo apt-get update
$ sudo apt-get isntall lxc
$ sudo modprobe configs
$ lxc-checkconfig

# Everything looked fine, we can continue

# Set up the Iptable so requests to the pi are forwarded to the first container
# with ip adress 10.0.3.11 (we will assign this one later), THIS SHOULD BE DONE EVERYTIME THE PI STARTS. 

$ sudo iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 80 -j DNAT --to-destination 10.0.3.11:80
$ sudo sysctl net.ipv4.ip_forward=1

# Check the forwarding

$ sudo iptables -t nat -L

# Enable unprivileged usage

# Check the values in subuid and subgid

$ grep $USER /etc/subuid
$ grep $USER /etc/subgid

# Add these values to ~/.config/lxc and config the bridge

$ mkdir -p ~/.config/lxc
$ echo "lxc.id_map = u 0 100000 65536" > ~/.config/lxc/default.conf
$ echo "lxc.id_map = g 0 100000 65536" >> ~/.config/lxc/default.conf
$ echo "lxc.network.type = veth" >> ~/.config/lxc/default.conf
$ echo "lxc.network.link = lxcbr0" >> ~/.config/lxc/default.conf
$ echo "$USER veth lxcbr0 10" | sudo tee -a /etc/lxc/lxc-usernet

$ sudo nano /etc/lxc/default.conf 
	lxc.network.type = veth
	lxc.network.link = lxcbr0
	lxc.network.flags = up
	lxc.network.hwaddr = 00:16:3e:xx:xx:xx
$ sudo nano /etc/default/lxc-net
	USE_LXC_BRIDGE="true"

# Make the ip adresses of the containers static

$ sudo nano /etc/lxc/dhcp.conf
	dhcp-host=Con1,10.0.3.11
	dhcp-host=Con2,10.0.3.12	
$ sudo nano /etc/default/lxc-net
	LXC_DHCP_CONFILE=/etc/lxc/dhcp.conf

# Restart the lxc service

$ systemctl restart lxc-net

# Let's create the first container

$ lxc-create -n Con1 -t download -- -d alpine -r 3.4 -a armhf

# Start the container

$ lxc-start -n Con1

# Go inside the container

$ lxc-attach -n Con1

# Install lighttpd and php

$ lxc-attach -n Con1 -- apk update
$ lxc-attach -n Con1 -- apk add lighttpd php5 php5-cgi php5-curl php5-fpm

# Uncomment the include "mod_fastcgi.conf" line in /etc/lighttpd/lighttpd.conf

$ vi /etc/lighttpd/lighttpd.conf

# Start the lighttpd service

$ rc-update add lighttpd default
$ openrc

# Create /var/www/localhost/htdocs/index.php and add the following:
	
	<!DOCTYPE html>
	<html><body><pre>
	<?php 
		// create curl resource 
		$ch = curl_init(); 
		// set url 
		curl_setopt($ch, CURLOPT_URL, "Con2:8080"); 
		//return the transfer as a string 
		curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1); 
		// $output contains the output string 
		$output = curl_exec($ch); 
		// close curl resource to free up system resources
		curl_close($ch);
		print $output;
	?>
	</body></html>

# Create a second container

$ lxc-create -n Con2 -t download -- -d alpine -r 3.4 -a armhf

# Install socat

$ apk add socat

# Make a script called bin/rng.sh and make it executable

$ vi bin/rng.sh
$ chmod +x bin/rng.sh

# Add this to the script:

	#!/bin/ash

	dd if=/dev/urandom bs=4 count=16 status=none | od -A none -t u4

# Serve the script with socat

$ socat -v -v tcp-listen:8080,fork,reuseaddr exec:/bin/rng.sh

# Now check what the ip adress is op wlan0

$ ifconfig

# In a browser, go to this adress and everything should work perfectly

