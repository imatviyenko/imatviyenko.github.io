---
layout: post
title: "Active Directory Kerberos authentication for Apache web server"
description: "Configuration of AD Kerberos authentication for Apache web site running on a Linux host - step by step guide with some background information"
date: 2018-09-11
tags: 
    - AD
    - authentication
    - Apache
    - Linux
---

I was recently involved in configuration of Kerberos authentication for a newly deployed Apache web site, using [mod_auth_kerb][Mod_auth_kerb official page] module. We have an Active Directory environment with the largest part of our users working on Windows 7+ computers, but the Apache web site was supposed to be running on a Linux host. In the end, I was able to configure an Apache web site hosted on a Linux box to use Kerberos to transparently authenticate AD users connecting from Windows computers (IE and Chrome browsers). I also enabled support for both RC4 and AES256 Kerberos encryption methods, and this was the trickiest part of all. While we still support the older RC4 encryption for Kerberos in our environment because we have some legacy systems not capable of using AES encryption, it is a very good idea to ensure that all new installations support AES by default as its more secure. You can find the following statement in the official IETF [description][The RC4-HMAC Kerberos Encryption Types Used by Microsoft Windows] of the Microsoft implementation of the RC4 encryption for Kerberos:  
__*It is thus   RECOMMENDED not to use the RC4 encryption types defined in this   document if alternative stronger encryption types, such as    aes256-cts-hmac-sha1-96 [RFC3962], are available.*__

## Overview of configuration steps
There are many Internet resources describing the configuration process for Apache + mod_auth_kerb + AD - just google for "Apache mod_auth_kerb AD" and you get tons of step-by-step instructions and guides. But in the end, I had to gather pieces of information from many different source, because there was not one guide which explained all Kerberos machinery, options and required configuration actions in sufficient details in one place. The most useful and comprehensive resources that I was able to find are listed in the end of this document.

The tested configuration process looks like that:
1. Prepare and write down the crucial configuration parameters which you will use in the next steps:
    - Apache web site FQDN (more details on this below)
    - Active Directory account to be used as the service account (service identity) for this web site
    - FQDNs of at least two AD domain controllers that you are going to use as the KDCs for the Linux box
    - The DNS name of you AD domain
2. Prepare Linux box:
    - Install **krb5-user** package - binaries required to configure the Linux server as a "service" per Kerberos protocol terminology
    - Install **Apache 2**
    - Install **mod_auth_kerb** Apache module
3. Run Windows tool **ktpass** on AD domain controller to generate and output to the console two secret keys (for AES256 and RC4 encryption methods, respectively) associated with the service account specially created in the AD to be used as the identity of the web server. At this step I deviate from most of the instructions and manuals published in the Internet, as I describe in the next section.
4. Run Windows tool **setspn** on AD domain controller to configure proper SPN attribute (Kerberos Service Principal Name) in the properties of the service account AD object.
5. Create a DNS record for the public FQDN of the web server (the one entered in the browser address bar).
6. On the Linux box, use **ktutil** tool (part of the **krb5-user** package) to create a new empty keytab file and then use its **addent** subcommand to add two entries for AES256 and RC4 encryption schemes using the secret keys output to the console by **ktpass** tool at step 3.
7. Configure host-wide Kerberos parameters for the **krb5-user** package by editing configuration file **/etc/krb5.conf**
8. Configure **mod_auth_kerb** Apache module parameters by editing the Apache web site configuration file (i.e., **/ect/apache2/sites-enabled/000-default.conf**) and adding Kerberos specific entries under the **VirtualHost** section.
9. Optionally enable the fall-back mechanism available in the **mod_auth_kerb** which allows clients which do not support Kerberos to use Basic HTTP authentication scheme instead (**SECURITY WARNING! Do not enable this option unless you also enable AND force SSL/TLS for your web site.**)
9. Make sure that the browsers on Windows client computers are configured to start Kerberos authentication with your web site automatically (check Internet Explorer "Local intranet" zone settings).

## Example configuration step-by-step
The example below has been tested on a freshly installed Ubuntu system in a mixed Windows 2008R2/2012R2 AD domain controllers environments.
Change the URLs, accounts and domain names to match your own settings and it should probably work for you as well.   
As we remember, from the Kerberos protocol point of view, there are three computers participating in the authentication process: KDC (domain controller), service (Apache web server) and client (Windows computer accessing the web site). In the table below, the second column specifies the computer where the configuration commands and actions for the current step must be executed. The third columns specifies whether this step is a command that you need to execute or some other type of action that you must carry out. Run all commands exactly as given in the table in Windows CMD or Linux shell prompt. I executed all Linux commands with root privileges (**sudo su**), but you can prefix each command with **sudo** if you like. **Many of the command parameters and settings in the configuration files are case sensitive, so it is very important to use exactly the same case for each step as is used in the example below!**

Our example is for the following fictitious configuration:
- AD domain DNS name: **company.com**
- AD domain controller 1: **dc1.company.com**
- AD domain controller 2: **dc2.company.com**
- Web server FQDN: **webserver1.company.com** 
- Web site URL: **http://webserver1.company.com** [^1]
- AD service account name (userPrincipalName): **srvaccount1@company.com**
- AD service account password[^2]: **THEpassword**


{::options parse_block_html="true" /}
<div  class="smalltext">

| ## | Computer | Type | Command/action                                                         | Comments                                    |
|----|----------|------|------------------------------------------------------------------|---------------------------------------------|
| 1 | DC | Command | setspn -s HTTP/webserver1.company.com srvaccount1 | Register SPN for the web site in the properties of the account **srvaccount1** |
| 2 | DC | Command | ktpass /out temp.keytab /princ srvaccount1@COMPANY.COM -SetUPN /mapuser srvaccount1 /crypto AES256-SHA1 /ptype KRB5_NT_PRINCIPAL /pass THEpassword -SetPass +DumpSalt /target dc1.company.com| Generate and output secret key and key serial number (vno) for AES algorithm|
| 3 | DC | Action | Copy the **key** and **vno** values for AES algorithm from the output of the previous commands to a temporary text file| We only need the values written to screen by the previous **ktpass** command, the generated file temp.keytab won't be used|
| 4 | DC | Command |ktpass /out temp.keytab /princ srvaccount1@COMPANY.COM -SetUPN /mapuser srvaccount1 /crypto RC4-HMAC-NT /ptype KRB5_NT_PRINCIPAL /pass THEpassword -SetPass +DumpSalt /target dc1.company.com| Generate and output secret key and key serial number (vno) for RC4 algorithm |
| 5 | DC |Action | Copy the key and vno values for RC4 algorithm from the output of the previous commands to a temporary text file| We only need the values written to screen by the previous **ktpass** command, the generated file temp.keytab won't be used|
| 6 | DC |Action | Create **A** type DNS record **webserver1.company.com**, pointing directly to the IP address of the Linux web server | There are additional issues to be aware of if you use CNAME DNS aliases, as is explained later in this document [^1] |
| 7 | Linux server | Action | Install apache2 | Install apache2 if it's not already running on your Linux box|
| 8 | Linux server | Command | apt install libapache2-mod-auth-kerb | Install mod_auth_kerb Kerberos authentication module for Apache |
| 9 | Linux server | Command | apt install krb5-user |Install **krb5-user** package - kerberos 5 client library |
| 10 | Linux server | Command | nano /etc/krb5.conf | Open Kerberos 5 system-wide configuration file **/etc/krb5.conf** for editing |
| 11 | Linux server | Action | Replace the text in **/etc/krb5.conf** file with the contents of the **krb5.conf** block below | Configure system-wide Kerberos settings|
| 12 | Linux server | Action | **Ctrl + O**, **Ctrl + X** | Save file **/etc/krb5.conf** and exit nano |
| 13 | Linux server | Command | nano /etc/apache2/sites-enabled/000-default.conf | Edit the file **/etc/apache2/sites-enabled/000-default.conf** - definition of the default apache2 web site |
| 14 | Linux server | Action | Add **\<Location\>** block inside the **\<VirtualHost\>** section - use the contents of the block **Location** | Configure **mod_auth_kerb** settings [^3] |
| 15 | Linux server | Action | **Ctrl + O**, **Ctrl + X** | Save file **/etc/apache2/sites-enabled/000-default.conf** and exit nano |
| 16 | Linux server | Command | ktutil | Start **ktutil** tool (part of the **krb5-user** package) which is used to manipulate keytab files |
| 17 | Linux server | Command | list | Run **list** subcommand to verify that you have no entries in the currently open empty new keytab file |
| 18 | Linux server | Command | addent -key -p HTTP/webserver1.company.com@COMPANY.COM -k **n** -e aes256-cts| Run **addent** subcommand to add AES key for the HTTP SPN associated with the web server FQDN [^1]. The **n** parameter must be substituted with the **vno** value that you captured from the output of the **ktpass** command in step 3. The command will ask you to enter the key in hexadecimal format.
| 19 | Linux server | Action | Enter the AES256 **key** value from the output of the **ktpass** command in step 3 | Enter 32 hexadecimal digits - the generated Kerberos secret key for AES256 encryption method derived from the current password and account name of the service account **srvaccount1** |
| 20 | Linux server | Command | addent -key -p HTTP/webserver1.company.com@COMPANY.COM -k **n** -e RC4-HMAC| Run **addent** subcommand to add RC4 key for the HTTP SPN associated with the web server FQDN [^1]. The **n** parameter must be substituted with the **vno** value that you captured from the output of the **ktpass** command in step 5. The command will ask you to enter the key in hexadecimal format.
| 21 | Linux server | Action | Enter the RC4 **key** value from the output of the **ktpass** command in step 5 | Enter 16 hexadecimal digits - the generated Kerberos secret key for RC4 encryption method derived from the current password and account name of the service account **srvaccount1** |
| 22 | Linux server | Command | list | Run **list** subcommand to verify that now you have two entries for **HTTP/webserver1.company.com@COMPANY.COM** in the currently open new keytab file  - one for AES256 key and the other one for RC4 |
| 23 | Linux server | Command | wkt /etc/apache2/kerb.keytab | Save the open keytab file with the two entries as **/etc/apache2/kerb.keytab** |
| 24 | Linux server | Command | q | Quit the **ktpass** interactive prompt and return to the shell |
| 25 | Linux server | Command | chown root:www-data /etc/apache2/kerb.keytab | Change the owner and group for the newly generated **kerb.keytab** file to grant access to this file for the apache process |
| 26 | Linux server | Command | chmod 0640 /etc/apache2/kerb.keytab | Set the file system permissions for the newly generated **kerb.keytab** |
| 27 | Linux server | Command | systemctl restart apache2 | Restart apache to apply the configuration changes | 
| 28 | Client computer | Action| **Internet Explorer settings** -> **Security** -> select **Local intranet**, click **Sites** -> **Advanced**, , add the web site URL **http://webserver1.company.com** | Internet Explorer (and Chrome) will only try to authenticate automatically via Kerberos protocol with the sites in the **Local intranet** zone |

</div>



### krb5.conf
```
[libdefaults]
    default_realm = COMPANY.COM
    default_tkt_enctypes = aes256-cts-hmac-sha1-96 rc4-hmac
    default_tgs_enctypes = aes256-cts-hmac-sha1-96  rc4-hmac
    permitted_enctypes = aes256-cts-hmac-sha1-96 rc4-hmac


[realms]
    COMPANY.COM = {
            kdc = dc1.company.com
            kdc = dc2.company.com
            admin_server = dc1.company.com
    }

[domain_realm]
    .company.com = COMPANY.COM
    company.com = COMPANY.COM

```

### \<Location\>
```
<Location />
  AuthType Kerberos
  AuthName "Active Directory"
  KrbAuthRealms COMPANY.COM
  KrbServiceName HTTP
  Krb5Keytab /etc/apache2/kerb.keytab
  KrbMethodNegotiate On
  KrbMethodK5Passwd Off
  require valid-user
</Location>
```

## Testing the results
If you correctly configured all components, you should be able to launch Internet Explorer, Edge or Chrome browser on a domain joined Windows machine and open **http://webserver1.company.com** without any password prompt. You can also verify if a Kerberos ticket was issued by the KDC (domain controller) to your client system to be used for authentication against the web server. Type the following command in the CMD prompt:
```
klist
```

You should get the list of Kerberos tickets for various services that your system is talking to. Find the ticket for **http://webserver1.company.com** and check the **KerbTicket Encryption Type** and **Session Key Type** fields. These two fields specify the encryption method that the KDC used for encrypting the issued service ticket and the requested encryption method that will be used for the session key between the client and the server, respectively [^4]. 

## Background information

### How Kerberos authentication works
First of all, you have to define how your web site will be accessed by the clients from the DNS and IP address assignment point of view. There can be many possible configurations, and we have to know exactly how DNS records are configured and how IP connections from the clients to the web server are routed. The main thing to remember is that Kerberos clients (web browsers on Windows clients) use DNS lookups and special Kerberos protocol functionality to find out which AD account is the identity of the web server they are connecting to. In other words, they need to know the service account name for the web server in order to generate a valid Kerberos service ticket which can only be used by this particular account. The whole process looks like that:
- A user launches IE or Chrome browser on his Windows PC joined to the AD domain while being connected to the corporate network;
- He enters **http://webserver.company.com** in the browser address bar and presses Enter;
- The browser issues a DNS lookup for the hostname **webserver.company.com**;
- If the DNS server returns an A record with the IP address for **webserver.company.com**, it is considered to be the **canonical DNS name** of the web server. However if DNS replies with a CNAME (DNS alias) record for **webserver.company.com** and then resolves the alias to an A record with a different hostname __*anotherhost.company.com*__, then this second hostname will be considered to be the **canonical DNS name** of the web server, and all SPN lookups will be issued for that __*anotherhost.company.com*__ FQDN and NOT for the original FQDN **webserver.company.com** entered in the address bar! This issue with CNAME aliases is often a major source of confusion, so it might be a good idea to look through a couple resources [here][Kerberos configuration known issues] and [here][The Chromium Projects - HTTP authentication] to get the idea of the whole process. The popular approach is to only create A type DNS records for your web sites with Kerberos authentication, so that the FQDN in the address bar is directly resolved to the actual IP address of the web server or to the VIP of the load balancer if one is used.
- The browser constructs the so called SPN (Service Principal Name) the **canonical DNS name** it determined in the previous step. The SPNs a strings used to server as identifiers of resources and services in the network and are actually just string values in the format like that:
    - HTTP/webserver.company.com (this form of SPN identifier is commonly used for web servers)
    - HOST/server1.company.com (SPN for some generic service running on computer server1.company.com)
    - MSSQLSvc/sqlserver.company.com:1433 (SPN for MS SQL Server instance running on machine sqlserver.company.com on TCP port 1433)
- The browser issues a special Kerberos request (KRB_TGS_REQ) to its KDC specifying the SPN string as a parameter. The KDC looks through its accounts database searching for an account associated with the requested SPN. In our case KDC is an AD domain controller and it searches for a user/computer account having an LDAP attribute named **servicePrincipalName** with the value equal to the SPN string in the request. In corporate AD environment, the common practice is to create a separate "service" account for each distinct service/URL/web site, which is technically a regular AD user account often configured with a more relaxed password expiration policy.
- The KDC should find one and only one matching account for each SPN. In this ideal case it generates a so called Kerberos service ticket and returns it to the calling browser application in KRB_TGS_REP response. If KDC finds more than one matching account for one given SPN, it is a considered an [error state][Why you can still have duplicate SPNs in AD 2012 R2 and AD 2016] and Kerberos authentication in this case won't work. The same goes for the case if KDC cannot find any matching account at all.
- The Kerberos service ticket generated by the KDC contains in its encrypted payload some key data which can only be decrypted using the password for the account associated with the SPN. **This ticket cannot be used with any other account!**
- The browser sends the service ticket to the web server.
- The web process on the web server tries to decrypt the ticket it received from the browser. If we are taking about Apache running on Linux with **mod_auth_kerb**, it looks for a special **keytab** file which must be supplied by the administrator. This **keytab** file is essentially a small database, matching SPN strings to secret keys to be used for encryption/decryption. Its structure is like that:

| SPN                         | Algorithm | Secret key                                                        |
|-----------------------------|-----------|-------------------------------------------------------------------|
|HTTP/webserver.company.com   | AES256    | f1a015ea515c317737aa78f3e19eec41b7e3b0f723bfde2d973069e8296f3e9f9 |
|HTTP/webserver.company.com   | RC4       | 9472a6e31280efe2acdac7d51398fa89                                  |

- As you can see, the **keytab** file in our example contains two entries for the __*same*__ SPN, but for two different ciphers - AES256 and RC4. And, of course, we have a separate keys of different length for each cipher. Not incidentally, our real **keytab** file that we are going to create will look just like that and will have two keys for two algorithms for the same one SPN of our web server. We configure it this way because in many networks both ciphers (RC4 and AES) are used simultaneously. In my organization, my own computer sometimes gets RC4 service tickets from the KDC even while the AES cipher is the preferred algorithm for recent Windows versions [^4]. Is these conditions some clients may obtain RC4 service tickets for authentication to our web server, while others may get and send to the web server AES encrypted tickets, and our web server must be able to decrypt both types.
- If the web server is able to find the matching entry in the  **keytab** file [^1], it extracts from it the secret key and uses it to decrypt and verify the service ticket received from the browser. If the decryption and verification is successful then the client is deemed to be authenticated.

### Different scenarios for DNS configuration and hosting multiple sites on one web server 
In the table below, I outlined two possible scenarios and my recommendations for DNS and keytab configuration.

{::options parse_block_html="true" /}
<div  class="smalltext">

| ## | Scenario | DNS configuration | Service account/keytab configuration | Comments |
|----|----------|-------------------|--------------------------------------|----------|
| 1  | One web site with public URL the same as the web server FQDN | host FQDN A record pointing to the server's IP address | One service account, keytab contains two entries for the server's FQDN with two keys for AES and RC4 | This simplest scenario is used in the example section | 
| 2  | Two or more web sites, their public URLs are different from the server's FQDN, same service account | Server A record: **srv1.company.com**; site1 CNAME: **site1.company.com** -> **srv1.company.com**; site2 CNAME: **site2.company.com** -> **srv1.company.com**;  | One service account for both sites, keytab contains two entries for the server's FQDN **HTTP/srv1.company.com@COMPANY.COM** with two keys for AES and RC4 for the same service account | Web browser use CNAME canonization for both sites and request Kerberos service ticket for the server's FQDN **srv1.company.com** from the KDC | 


</div>


## References 
1. [Mod_auth_kerb official page][Mod_auth_kerb official page]
2. [The RC4-HMAC Kerberos Encryption Types Used by Microsoft Windows][The RC4-HMAC Kerberos Encryption Types Used by Microsoft Windows]
3. [How-to – Single sign on with Active directory and Apache][How-to – Single sign on with Active directory and Apache]
4. [How the Kerberos Version 5 Authentication Protocol Works][How the Kerberos Version 5 Authentication Protocol Works]
5. [Kerberos Keytabs – Explained][Kerberos Keytabs – Explained]
6. [All you need to know about Keytab files][All you need to know about Keytab files]
7. [Encryption Type Selection in Kerberos Exchanges][Encryption Type Selection in Kerberos Exchanges]
8. [ktutil - problems generating AES keys (salt?)][ktutil - problems generating AES keys (salt?)]
9. [Apache sends wrong server principal name to Kerberos][Apache sends wrong server principal name to Kerberos]
10. [Kerberos configuration known issues][Kerberos configuration known issues]
11. [The Chromium Projects - HTTP authentication][The Chromium Projects - HTTP authentication]
12. [Why you can still have duplicate SPNs in AD 2012 R2 and AD 2016][Why you can still have duplicate SPNs in AD 2012 R2 and AD 2016]

[Mod_auth_kerb official page]: http://modauthkerb.sourceforge.net 

[The RC4-HMAC Kerberos Encryption Types Used by Microsoft Windows]: https://tools.ietf.org/html/rfc4757 

[How-to – Single sign on with Active directory and Apache]: https://smeretech.com/single-sign-active-directory-apache

[How the Kerberos Version 5 Authentication Protocol Works]: https://technet.microsoft.com/pt-pt/library/cc772815(v=ws.10)aspx#w2k3tr_kerb_how_pzvx

[Kerberos Keytabs – Explained]: https://social.technet.microsoft.com/wiki/contents/articles/36470.kerberos-keytabs-explained.aspx

[All you need to know about Keytab files]: https://blogs.technet.microsoft.com/pie/2018/01/03/all-you-need-to-know-about-keytab-files

[Encryption Type Selection in Kerberos Exchanges]: https://blogs.msdn.microsoft.com/openspecification/2010/11/17/encryption-type-selection-in-kerberos-exchanges

[ktutil - problems generating AES keys (salt?)]: http://kerberos.996246.n3.nabble.com/ktutil-problems-generating-AES-keys-salt-td41104.html

[Apache sends wrong server principal name to Kerberos]: https://stackoverflow.com/questions/14687245/apache-sends-wrong-server-principal-name-to-kerberos

[Kerberos configuration known issues]: https://docs.microsoft.com/en-us/previous-versions/office/sharepoint-server-2010/gg502606(v=office.14)

[The Chromium Projects - HTTP authentication]: https://www.chromium.org/developers/design-documents/http-authentication

[Why you can still have duplicate SPNs in AD 2012 R2 and AD 2016]: https://blogs.technet.microsoft.com/389thoughts/2017/02/08/why-you-can-still-have-duplicate-spns-in-ad-2012-r2-and-ad-2016

## Notes
[^1]: It seems that the Kerberos auth module and the krb5user library are only looking for those entries in the keytab file where the hostname matches the value configured in the "/etc/hosts" file. There many possible scenarios for DNS configuration and the only one of them which is straightforward to configure is when the public FQDN for your web site matches the FQDN of your server associated with the server's own IP address in the **/etc/hosts** file. The example configuration in this document describes just that scenario. However, there are other ways to configure DNS and Kerberos keytab file for your web server. Please refer to the **Different scenarios for DNS configuration and hosting multiple sites on one web server** section of this document for more information on this subject. Please note, that there should be a way to force **mod_auth_kerb** and the client library **krb5-user** to accept any entries in the keytab file and not only those matching the FQDN hostname configured in the **/etc/hosts** file. To do this, you can add **KrbServiceName Any** option to the **/<Location/>** section of the apache config file, according to this [resource][Apache sends wrong server principal name to Kerberos]. However, I had no luck with this option, no matter how I tried. But still, perhaps it could've been just the wrong version of binaries that I used and this option would still work with your setup.

[^2]: There is also an option to change the password to a random value and generate keytab secret keys for this newly changed password value. In this case DO NOT use '-SetPass' option and add the '+rndPass' option insted. See [this resource][All you need to know about Keytab files] for more information.

[^3]: The **mod_auth_kerb** option **KrbMethodK5Passwd On** will enable the fallback to HTTP Basic authentication. However, as the password is not encrypted when this authentication scheme is used, this option __*must not*__ be enabled if your site is accessible via plain HTTP protocol with no SSL/TLS encryption. This option, if used with mandatory SSL/TLS encryption of HTTPS traffic between the browser and the server, **is the only way to authenticate clients and browser which do not support Kerberos authentication.** In that respect, HTTP Basic authentication plays a role similar to that of NTLM in Microsoft Windows/IIS environment, which also serves as the fallback authentication mechanism when Kerberos cannot be used. The examples of clients, which do not support Kerberos and will require **KrbMethodK5Passwd On** option are mobile clients (smartphones, tablets and etc.), Linux/Windows/Mac computers not joined to the AD domain/Kerberos realm and corporate domain joined Windows machines connecting to the web server over the Internet via the public IP (out of office scenario), in which case the domain controllers and the KDC service are behind the firewall and cannot be contacted.

[^4]: This [document][Encryption Type Selection in Kerberos Exchanges] describes how different encryption types are selected for different Kerberos messages and exchanges. The rules for encryption types are very complex and vary from one Windows version to another. However, there is a small hack, which can force your Windows client (tested on Windows 10) to use only the strongest available algorithm AES256 for all Kerberos exchanges. Launch regedit and add a new DWORD value **DefaultEncryptionType** under **HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters**, set it to 18 (decimal) or 0x12 (hexadecimal), which will enforce AES256 encryption for Kerberos pre-authentication and make KDC use AES256 when it will be issuing service tickets. You'll need to reboot to apply this settings.