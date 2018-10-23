---
layout: post
title: "Active Directory Kerberos authentication for Apache web server - follow up"
description: "Some additional notes related to the post 'Active Directory Kerberos authentication for Apache web server'"
date: 2018-10-22
tags: 
    - AD
    - authentication
    - Apache
    - Linux
---

In this post I want to share some additional tips and tricks that came out of a very productive email discussion I had with one reader of the [original post][Active Directory Kerberos authentication for Apache web server]. The notes in this post are somewhat ad-hoc and should only make sense in the context of the original article.


## User/service principal names, 'princ' argument for ktpass and doing it Windows way
I have seen some posts in the Internet where people passed something like HTTP/hostname.domain.com@DOMAIN.COM as the "princ" argument 
to ktpass command. One example of this approach is [this resource][Create a Kerberos service principal name and keytab file]. As you can see they are passing some pre-existing real AD user login name as the value for the "mapUser" parameter, and as far as I understand with this approach the ktpass command will **overwrite the "userPrincipalName" property of the AD user account object with something like "HTTP/hostname.domain.com@DOMAIN.COM"**.

This is mentioned in this great [resource][All you need to know about Keytab files]:
> Note that the userPrincipalName has been updated with the value you provided in the parameter /princ.

I don't want ktpass to to update the **userPrincipalName** attribute of my service account object in AD with some strange values. I much prefer the **userPrincipalName** property to remain as it was when the account was created, e.q. "username@domain.com". That's why I add "-SetUPN" option to ktpass command which according to the same resource activates a different behavior:
> In that case, the UPN will not be touched and the value specified with /princ will only be added to the servicePrincipalName (SPN).

This behavior is more in line with what you'd expect if you configure SPNs for Windows web server with Kerberos authentication. In this scenario **"userPrincipalName"** attribute of the service account in AD is never modified and all HTTP end points linked to this service account are added as **"servicePrincipalName"** multivalued attributes.

## SPN format for HTTPS/SSL/TLS web sites
If you use either http or https with SSL/TLS encryption, the format of the SPN records stays the same - it's HTTP/****. The "HTTP" part is not replaced with "HTTPS" when you switch over to SSL.

## CNAME/DNS alias scenario
For our production Intranet web server on Linux, we also used DNS CNAME record to point the users to the correct server IP.
So I configured DNS records and keytab like this:

**Public URL which the users will use to access the web site**
http://intra.company.com

**DNS records**
server.company.com A 192.168.1.100 (real Linux server)  
intra.company.com CNAME server.company.com (DNS alias)


**AD user account to be used as the web site service account**  
intra_app


**Create SPNs  - execute on a domain controller**  
setspn -s HTTP/intra.company.com intra_app
setspn -s HTTP/server.company.com intra_app

*Note: This will add two new "servicePrincipalName" attributes in the properties of the "intra_app" AD user object - one for the A record and another one for the CNAME hostname. I do it because some browsers might perform DNS name canonization before starting to query for the SPN.  No matter which hostname (A record or CNAME record) the browser uses to perform the SPN query for, it will get the Kerberos service ticket for the same correct AD user account intra_app. However, so far I see that in fact canonization do occur and Windows clients are requesting Kerberos service tickets for the A record (i.e., the real host name directly linked to IP, not for CNAME alias)*


**Generate AES256 key - execute on a domain controller**  
ktpass /out intra_app.keytab /princ intra_app@company.com -SetUPN /mapuser intra_app /crypto AES256-SHA1 /ptype KRB5_NT_PRINCIPAL /pass XXX -SetPass +DumpSalt /target dc1.company.com  

*Note: Generate AES256 key and output it to the command prompt, so that you can mark and copy it to notepad. The keytab file created by this command will not be used and can be discarded. Also copy "vno" number to the notepad. EXCLUDE THE '0x' PREFIX WHEN YOU SAVE THE GENERATED KEY VALUE IN THE NOTEPAD!*


**Generate RC4 key - execute on a domain controller**  
ktpass /out intra_app.keytab /princ intra_app@company.com -SetUPN /mapuser intra_app /crypto RC4-HMAC-NT /ptype KRB5_NT_PRINCIPAL /pass XXX -SetPass +DumpSalt /target dc1.company.com

*Note: Generate RC4 key and output it to the command prompt, so that you can mark and copy it to notepad. The keytab file created by this command will not be used and can be discarded. Also copy "vno" number to the notepad. EXCLUDE THE '0x' PREFIX WHEN YOU SAVE THE GENERATED KEY VALUE IN THE NOTEPAD!*


**Modify apache config file - on the Linux box**  
Add the mod_auth_kerb settings as is described in the blog post.

*Note: the file name specified in the Krb5Keytab parameter should be pointing to the keytab file that we will create in the next step*


**Create a new empty keytab file using the Linux ktutil command and add two entries - on the Linux box**  
addent -key -p HTTP/server.company.com@company.com -k 2 -e aes256-cts

*Note: Enter the saved AES key from the notepad, and remember that the number after -k switch in the "addent" command must be equal to the saved "vno" in the notepad.*

addent -key -p HTTP/server.company.com@company.com -k 2 -e RC4-HMAC  

*Note:  Enter the saved RC4 key from the notepad, and remember that the number after -k switch in the "addent" command must be equal to the saved "vno" in the notepad.*

*Also note that we are creating the key entries for the server own hostname (A record). I found that in my case I can only make it work if I create the entries here for the server's own FQDN configured in the local /etc/hosts file.*

*My /etc/hosts files contains records like these:*

*127.0.0.1       localhost*  
*192.168.1.100     server.company.com server*  

*So, on Linux server "server.company.com" I have to add keytab entries for "HTTP/server.company.com@company.com".*

list  
wkt /etc/apache2/<some_name>.keytab  

*Note: the file name should be the same as specified for the "Krb5Keytab" param in the apache conf file.*

q

*Note: this will quit the interactive mode of ktutil and return you to the shell prompt.*

**Change file system permission on the newly created keytab file to grant read access to the apache process - on the Linux box**  
chown root:www-data /etc/apache2/<some_name>.keytab  
chmod 0640 /etc/apache2/<some_name>.keytab  
systemctl restart apache2

## Logs and troubleshooting
1) Apache error log is one of the most useful resource for troubleshooting. There may be some way to enable verbose logs for the Kerberos client on the Linux box, but I didn't try this.

2) Also, I heavily used Wireshark running on my own Windows PC during the configuration. I inspected Kerberos traffic between my PC and the domain controller to see different things like:
- for which destination hostname my PC is querying SPNs;
- for which service principal name (i.e. which service user name) my PC is requesting and obtaining Kerberos service tickets.

3) You can see which tickets for which hostnames your test PC has acquired by running "klist" in a regular (non-admin!) Windows command prompt.

4) For reviewing the AD object properties I would highly recommend this free tool: https://www.ldapadministrator.com/softerra-ldap-browser.htm  
I use it after I run setspn command to verify the values of the "servicePrincialName" attribute for my service account user object.

5) As soon as you got all DNS and SPN records configured properly, you can confirm by klist and Wireshark inspection that your PC is obtaining correct Kerberos service tickets for correct hostnames from domain controllers. After that the remaining problems should be related to the Linux part, and that's where the apache error logs are getting very useful for troubleshooting.


## References 
1. [Active Directory Kerberos authentication for Apache web server][Active Directory Kerberos authentication for Apache web server]
2. [Create a Kerberos service principal name and keytab file][Create a Kerberos service principal name and keytab file]
3. [All you need to know about Keytab files][All you need to know about Keytab files]

[Active Directory Kerberos authentication for Apache web server]: https://imatviyenko.github.io/blog/2018/09/11/Apache-AD-kerberos 

[Create a Kerberos service principal name and keytab file]: https://www.ibm.com/support/knowledgecenter/en/SSAW57_8.5.5/com.ibm.websphere.nd.multiplatform.doc/ae/tsec_kerb_create_spn.html

[All you need to know about Keytab files]: https://blogs.technet.microsoft.com/pie/2018/01/03/all-you-need-to-know-about-keytab-files


