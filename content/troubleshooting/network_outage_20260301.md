---
title: Network Outage 20260301
date: 2026-03-01
draft: false
description: Never trust SD cards.
tags:
  - network
  - ecowitt
  - timescaledb
  - pi
categories:
  - Troubleshooting
showToc: "true"
comments: "false"
searchHidden: "false"
---
Ah, Sunday.  The day of rest and relaxation... and troubleshooting why your entire network is down when you wake up.  Now, I'm not superstitious, but I am a little stitious, I should have seen this coming.  Last night, I woke up in the middle of the night and noticed my watch's screen was completely white and battery reporting 1%, which never happens as I charge every morning while I get ready and the battery will last until the next morning.  The watch is unrelated, but it was a sign for sure.

### Survey the Situation
Anyway, I wake up, no connection to Grafana to check my weather station data.  Morning briefing didn't fire at 06:15, so the network has been down for at least two hours.

Walking down the hall to let the doggo out, a quick glance shows all of the lights on the rack are lit up, so I've got power.  No notice from my ISP or power utility regarding any sort of outage.  I can't get any sort of connection to anything internal or external with my phone while having my coffee.  I check my DNS settings on my phone... my secondary server isn't on the list.  I add Quad9, boom, I've got network.  It's always DNS folks (until it's not).

I'll dig into the UDM and correct my DNS settings.  It turns out I applied Quad9 as my backup server on the UDM ISP settings, but each VLAN also had their own settings defined with only the Pi-hole being listed.  I'll go through and remove those per-VLAN custom settings.

The Pi that runs my Pi-hole, Ecowitt ingest, manual event logging API, NWS scrapers, and daily briefing was running the OS from the SD card that came with it.  When using Pi-hole, SD cards aren't huge fans of all of the read/write action going on, and with running other heavy read/write services on top of that, so it's likely the SD card corrupted.

Past me was ambitious and was thinking I'd build faster than I have, so I've got a second 8 GB Pi5 laying around.  I ordered SSDs and active coolers for both Pis as the plan was to boot from the SSDs and put them in my 1U GeeekPi chassis and run the two Pis as an HA pair.  A while ago, I built the second Pi with the active cooler and SSD installed, and put it in the chassis, I just hadn't gotten around to the configuration yet.  I guess today is that day.

### Correcting DNS
First, I'll manually modify the DNS settings on my laptop to use Quad9 so I can connect to my password manager and be able to get into all my devices.  This will allow me to log into my UDM to correct the DNS issue.  

I'll go to Settings > Internet > WAN1 and make sure my "global" DNS settings are correct.  I'll change my primary DNS server to the IP that the new Pi will have, and leave Quad9 as my backup for now.

Now, I'll go through each VLAN and remove the custom DNS settings for each VLAN and have them rely on the "global" DNS config from the WAN1 interface.

### Pi Configuration
Now that DNS is using Quad9 as the backup for now and the network is functional, I'll dig into configuring the new Pi.  

I'll grab the supplied SD card, and reimage it with Raspberry Pi Imager using Raspberry Pi OS Lite 64-bit, give it a hostname and enable SSH so I can run it headless from the start instead of having to hook up a monitor and keyboard.  I'll also grab the public SSH key that I used with the old Pi to use on this one for now.  I'll make a new pair for the old Pi when I bring that one back up later.

Now that I've got my hostname, user, and SSH configured and formatted, I'll pop the SD card into the new Pi, connect power, and make an Ethernet cable to connect to the pre-configured switch port.

While I'm waiting for the Pi to do its thing on first boot, I'll check Grafana, and it looks like the Pi went down between 03:00 and 04:00.  I'll make an annotation on one of my graphs to note the time of the Pi SD card corruption.

At this point, I've tried creating images multiple times and each time I'm having my SSH connections refused.  I've double checked my firewall rules and I should be able to SSH into the Pi.  The connection refused error indicates an issue with the Pi anyway, not the network.  

After some searching, I've found the solution is to manually add an ssh file with no extension to the boot partition.  This is recommended to enable SSH if the imager isn't working properly.  Once the image is created and fresh, I'll open up PowerShell, cd to the SD card, then enter 
```
New-Item ssh
```  

No file extension is added here.  I'll eject the SD card, then pop it back in the Pi.

Ok, I'm finally in using SSH and my SSH key.  Now updating the OS isn't working.  It looks like it's trying to use HTTP via port 80, which I'm blocking at the firewall, so I'll need to remove that rule at least temporarily.  I had blocked any HTTP traffic using port 80 to prevent unencrypted traffic from entering or leaving my network.  Updates were successful after removing the rule.

Now I need to check that the SSD is getting power and detected by running:

```
lsblk
```

I'll copy the OS to the SSD with the command:

```
sudo dd if=/dev/mmcblk0 of=/dev/nvme0n1 bs=4M status=progress 
```

then change the boot order to boot from the SSD with:

```
sudo raspi-config
```

I can now boot from the SSD successfully after shutting down and removing the SD card.  

Now I'm going to use SCP to copy all of my Python projects.  I'm going to start with my Ecowitt ingest service (an API that catches data from my weather station) so I can get data flowing into the database again.  To do that, I'll copy my project folder, create my new venv, create the service and start it.  Then I'll need to point the weather station console to push to the new Pi IP and API endpoint.  I'll also need to modify my firewall rules to allow the weather station console to talk to the new Pi and the new Pi to talk to the database.

I ran into some issues with getting the [Ecowitt Ingest service](/infrastructure/weather_station) back up.  I noticed when pointing the console to the new Pi IP that it had reset itself to point to the Ecowitt servers.  I think what it's doing is if it doesn't get a 200 response from its pushes for a given amount of time, it fails over and starts trying to send data to the Ecowitt cloud.  I reconfigured it and nothing appeared to be saving to the database.  After doing some troubleshooting, I noticed the timestamps from the raw data were "2024-01-01 ...".  So I checked my code and found no issues, it's the same code that was working just fine on the old Pi.  Then I checked to make sure the Pi had the correct date and time, which it did.  After rebooting the Ecowitt console, it had the correct timestamps and I can see data in Grafana and the database now.

Now I'll get my [Daily Briefing](/services/daily_briefing) and [NWS scrapers](/services/timescaledb) up and running.  These follow the same process as the Ecowitt ingest with the exception that these two run as cron jobs instead of services.  Then finally I'll set up my [Manual Event Logging service](/services/manual_events) with the same process.  After setting up each service, I test them to verify they're working properly before moving on to the next.

Now I'll set up logrotate for all of the log files each project drops.  I can do them all in one shot with the following config at path "/etc/logrotate.d/piservices" (using sudo nano):

```
/opt/ecowitt/error.log
/home/prax/daily_briefing/cron.log
/home/prax/weather/cron.log {
    weekly
    compress
    delaycompress
    missingok
    notifempty
    rotate 4
} 
```

That's all for today, it's been a long day.  Resolving this issue while doing laundry and other chores around the house took basically my whole day.  I still need to install Pi-hole, but that can be a project for another day.  The next chapter in this saga will be removing the old Pi from its case, rebuilding it with the new SSD and active cooler, putting it in the chassis with the new Pi, then configuring Pi-hole and HA.  The plan is to have the two Pis be twins that can take over for each other if one goes down and maybe do some load balancing.  I'll have to do some research and see what would be the best way to go about having some redundancy with this setup.
