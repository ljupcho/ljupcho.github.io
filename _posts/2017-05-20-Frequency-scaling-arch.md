---
layout: post
title:  "CPU Frequency Scaling with Arch Linux"
date:   2017-05-20 19:18:10 +0200
tags: frequency scaling archlinux
comments: true
---	

I wouldn't go into installing and configuration of Arch Linux, the wiki is accurate, simple and details enough. After installing the main configuration I do is CPU frequency scaling, because after installation the kernel will be set to performance governor and as a result the fans will start running very often. What I want is a quite laptop, I appreciate the silence.

The wiki page I am talking here about is https://wiki.archlinux.org/index.php/CPU_frequency_scaling. In short, I have tried 2 drivers intel_pstate and acpi-cpufreq. The first is by default with performance already set. Now, I have changed that to powersave governor (sudo cpupower frequency-set -g powersave). There is immediate difference. Using the acpi driver I have tried conservative governor and that one is also good, but I've read the Intel managed to make powersave governor with better characteristics rather than the others governors from the acpi driver. I check the governers by driver using: cpupower frequency-info, by available cpufreq governors. 

The details from frequency-info give information about the frequency range and what I would like to have is my laptop running on lower frequency, meaning I am not partially interested in coping files faster or other CPU related task perform quicker, it is quick enough with powersave as well. Having that with combination of SSD and you get the best combination of laptop, as CPU is very sensitive on I/O when hitting the hard disk.

There are few more tweaks I do in order to make the laptop run smoother. I use powertop (https://wiki.archlinux.org/index.php/powertop). After installing powertop and run it (sudo powertop --html=powerreport.html), you get a html page and under Tunning tab I copy all the Script commands from Software Settings in Need of Tuning except for the mouse command and put them in powertop-fix.sh. You may set it to auto-tune, but I had problems with my optical mouse, so tried a different approach. I call that script on boot after I set it under: sudo nano /etc/systemd/system/powertopfix.service

<code>
[Unit]
Description=Powertop Service Fix
[Service]
Type=oneshot
ExecStart=/home/ljupcho/powertop-fix.sh
[Install]
WantedBy=multi-user.target
</code>

You'd start the script on boot with: <code>sudo systemctl enable powertopfix.service</code>


