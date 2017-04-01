---
layout: post
title: A Super Scaling Story
author: flock3
---

InfinityWorks were recently invited to talk at a North London BCS event, we brought along two presentations with which to tempt the audience, "Super 6 – New Adventures in DevOps" and "SUS+ - Challenging perceptions of NHS IT".  The latter talk was present by the inimitable Dan Rathbone and of IW director fame, and Ed Hiley of NHS fame, their talk was both exceptionally well presented, and very interesting, you should certainly watch it!  I was involved however in the Super6 story, and was asked to perform a live demo during the event, to show off some of the tech we've brought to the Super6 stack, and see what can be done quite simply with modern technology.

----
***

I only had a couple of days in which to prepare a live demo, so I chose a subject that's pretty engaging, and yet I knew well enough to be able to answer any question that might popup.  I chose "Super6 - A scaling story" as my (in head only) title for my demo.  I only had a roughly 10 minute slot to demonstrate how we scale our stack, so with a little preparation, here's what I did.

## Why do we need to scale?

The Super6 talk (delivered by Ed Marshal and Paul Henshaw most recently) covers a wide variety of topics, from the business side of our relationship with SBG, to the practical issues with scaling a website to deal with ~1.2M players a week, with peaks of 200,000 requests/minute.

One of the biggest problems with that scale is not the overall size (at highest load, we don't need to run more than 40 or 50 web servers) but the massive spikes in traffic that we might recieve.

Below is a graph of one of the traffic spikes from this weekend, to serve as a practical example:

![Super6 Spike]({{ site.baseurl }}/images/super-scaling/super-scaling-spike.png "Super6 Spike")

As you can see from the above image, we were hovering fairly steadily (in that case) at around 50K/minute requests, but then within a 1 minute period, we doubled that to 100K minute, and then to 120K/minute.  As anyone who operates on any cloud platform knows, yes, you do have a basically unlimited level of scale, but increasing scale is not nessicarily fast.

During the demonstration I gave on Wednesday, it took about 6 minutes for us to bring 8 new web servers online and fully operational. In the above graph, we the traffic would have started to decline greatly after that 6 minute period (back down to around 80K/minute), if we had started to scale as soon as we saw that first spike (and our services were running at their normal scale already) we would have not been able to serve most of those requests within a reasonable time frame, and that would have been a reduction in service, and we avoid that at all costs.

The only solution then to deal with that, is to have our servers already there and available to serve requests, before we get these big spikes. But we can't sit there all week with 40 web servers ticking away. Part of why SBG asked us to come and help with Super6 is to reduce the cost,  but the other factor for them was to reduce the impact on their business (as they focused more on their own offerings), so we can't have a service incident, and we don't want to waste the client's money - so our solution is to model our traffic.

We've been running the Super6 service now for about a year, so we've got a ton of data as to how the game works, and how our users behave.  When Jeff Stelling on Sky Sports announces his Super6 predictions for the week, we know that will generate a huge surge in traffic as all of the people watching Sky Sports, either remember that they haven't put in their predictions yet, or go into review/adjust the prediction they made earlier.

We also know that the marketing team are likeley to send out notifications to mobile devices, and emails at around the same times every week, so we can be prepared for them.

So now we know when we need to scale, let's go into the fine detail about what happens when we scale.

## The lifecycle of a Super6 web server

**-- Go into the auto scaling group, and add 10 new web hosts**

**Here’s what’s going to happen in terms of actions**

*   **Auto scaling group receives the request and adds 10 more web servers**
*   **Amazon spin up another 10 nodes for our stack to sit on**
*   **Each of those servers has the tag “web” on it, allowing Rancher to know what containers to schedule onto it**
*   **New nodes come online with a Rancher OS AMI**

**-- Go to CloudFormation Tab **

*   **When a server comes online it needs needs to know where it’s scheduler is, so we set that in CloudFormation**
*   **Demonstrate some of the options we have in CloudFormation and show the instance sizes and everything else as options in there**
*   **As soon as a node comes up it registers itself into rancher with a token set in our CloudFormation configuration**

**-- Go to Rancher Tab -- Click Infrastructure**

*   **Talk about the types of servers that make up the Super6 Stack

    *   Monitoring 
        *   ES
        *   Kibana
    *   PHP App
        *   Admin Area
        *   Service endpoints
        *   Vault / Authentication Exchange
    *   Webs
    *   App Servers
        *   Calc Engine

    **
*   **Show them an existing web and talk about all of the components 

    *   Fluentd for web log collection
    *   Node exporter for prometheus metrics
    *   Super6-web-blue
    *   Super6-web-green
    *   Super6-bluegreen-lb
    *   Super6 Load Balancer
        *   Stovepipes - Avoids rancher nasty during upgrades
        *   HAProxy Load Balancer
        *   Each container queries Redis to find out if it is in load at that moment
            *   Returns 200 if it is ok to accept queries, returns 404 if it is not
                *   404 Tells HaProxy to drain connections from it if it has any, and not to create new ones

    **

**-- Stay in Rancher Infrastructure page**

*   **Now going to talk about the startup of a container, and about the vault and authentication exchange containers I mentioned earlier on**
*   **Explain that I’m a huge fan of Hashicorp Vault**
*   **Vault wasn’t our first solution, originally we had Concourse in scope**
*   **Explain that vault was a completely new tool to us, and our first solution could have been better, but that we iterated on it, and we’re pretty happy with it**
*   **Container Start Process

    *   When each container loads, the container sends it’s container ID over to the authentication engine
        *   Auth engine queries rancher to find what roles the container is allowed
            *   Show rancher labels
        *   Auth engine generates a vault token for that container 
        *   Container queries vault and retrieves it’s secrets, loads then into environment
        *   Application reads secrets from environment and starts

    **

**-- _HOPEFULLY_ containers will be starting by now**

*   **Talk about Super6-BlueGreen-LoadBalancer**
*   **Talk about the old “Rancher load balancer” down-scaling causing the ELB to think the wrong servers were out of service**
*   **Talk about the deployments that caused a slight outage (of a few seconds)**
*   **Talk about what the stovepipe gives us**
*   **Talk about how we swap between the blue/green, and how each stack knows if they are in service**
*   **Talk about HAProxy draining connections on a 404 from a health check**
*   **Talk about ALB healthchecks and Shallow and deep pings**
*   **Talk about CloudFront serving our error pages**

**-- Go into Grafana and wait for new servers to come into service**

*   **Talk about each of the panels in a fair amount of detail**
*   **Explain that I am running some test load through just to give them a base line**
*   **Explain that SBG service desk see the same monitoring and monitor the same stuff we do**
*   **---- Need verification here ---- Talk about what happens when the load is too high**

**-- Go back to Amazon and kill off the web servers**

*   **When server is no longer needed I.e. we’re not under heavy load**
*   **Server is killed through AWS console**
*   **HaProxy detects the newly failed health check, immediately removes that container from service**
*   **Redirects requests to other active containers**
*   **We can scale from 40 hosts down to 2 hosts without impacting service**