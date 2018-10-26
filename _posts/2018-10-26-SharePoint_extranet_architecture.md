---
layout: post
title: "SharePoint extranet architecture"
description: "Building a corporate Extranet web site in SharePoint"
date: 2018-09-06
tags: SharePoint ADFS authentication
---

## Overview
Many companies deploy so called "Extranet" web solutions or portals, often in DMZ network segment, to publish not-so-public restricted content, not suitable for the public web site and the general Internet audience and make it accessible for certain clearly identified users representing partner organizations, registered customers or other external entities. The extranet web site can also serve as a some kind of a "meeting place", where the corporate users accessing this web site from within the company's internal network can collaborate with the partners and customers in a secure fashion and participate in common business process workflows stretching across the organizational boundaries. One of the popular platforms for building such solutions is Microsoft SharePoint Server (on-premises version), and this article explores the ways this technology can be used in the Extranet portal scenario.

## Cloud VS on-premises
There are two completely different approaches to using Microsoft SharePoint for building Extranet:
- Cloud-based SharePoint Online (part of Office 365);
- On-premises SharePoint farm.
Microsoft suggests [here][Use SharePoint Online as a business-to-business (B2B) extranet solution] that SharePoint Online offers the most effective and easiest way to deploy Extranet solution. I agree with this, but sometimes it is not possible or desirable to use the cloud, and companies in many cases are left only with the on-premises option. Here are some reasons why company may decide that SharePoint Online in not on the table:
- regulatory/legal restrictions in some countries, when it is prohibited to store and process certain types of information outside the countries boundaries;
- Internet channel availability - when some of your users just can't access the O365 cloud;
- application compatibility issues - some third-party or in-house developed SharePoint applications cannot be deployed to SharePoint Online (legacy farm solutions).

## Separate farm for Extranet
One other question that we need to answer - if we stick with the on-prem SharePoint, why not just use the same single farm for Intranet and Extranet? I think that it's an option worth considering, especially for small companies where external access is limited to only few highly trusted entities. But for many larger companies this approach poses a number of challenges:
1) Do we need to publish the internal SharePoint farm to the Internet?
2) If the farm is not published and the corporate users access it via VPN line when roaming, do we grant VPN access to all those external partners and customers as well?
3) How do we authenticate external users? Do we create Active Directory logins for each customer and partner? How do we control what they do with the live AD credentials that we've entrusted them with?
4) If do not like the idea of creating AD user accounts for external parties (why we should?) and we come up with some secondary authentication scheme for external users, how do we guarantee that these external users do not have access to some highly sensitive confidential information on our SharePoint, either by malice or mistake?
5) Do we want to make our Intranet farm more complex and fragile by introducing additional authentication schemes for external users, mess up with separate user sources for User Profile Service and People Picker and configure all those additional IIS web sites we might need for this? Or is it better to use only Windows authentication for the Intranet farm and keep it simple for our corporate users, SharePoint developers and farm admins alike?

**In many cases, deploying a new separate SharePoint farm in DMZ network segment is the best answer.**

## Additional costs
The decision to deploy a separate SharePoint farm for Extranet comes with its own price. This statement in [Microsoft article][Use SharePoint Online as a business-to-business (B2B) extranet solution] is quite true: *"...deploying a SharePoint on-premises extranet site involves complex configuration to establish security measures and governance, including granting access inside the corporate firewall, and expensive initial and on-going cost"*. In addition to the ShareFarm, we will generally have to deploy many other supporting servers and services, such as AD domain controllers, load balancers, SQL servers, reverse proxy and ADFS servers. However, the initial costs associated with the deployment of such supporting infrastructure may be a good investment and will bring huge benefits in future. For example, an isolated AD forest/domain in the DMZ segment may well become the basis and authentication backend for other externally facing IT services beyond SharePoint Extranet portal. 

## SharePoint Extranet architecture
There are many choices we have to make when we design Extranet solution on SharePoint platform. In this article I will describe the configuration that we successfully deployed and have run for several years, but a good research of available options should be done for each new deployment to explore the alternatives and find the best solution for each case.

**Example SharePoint Extranet architecture:**

| Subsystem | Suggested implementation | Comments |
|-----------|--------------------------|-------------------------------|
| SQL Backend | AlwaysOn AG | In my opinion, Availability Groups in SQL Server Enterprise Edition are the best way to provide highly-available SQL backed for SharePoint, especially if spread your replicas across two or more datacenters. I also had some experience with SQL Mirroring available in SQL Server Standard edition, but this technology is not that easy to administer and have a major drawback - **a part** of farm databases may failover to the secondary instance while other databases remain active on primary, in which case the farm goes down and a recovery is needed. |
| SharePoint farm topology | 2 WFE + 2 APP + OWA | I strongly advice to go for multi-server deployment and deploy at least two servers for web front end role and two server for app server role. Please review Microsoft recommended topologies for SharePoint 2016/2019 [here][Technical diagrams for SharePoint Server]. You might also want to deploy Office Web App (OWA), in a single server of farm configuration, but this is beyond the scope of this article. |



## References 
1. [Use SharePoint Online as a business-to-business (B2B) extranet solution][Use SharePoint Online as a business-to-business (B2B) extranet solution]
2. [Technical diagrams for SharePoint Server][[Technical diagrams for SharePoint Server]]

[Use SharePoint Online as a business-to-business (B2B) extranet solution]: https://docs.microsoft.com/en-us/sharepoint/create-b2b-extranet 

[Technical diagrams for SharePoint Server]: https://docs.microsoft.com/en-us/sharepoint/technical-reference/technical-diagrams


