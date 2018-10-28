---
layout: post
title: "SharePoint Extranet architecture"
description: "Building a corporate Extranet web site in SharePoint"
date: 2018-10-26
tags: SharePoint ADFS authentication
---

## Overview
Many companies deploy so called "Extranet" web solutions or portals, often in DMZ network segment, to publish not-so-public restricted content not suitable for the public web site and the general Internet audience and make it accessible for certain clearly identified users representing partner organizations, registered customers or other known external entities. The extranet web site can also serve as a some kind of a "meeting place", where the corporate users accessing this web site from within the company's internal network can collaborate with the partners and customers in a secure fashion and participate in common business process workflows stretching across the organizational boundaries. One of the popular platforms for building such solutions is Microsoft SharePoint Server, and this article explores the ways in which the on-premises version of this technology can be used in the Extranet portal scenario.

## Cloud VS on-premises
There are two completely different approaches to using Microsoft SharePoint for building Extranet:
- Cloud-based SharePoint Online (part of Office 365);
- On-premises SharePoint farm.

Microsoft suggests [here][Use SharePoint Online as a business-to-business (B2B) extranet solution] that SharePoint Online offers the most effective and easiest way to deploy Extranet solution. I agree with this, but sometimes it is not possible or desirable to use the cloud, and companies in many cases are only left with the on-premises option. Here are some reasons why company may decide that SharePoint Online in not on the table:
- regulatory/legal restrictions in some countries, when it is prohibited to store and process certain types of information outside the countries boundaries;
- Internet channel availability - when some of your users just can't access the O365 cloud;
- application compatibility issues - some third-party or in-house developed SharePoint applications cannot be deployed to SharePoint Online (e.g., legacy farm solutions).

## Separate farm for Extranet
One other question that we need to answer - if we stick with the on-premises SharePoint, why not just use the same single farm for Intranet and Extranet? I think that it's an option worth considering, especially for small companies where external access is limited to only few highly trusted entities. But for many larger companies this approach poses a number of challenges:
1) Do we need to publish the internal SharePoint farm to the Internet? Many companies have internal policies which restrict publishing of the internal servers and services to the Internet.
2) If the farm is not published and the corporate users access it via VPN line when roaming, do we grant VPN access to all those external partners and customers as well?
3) Even if we decide to publish the internal SharePoint farm to the Internet, how do we authenticate external users? Do we create logins in the internal corporate AD domain for each customer and partner? Most companies where IT security issues are taken seriously, will never do this. 
4) If can't create user accounts in the internal AD domain for external parties, than we will have to come up with some secondary authentication scheme for external users accessing our internal SharePoint from the Internet. But how do we guarantee that these external users don't get access to some highly sensitive confidential information on our SharePoint, either by malice or mistake?
5) Do we want to make our Intranet farm more complex and fragile by introducing additional authentication schemes for external users, mess up with separate user sources for User Profile Service and People Picker and configure all those additional IIS web sites we might need for this? Most of the time, it is better to keep is simple and only use Windows authentication for the Intranet farm and make life easier for our corporate users, SharePoint developers and farm admins alike.

**In many cases, deploying a new separate SharePoint farm in DMZ network segment is the best answer.**

## Additional costs
The decision to deploy a separate SharePoint farm for Extranet comes with its own price. This statement in [Microsoft article][Use SharePoint Online as a business-to-business (B2B) extranet solution] is quite true: *"...deploying a SharePoint on-premises extranet site involves complex configuration to establish security measures and governance, including granting access inside the corporate firewall, and expensive initial and on-going cost"*. In addition to the SharePoint farm, we will generally have to deploy many other supporting servers and services, such as AD domain controllers, load balancers, SQL servers, reverse proxy and ADFS servers. However, the initial costs associated with the deployment of such supporting infrastructure may be a good investment and will bring huge benefits in future. For example, an isolated AD forest/domain in the DMZ segment may well become the basis and authentication backend for other externally facing IT services beyond SharePoint Extranet portal. 

**Example SharePoint Extranet architecture:**
There are many choices we have to make when we design Extranet solution on SharePoint platform. In this article I will describe the configuration that we successfully deployed and have run for several years, but a good research of available options should be done for each new deployment to explore the alternatives and find the best solution in each case.

### Suggested implementation


|   Subsystem   | Suggested implementation | Comments |
|------------|----------------------|----------------|
| SharePoint farm topology | 2 WFE + 2 APP + OWA | The best option is multi-server deployment with at least two servers for web front end role and two servers for app server role. Please review Microsoft recommended topologies for SharePoint 2016/2019 [here][Technical diagrams for SharePoint Server]. You might also want to deploy Office Web App (OWA) in a single server of farm configuration, but this is beyond the scope of this article. |
| Network segment/IP addressing | All SharePoint and other supporting servers in DMZ segment | SharePoint Extranet farm servers and all other supporting servers should be deployed in the DMZ segment which is separated from the internal IP network by the firewall with "denied by default" policy. Network connectivity must be allowed through the firewall only for certain ports, sources and destinations required by the system's components. |
| Active Directory | Separate isolated AD forest and domain and no inter domain and forest trusts | A new AD forest and domain should be deployed on at least two domain controllers placed in the DMZ segment. These domain controllers should be completely isolated from the internal AD forest/domain and no forest/domain level trusts should be created between the internal and the external domains. SharePoint service and farm admin accounts must be created in this new external AD domain. SharePoint and SQL servers must be members of this domain |
| SQL Backend | AlwaysOn AG | In my opinion, using Availability Groups feature in SQL Server Enterprise Edition is the best way to provide highly-available SQL backend for SharePoint, especially if you spread the DB replicas across two or more datacenters. I also have had some experience with SQL Mirroring available in SQL Server Standard edition, but this technology is not that easy to administer and have a major drawback - **a part** of farm databases may failover to the secondary instance while other databases remain active on primary, in which case the farm goes down and a manual recovery is needed. |
| Web applications and IIS web sites | One HNSC web application and one IIS web site (two IIS sites if using FBA) | The recommendation is to configure SharePoint for Host Named Site Collections (HNSC) and create a single web application with a single IIS web site for the Default zone. In order to make SharePoint Search crawling work, enable BOTH Windows and SAML authentication for the Default zone (more details [here][Problems Crawling the non-Default zone]). |  
| Authentication | SAML/ADFS for internal corporate users + FBA or SAML/ADFS for external users | With SAML authentication enabled on SharePoint and ADFS trusts created between external and internal ADFS servers, it is possible to provide a transparent SingleSignOn authentication experience for internal corporate users. Regarding external users, you have several choices. Among them are using Windows authentication with accounts created in the external AD domain, configuring Form Based Authentication or adding trusts to external identity providers in the local (deployed in DMZ) ADFS servers's configuration. If you choose to use SAML authentication for external users, there is an interesting option to configure ADFS trusts with Facebook or Gmail and allow the external users to use their social accounts to log on to your Extranet SharePoint. |
| Publishing/Reverse proxy | WAP + load balanced VIPs for all services | The easiest way to provide connectivity to SharePoint Extranet for internal and external users is to deploy Microsoft Web Application Proxy service on a couple of Windows Server machines, and make it listen on a load balanced VIP. This WAP service serves as a traditional reverse-proxy and protects the web resources in the DMZ. But it also plays another important role - it securely exposes the ADFS servers across the network boundaries, so that no network connections from external sources are handled by sensitive unprotected ADFS servers. You configure WAP endpoint to publish your Extranet SharePoint URLs to your **external users** in the Internet. You also make your **internal users** access the SharePoint Extranet farm **NOT directly** by connecting to the VIP of the front end servers, **but ONLY via the WAP endpoint**, the same way it goes for external users. This way your load balanced pair of WAP servers is the only https resource in the DMZ visible to the users and systems in the Internet and in the internal network. |
| Claims provider | LDAPCP or UPSBrowser/UPSClaimProvider | One of the issues that you have to solve when you enable SAML or FBA authentication on SharePoint is that the standard People Picker control behaves differently and stops resolving users via lookups to the AD domain controllers (more details [here][Fixing People Picker for SAML Claims Users Using LDAP]). To make People Picker work as expected, you have to deploy a so called Claim Provider, which is a farm solution containing custom code for users lookup and resolution. You can either develop this custom Claim Provider yourself or use one of the free solution available in the Internet. There is a very popular solution [LDAPCP][LDAPCP SharePoint 2013/2016 claims provider for Active Directory and LDAP servers], which you can use if your users authenticating via SAML to SharePoint reside in an LDAP directory. Also, there is a [toolset][UPSClaimProvider] created by the author of this blog consisting of UPSBrowser application and UPSClaimProvider custom claims provider, which allows the User Profile Service to serve as the source of users for People Picker. Please see the next paragraph for more information. |


### Technical diagram

<img
    src="https://imatviyenko.github.io/files/2018-10-26-SharePoint_extranet_architecture/sp_extranet_architecture_ver1.jpg"
    alt="SharePoint Extranet architecture"
    style="width: 100%;" 
/>

### Important flows in the diagram
1) The blue arrows depict the SAML authentication flow for internal corporate users accessing the Extranet SharePoint using their internal AD credentials in a transparent way. Even while they access SharePoint farm deployed in the completely isolate AD forest behind the firewall, their browsers authenticate against their home domain controllers with the regular Windows auth (Kerberos or NTLM) and then the ADFS redirection magic begins. Please consult additional [sources][Configuring SharePoint 2010 and ADFS v2 End to End] if you need more information on ADFS, SAML and federated authentication.  
2) The external users accessing your SharePoint Extranet follow the red arrows in the diagram. The diagram depicts the case when Form Based Authentication (FBA) is enabled for external users and there is a separate AAM URL in a separate zone configured for them. If the external users use SAML authentication, the flow would be similar to the one for internal users with the difference that they will be authenticating against Facebook, Google or other external IDPs.  
3) The green arrows show the ADFS/SAML trust direction. It's **NOT** showing how packets flow one system to another - in fact no network connection is established between ADFS server in the DMZ segment and its counterpart in the internal network when user authentication is in progress. The trusts and the claims flow, which are the terms you will find in many ADFS/SAML guides, are virtual/logical things you have to imagine, representing which system trusts which other system's certificate. The diagram shows the configuration required to allow the internal corporate users to login to the SharePoint Extranet in the DMZ using their internal AD credentials. For this to work, we configure SharePoint in DMZ to trust the security tokens (by registering its certificate as trusted) issued by the DMS ADFS servers, and we make the ADFS servers in DMZ trust the tokens issued by the ADFS servers in the internal network.




## Custom Claims Provider
As mentioned above in this article, a custom Claim Provider is an essential additional component which you have to deploy to SharePoint when you enable SAML or FBA authentication. You can develop this component yourself if are ready for a deep dive into C# programming for SharePoint, but there are a couple of ready-to-use custom components out there which you can deploy to the farm and configure according to your needs. I review the pros and cons of two solutions below. The first solution [LDAPCP][LDAPCP SharePoint 2013/2016 claims provider for Active Directory and LDAP servers] is well known and have been used in many organizations, so it should be the best choice in many scenarios. But sometimes you can't use it because of some technical, organizational or IT Security restrictions, and in this case you can try the second approach with UPSBrowser/UPSClaimProvider tools, which were developed by the author of this blog for a very specific scenario where the common LDAPCP solution could not be used. It's not as mature and well tested as LDAPCP, but nevertheless it's been successfully used in my organization in the SharePoint Extranet scenario.

### LDAPCP
This [Claim Provider][LDAPCP SharePoint 2013/2016 claims provider for Active Directory and LDAP servers] queries the LDAP servers to provide People Picker component with the information it needs to find and resolve users. This solution is highly configurable, and you can specify which LDAP servers to use for queries. However, in some scenarios it is not possible to enable network connectivity between SharePoint servers and the LDAP servers. For example, if your SharePoint servers are deployed in DMZ network segment, you might not be able to use LDAPCP for finding and resolving your internal AD users, because many organizations do not allow connections from DMZ to such highly sensitive systems as internal AD domain controllers.

**Pros:**
- Well known and mature solution;
- Highly flexible and configurable as long as your users data is accessible via LDAP protocol;

**Cons:**
- Requires LDAP ports to be opened between the SharePoint servers and the LDAP servers where users are stored;


### UPSBrowser/UPSClaimProvider
[UPSClaimProvider][UPSClaimProvider] uses the built-in SharePoint User Profile Service (UPS) as the source of users. So it allows People Picker to find and resolve all users which have already been added to the User Profile Service database. You can use many methods to populate the UPS database with users data, such as the standard User Profile Synchronization service. However, you would probably only use this Claim Provider when there are some technical or security restrictions preventing you from using the LDAPCP solution which is much easier to configure, and if there are such restrictions, then you won't be able to use the standard User Profile Synchronization either. 
For a similar scenario, where the access to a DMZ SharePoint Extranet was limited to only a part of the users in the internal Active Directory, I developed an accompanying tool [UPSBrowser][UPSBrowser], which is essentially a web tool for browsing, creating and updating users in the UPS database with the ability to import users from external sources (even through the firewall) via a simple intermediate web service.

This toolset is briefly described below:
- PeoplePicker component uses [UPSClaimProvider][UPSClaimProvider] to find and resolve users, which must be pre-populated in the UPS database; 
- SharePoint administrator can manually browse, create, edit and delete users in the UPS Database using [UPSBrowser][UPSBrowser] web page;
- While it is possible to create a user profile in the UPS database manually, the UPSBrowser tool also offers a simplified way to import users from any external sources in the DMZ segment or in the internal network. The import is performed via the intermediate web service, and by adapting this web service for your needs you can import users from any source. I included one example implementation of this web service, which just like LDAPCP imports data from AD domain controllers via LDAP protocol, but this configuration is much more flexible as it allows the web service to be placed in the internal network and thus allow SharePoint servers in the DMZ to access internal AD domain controllers indirectly without the need to open TCP ports for direct LDAP connectivity. This import procedure is performed by the administrator in a convenient semi-automatic way using the provided GUI in the UPSBrowser tool, **and offers a quick and easy way to populate the UPS database of the Extranet SharePoint farm with the logins and details of the internal corporate users in scenarios when using LDAPCP or importing all internal users directly from AD domain controllers is not an option.**


**Pros:**
- Users to be found and resolved by PeoplePicker can be imported from any backend database or created manually;
- Access to the user information source for import is done via an intermediate web service, which means that this solution can be used when opening LDAP connections from DMZ network to internal AD domain controllers is not possible.

**Cons:**
- Complex to deploy.


## References 
1. [Use SharePoint Online as a business-to-business (B2B) extranet solution][Use SharePoint Online as a business-to-business (B2B) extranet solution]
2. [Technical diagrams for SharePoint Server][Technical diagrams for SharePoint Server]
3. [Configuring SharePoint 2010 and ADFS v2 End to End][Configuring SharePoint 2010 and ADFS v2 End to End] 
4. [Problems Crawling the non-Default zone][Problems Crawling the non-Default zone]
5. [Fixing People Picker for SAML Claims Users Using LDAP][Fixing People Picker for SAML Claims Users Using LDAP]
6. [LDAPCP SharePoint 2013/2016 claims provider for Active Directory and LDAP servers][LDAPCP SharePoint 2013/2016 claims provider for Active Directory and LDAP servers]
7. [UPSClaimProvider][UPSClaimProvider]
8. [UPSBrowser][UPSBrowser]


[Use SharePoint Online as a business-to-business (B2B) extranet solution]: https://docs.microsoft.com/en-us/sharepoint/create-b2b-extranet 

[Technical diagrams for SharePoint Server]: https://docs.microsoft.com/en-us/sharepoint/technical-reference/technical-diagrams

[Configuring SharePoint 2010 and ADFS v2 End to End]: https://samlman.wordpress.com/2015/02/28/configuring-sharepoint-2010-and-adfs-v2-end-to-end/ 

[Problems Crawling the non-Default zone]: https://blogs.msdn.microsoft.com/sharepoint_strategery/2014/07/08/problems-crawling-the-non-default-zone-explained/

[Fixing People Picker for SAML Claims Users Using LDAP]: https://blogs.msdn.microsoft.com/kaevans/2013/05/26/fixing-people-picker-for-saml-claims-users-using-ldap/

[LDAPCP SharePoint 2013/2016 claims provider for Active Directory and LDAP servers]: https://ldapcp.com/

[UPSClaimProvider]: https://github.com/imatviyenko/UPSClaimProvider

[UPSBrowser]: https://github.com/imatviyenko/UPSBrowser
