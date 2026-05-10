---
title: Manual Event Logging
date: 2026-02-14
draft: false
description: Logging things my sensors can't.
tags:
  - grafana
  - timescaledb
  - fastapi
categories:
  - Services
showToc: "true"
comments: "false"
searchHidden: "false"
---
There is a lot of data that I want to collect that my sensors can't.  With automation and machine learning application being the goal, I want to gather as much data as I possibly can to really create a complete picture of events in the yard.  Snow events are one example of that, so that's where I started.

[Check out the repo on GitHub](https://github.com/scriptedvestige/manual_events.git)

I again used Python, FastAPI, and uvicorn to define some API endpoints on the same Pi that runs my Ecowitt ingest API.  Just like the Ecowitt ingest, I'll make this API a service also.  The difference with this one though is that when the endpoint is requested in the browser, an HTML form is presented.

![Field Observations Service](/services/field_observations_service.png)

This form allows me to select event type (fresh, dusting, or melt), zone (backyard versus driveway), depth, snow type (wet versus dry), and allows me to add notes.  When type is selected as dusting, there is a little JavaScript function that hides the depth field.  The notes section has come in handy when troubleshooting issues with my stack (see [Network Outage 20260224](/troubleshooting/outage_20260224)).  When the submit button on the form is clicked, the data is inserted into a table in the database with date and time the form was submitted being set as the timestamp.  

![Snow Event Form](/services/snow_event_log.png)

Then I made a dashboard in Grafana to track snow events logged within the last *n* days.  Right now, I don't have any snow events appearing on the panel given the time frame, but here's a look at the underlying table.

![Manual Snow Logging](/services/manual_snow_events_table.png)

Some other manual event logging that I'll implement:
- Watering Events.  While the goal is to eventually have my irrigation system automated and become a closed loop system, I'm not there yet.  Even when that has been implemented, I still may need to water manually early on in the automation phase while I dial the system in.
- Fertilizer Events.  Dates of fertilization of plants, type of fertilizer used (nutrient breakdown), amount used.
- pH Measurements.  This data can be analyzed alongside the fertilizer events to see the effect of the fertilizer on the soil over time.
- Grape Phenology and Brix.  Bud break, shoot and leaf growth, flowering, fruit set, veraison events. Brix readings over time (possibly in another form and table).  The goal with tracking these things is to predict ideal harvest dates for each type of grape grown.
- Pruning Events
- Maintenance Events

At the end of the day, just like the Ecowitt ingest this setup is pretty basic and not very difficult to deploy, but returns a lot of value.  I've got the basic framework in place with the snow logger to set up other manual event logging by copying, pasting, and modifying what already exists.