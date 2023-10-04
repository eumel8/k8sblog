---
layout: post
tag: en
title: Promcon EU 2023 Recap
subtitle: A review of the Promcon 2023 in Berlin
date: 2023-10-01
background: '/images/k8s-cosmos.png'
twitter: 'images/k8s-blog2.png'
author: eumel8
---

# Bring the fire

Did you know? Prometheus, the de facto standard monitoring system for Kubernetes, was invented 2012 in Berlin by a company named Soundcloud? Soundcloud was (or is still) a music streaming service, which required a scalable monitoring system wide before Kubernetes was born. Luckily enough the software goes Open Source 2015 and was CNCF graduated one year later. Soundcloud was not so sucessul, so many people have to left the company and landed at Polarsignals, also located in Berlin, or moved on to Grafana Labs, a world-wide operating company with nearly 1000 employees. And many of them joined us today.

<img src="/images/promcon/promcon17.jpg" width="925" height="525"/>

These and other facts was presented as kickoff by Matthias Loibl ([@metalmatze](github.com/metalmatze)). We are at [Radialsystem](https://www.radialsystem.de), a formely pump station in the middle of Berlin (Germany) directly on the river Spree. The weather couldn't be better for this 2-days-CNCF event: PromCon EU 2023. Sunny, nearly 25 degrees, and it's the end of September. 
# Venue

The venue was divided in a theatre for the conference track, a second hall for taking food and conversation, and additionaly and outside area to take seat on a comfortable chair with view to the Spree. 

<img src="/images/promcon/promcon15.jpg" width="925" height="525"/>

<img src="/images/promcon/promcon18.jpg" width="925" height="525"/>

<img src="/images/promcon/promcon19.jpg" width="925" height="525"/>

<img src="/images/promcon/promcon2.jpg" width="925" height="525"/>

Speaking of theatre: Did you know [Prometheus The Documentary](https://youtu.be/rT4fJNbfe14)? A short movie about the history of Prometheus. And the second movie tip: There was a [live stream](https://www.youtube.com/@PrometheusIo/streams) of the whole conference, which is still available on Youtube. So, great prepared and well working infrastructure. Not to forget the excellent catering by [Hoflieferanten Berlin](https://hoflieferanten.berlin/). Many of the conference participants have now an idea about Königsberger Klopse.

# Prometheus Project Update

<img src="/images/promcon/promcon3.jpg" width="925" height="525"/>

Head-lighted features since last KubeCon:

- string-labeling for performance improvements
- Java Client 1.0.0
- New Notifiers in Alertmanager like Discord, Webex, and MS Teams

Community work was presented at [https://prometheus.io/community](https://prometheus.io/community). Not to forget the Dev Summit, which was held a day after the PromCon on Saturday in Berlin. There is a live agenda on [Google Docs](https://docs.google.com/document/d/11LC3wJcVk00l8w5P3oLQ-m3Y37iom6INAMEu2ZAGIIE/) full packed with links and informations. Beside the in-person event, where also guests are welcomed, a live stream on Google Meet was also available.

This comes to topic: what's next on Prometheus? Maybe a new Alertmanager UI will be helpful. Remote Write v2 will have more performance improvements.

<img src="/images/promcon/promcon20.jpg" width="925" height="525"/>

# Breakout Sessions

Not really breakout because we are still in the theatre. There is nothing to move in other rooms and buildings. An advantage of the conference with around 100 participants. 

## Open Telemetry

One of the biggest thing in the future will be the change to Open Telemetry Standard. Goutham Veeramachaneni and Jesus Vasquezfrom Grafana Labs drawed attention on it and made a comparison.

## Beyla

Another Grafana Labs session by Nikola Grcevski and Mario Macias presented [Beyla](https://github.com/grafana/beyla), a tool for zero-code application metrics with eBPF and Prometheus. Automatic instrumentation of a service with [eBPF](https://ebpf.io/), very interesting to see.

## Alert analytics

<img src="/images/promcon/promcon4.jpg" width="925" height="525"/>

A session from the field from Monika Singh by Cloudfare, how to handle Alertmanager data in bigger environments. Working example in [this](https://github.com/m0nikasingh/am2ch) Github repo.

<img src="/images/promcon/promcon5.jpg" width="925" height="525"/>

<img src="/images/promcon/promcon6.jpg" width="925" height="525"/>

## Finding useless and resource-hungry Prometheus metrics

<img src="/images/promcon/promcon7.jpg" width="925" height="525"/>

By David Calvert, best session so far. Every Prometheus user is overwhelmed with tons of metrics, which blows up the Prometheus Pod and eats up all memory of the Kubernetes Node, where the Pod is running. Now it's time to make a deep inspection, which metrics are really used and which are only stored in the Prometheus database. [Here](https://0xdc.me/blog/how-to-find-unused-prometheus-metrics-using-mimirtool/) is a blog post about three elementary things.

## Planet scale monitoring: Handling billions of active series with Prometheus and Thanos

<img src="/images/promcon/promcon8.jpg" width="925" height="525"/>

I was impressed, [Shopify](https://github.com/shopify) has nearly thousand public repos on Github, a brave Open Source company. What we've got to see from Sebastian Rabenhorst and Mikołaj Liberski were some high-level global architecture picture from the Shopify Operations, but it didn't quite fit the frame of the audience. PromCon is a conference for really tech. No marketing talks, no ads. I didn't catched the story of this session, while the real technical background remained hidden for confidentiality reasons or whatever.

<img src="/images/promcon/promcon9.jpg" width="925" height="525"/>

## Using Green Metrics to Monitor your Carbon Footprint

<img src="/images/promcon/promcon10.jpg" width="925" height="525"/>

Just another session by Grafana Labs by Ida Fürjesová and Niki Manoledaki. Are we aware that data center have the most carbon emission of the world? More than aviation? Climate adhesive would block our conference immediately when this becomes known. So, it's really time for GreenOps. But it's a different field for measure carbon footprints on container level. Cloud Providers like AWS provides some high-level data about your workload into the bill. With [Kepler](https://github.com/sustainable-computing-io/kepler) you can measure this at your own. But needs raw access to the Kubernetes nodes and works only in a region which is known by [https://www.watttime.org/](https://www.watttime.org/) with an efficient data flow in real time.

<img src="/images/promcon/promcon11.jpg" width="925" height="525"/>

<img src="/images/promcon/promcon12.jpg" width="925" height="525"/>

<img src="/images/promcon/promcon13.jpg" width="925" height="525"/>

<img src="/images/promcon/promcon14.jpg" width="925" height="525"/>

## Lightning Talks & Quiz

Each conference day ended with a bunch of lightning talks. Sessions with a length of 5 minutes, very spontaneous to different topics. Very interesting and funny. Also funny was the Surprise Trivia Quiz in the beginning of the next day. 

<img src="/images/promcon/promcon29.jpg" width="925" height="525"/>

## From Metrics to Profiles and back again

By Frederic Branczyk, the founder of Polarsignals. A very interesting session about Debugging and Profiling an application.
On [Youtube](https://www.youtube.com/@PolarSignalsIO/streams) there are various live hacking sessions. There is also this [Profile Exporter](https://github.com/polarsignals/profile-exporter) on Github available.

## Where's your money going? The Beginners Guide To Measuring Kubernetes Costs

<img src="/images/promcon/promcon22.jpg" width="925" height="525"/>

Erik Sommer, a young and ambitious SRE from Grafana Labs, explained the possibilities of KubeCost and Grafana to find out where's your money burned in load peaks while memory consumptions of applications.

<img src="/images/promcon/promcon23.jpg" width="925" height="525"/>

After the lunch of the second day it comes to a point where we got to much metrics and Prometheus topics. Right away:

## Metrics Explorer: A tool used to learn about Prometheus

<img src="/images/promcon/promcon24.jpg" width="925" height="525"/>

Brendan O'Handley and Catherine Gui from Grafana Labs draw attention to Prometheus users on beginners level. How to find rigth metrics of which metrics are present in general? For this the Grafana Metrics Explorer will help with a simple menue and overview. 

<img src="/images/promcon/promcon26.jpg" width="925" height="525"/>

## Learning from Mistakes - Choosing the Right Metrics for Prometheus Alerting

<img src="/images/promcon/promcon27.jpg" width="925" height="525"/>

Ankur Rawal and Dheeraj Reddy from Zenduty presented some really nice examples of [bit.ly/promalerts](Prometheus Alertrules)

And that was the PromCon 2023 in Berlin. See you next year at PromCon 2024 in Berlin!

<img src="/images/promcon/promcon28.jpg" width="925" height="525"/>







