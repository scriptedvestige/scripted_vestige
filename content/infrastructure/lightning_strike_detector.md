---
title: Lightning Strike Detection
date: 2026-04-10
draft: false
description: The first piece of my wildfire alert system.
tags:
  - ecowitt
  - timescaledb
  - grafana
  - fastapi
  - data
categories:
  - Services
showToc: "true"
comments: "false"
searchHidden: "false"
---
Alright, so we're adding more sensors to the stack!  I went with more Ecowitt sensors because they'll just connect to the existing console and send data through the existing pipeline.

### Hardware
I'm going to start with the WH57 lightning strike detector.  I've selected this sensor to pair with an AQI sensor to build a home brew wildfire detection and alerting system.

The lightning strike detector will detect strikes up to 25 miles out.  The physical installation is easy and it comes with the mount and hardware necessary, just add batteries.  

### Pipeline Integration
(The code for this project can be found on my [GitHub](https://github.com/scriptedvestige/ws_ingest.git).)

Next, I'll edit my existing api.py script on my Raspberry Pi to log the raw payload that it's catching from the console.  I'll also log into the console through the browser and set my pushes to come every 60 seconds instead of 5 minutes so I can build quickly.

I can see what fields were added with the new sensor reporting, so now I'll modify my process.py script which breaks the monolith of data down into smaller pieces to account for the new lightning strike data.  I'll also create a new hypertable in my Timescale instance to hold all of that data.

I'll use SCP to push my changes from my laptop to my Pi, then I'll restart my Ecowitt Ingest service.  I'm now seeing some errors in my error.log file about data not being able to be inserted into my tables because the lightning strike data has empty strings.  I'll modify the process.py to replace empty strings with NULL values.

I can now see rows populating in the lightning_events table and the ws_observations table is populating new rows again as well (one issue with the errors is it stops inserting to *all* tables instead of just the one that has issues, I'll need to address that down the road).  Now that data is inserting properly, I'll comment out the function to save the raw data, as that is not needed for now.

![Lightning Events Table](/services/lightning_events.png)

On the bright side, because of the issues that occurred, I can see that my data gap detection is working and I can see my 13 minutes of downtime.  Obviously I know what the cause was this time, but this could be useful to monitor blips in the pipeline and allow me to investigate what issue may have occurred with accurate timestamps to identify exactly when an issue has occurred.

This also shows me that my pipeline is pretty flexible and scalable, which is great.  I've still got eight soil moisture sensors and an AQI monitor to add in the future, so this exercise demonstrated that shouldn't be a huge issue.  I'm feeling good about that.  Now I'm going to sit back and hope the thunderstorms that are forecast for today give me some data for my new sensor!

I'll make a quick and dirty Grafana dashboard with a panel that just pulls the raw data from the DB.  Once I get some real data, I'll likely modify the Grafana table to only show strike data and no NULL events.  I'll add some more panels for strike counts, distance, etc. and start to shape what the wildfire alert system is going to look like.

![Grafana Table](/services/grafana_lightning.png)

##### Troubleshooting
I found when I had some data to report from the lightning strike sensor that the ingest was broken, which brought everything down.  The issue was that the time being reported was in Unix Epoch format, which didn't mesh with my table schema using TIMESTAMPTZ format.  I added a function to convert Unix Epoch time into UTC time to play nice with the table schema.  While writing that function, I also consolidated the logic to change empty strings to return None.  Technically those should probably be two distinct functions, but I'm trying to move quick here to get everything back online.  Quick and dirty now, I'll polish later.

Another thing that I needed to do was convert the strike distance from kilometers to miles (I live in the US, so kilometers does not compute in my brain), so I handled that in my conversions function as well.  Then I needed to rename the field from 'lightning' to 'lightning_mi' to match my table schema which I implemented after I split the main payload into smaller chunks.

### Going Forward
This is the first piece to a custom wildfire alert system that I'll be building.  There are apps and sites that offer the information that I need, but I noticed last year that updates were slower than years prior.  Given that I live directly next to wild land, and if I'm at work it could take me an hour or longer in heavy traffic to get home, I need to be able to act quickly if I need to get home and evacuate my dog.

The other pieces of the puzzle here will be adding an AQI sensor to detect levels of smoke in the air.  I'll also grab the URLs for some public wildfire cameras deployed by the local university that look directly at my house.  I'll create a Python script that will determine the potential threat given things like days since last rain, temperature, wind speed and wind direction along with the lightning and AQI data.  If an alert needs to be sent, I'm thinking a private Matrix server paired with Tailscale and using a Matrix bot to send the alert with the link to the cameras.  I think this will be an interesting project that has real world safety impact.