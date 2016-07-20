---
layout: post
title:  "Install VMWare Workstation, create CentOS machine & configure bridge network for wlan0 and install Oracle 12c on ArchLinux 64bit"
date:   2013-11-27 21:18:10 +0200
categories: arch
tags: arch admin
comments: true
---	

<h2>Part I: Installation of VMWare Workstation</h2>

Download from official site: VMware-Workstation-Full-9.0.2-1031769.x86_64.bundle
from terminal:
	
<code>	
chmod +x VMware-Workstation-Full-9.0.2-1031769.x86_64.bundle
sudo ./VMware-Workstation-Full-9.0.2-1031769.x86_64.bundle
</code>

,then you install the patch from aur:

<code>
	yaourt vmware-patch
</code>

,then you install the kernal_headers and run this if path is not correct when you try to install them with the vmware gui.

<code>
	ln -s /usr/src/linux-3.11.4-1-ARCH/include/generated/uapi/linux/version.h /usr/src/linux-3.11.4-1-ARCH/include/linux/version.h
</code>

This is because the version of your kernal_headers should match with linux kernal version, so you might want to update the kernal as well.
, then run the

<code>
	sudo vmware-modconfig --console --install-all
</code>

and all modules should be up and running without failure.
,then you need to add the license key with

<code>
	sudo /usr/lib/vmware/bin/vmware-enter-serial
</code>

<h2>Part II: Configure the bridge network for the centOS machine</h2>

For the bridge networking to be working you need to explicitly point out the wlan0 is the bridge internet adapter. you need to do this:

<code>
	sudo /usr/lib/vmware/bin/vmware-netcfg
</code>

in this popup set the vmnet0 (which is not shown in the ifconfig of the host) and set it to wlan0. Otherwise it is set to automatic and picks the eth0 instead of wlan0.
After you power on the virtual machine you need to set the networking in the centos. go to:

<code>
	cd /etc/sysconfig/network-scripts/
</code>

i had eth3 instead of eth0 (centos's ifconfig), so i copy the the ifcfg-eth0 to 3 and change the network to (00:50:56:34:7C:AE, which you get if you clicked bridged and change the networking mac address, which i did), then the file should look like this, if you copy it:

<code>
	scp ifcfg-eth3 192.168.1.100:/home/ljupcho/Downloads
</code>

file:

<code>
	DEVICE=eth3 
	HWADDR=00:50:56:34:7C:AE 
	TYPE=Ethernet
	UUID=1f969148-80a4-4105-ab9e-dcff960fe8f8
	ONBOOT=yes
	NM_CONTROLLED=yes
	BOOTPROTO=dhcp
	GATEWAY=192.168.1.1
</code>

restart networking

<code>	
	/etc/init.d/network restart; yum update
</code>

<h2>Part III: Install Oracle 12c on CentOS 64bit</h2>

These are the key points for the oracle install. You should give at least 20GB space for you VM so you won't have to add more latter. 
for this error: bash: xhost: command not found 
this helped me:

<code>
	yum install xorg-x11-server-utils
</code>

for this error: xhost unable to open display
First I had to install:

<code>
	yum groupinstall “X Window System”; init 6
</code>

then this in terminal:

<code>
	[root@centos ~]# export DISPLAY="127.0.0.1:10.0"
	[root@centos ~]# xhost +
	access control disabled, clients can connect from any host xhost:  must be on local machine to enable or disable access control.
</code>

, then you need to ssh to the centOS machine with the oracle user you've created, but you do it with: ssh -Y oracle@192.168.1.107 instead of ssh -X, then navigate to database folder of your oracle 12c archive and ./runInstaller. You would do a -Y because the input fields of the installer Oracle GUI won't be editable, you won't be able to type in. (this problem also might be because you have java 1.7 so if issue still exists downgrade to java 1.6) . Then follow the standard Oracle 12c installation tutorial with screen shots, you should be fine.