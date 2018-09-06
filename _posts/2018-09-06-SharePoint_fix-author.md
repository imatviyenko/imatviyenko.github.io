---
layout: post
title: "SharePoint script - fix-authors.ps1"
description: "PowerShell script for SharePoint to fix authors of files uploaded to a document library to solve the 'User cannot be found' error"
date: 2018-09-06
tags: SharePoint PowerShell scripts administration
---

## Problem description
Sometimes you may encounter a cryptic exception like the one below (taken from the real ULS logs) when your applications tries to read files stored in a document library:
```
Exception occured in scope Microsoft.SharePoint.SPFile.get_Author. Exception=Microsoft.SharePoint.SPException: User cannot be
found. at Microsoft.SharePoint.SPUserCollection.get_Item(String loginName)     at Microsoft.SharePoint.SPFile.get_Author() at
Microsoft.SharePoint.ServerStub.SPFileServerStub.GetProperty(Object target, String propName, ProxyContext proxyContext) at
Microsoft.SharePoint.Client.ServerStub.GetPropertyWithMonitoredScope(Object target, String propertyName, ProxyContext proxyContext)
```
In our specific case, we got this error when a SharePoint add-in custom code executed CSOM call against SharePoint to retrieve the list of attachment for a list item (electronic form), all stored in one folder in a document library. The error only surfaced for those files uploaded by users who since they did it has had their Active Directory login (samAccountName) changed. Those users were all ladies, changing their family name to that of the husband. Now if anyone, including those same ladies authenticating under their new logins, tried to open the saved form, the following C# code was executed to read the properties of the associated attachments and failed with the above mentioned exception:
```csharp
private void BindAttachments()
{
    using (var clientContext = TokenHelper.GetS2SClientContextWithWindowsIdentity(new Uri(LibraryUrl), Request.LogonUserIdentity))
    {
        try
        {
            scl.Folder folder = clientContext.Web.GetFolderByServerRelativeUrl(Attachments + "/" + Request["ID"].ToString());
            clientContext.Load(folder.Files);
            clientContext.ExecuteQuery();
            foreach (scl.File fi in folder.Files)
            {
                clientContext.Load(fi.Author);
                clientContext.Load(fi.ModifiedBy);
            }
            clientContext.ExecuteQuery();
            repAttachments.DataSource = folder.Files;
            repAttachments.DataBind();

        }
        catch (Exception ex)
        {
            if (ex.Message != "File Not Found.")
                ShowErrorMessage(ex.Message);
        }
    }
}

```

As it turned out, the reason for that error is that when a file is uploaded to a document library, the current login in claims format (i.e., **'i:0#.w|DOMAIN\samAccountName'**) as saved as string value property **'vti_author'** in the Properties array of SPFile.Item object. In other words:
- a user with the AD login **'COMPANY\user1'** uploads a file to doc lib;
- later you read the file as an SPFile object;
- the value of the expression **SPFile.Item.Properties["vti_author"]** will be equal to **"i:0#.w|COMPANY\user1"**.
When Microsoft.SharePoint.SPFile.get_Author() method is trying to get the author of the file, it reads the login as string from this property and looks for a user with the exact login among the users of the SPWeb where the doc lib is hosted. If it can't find such an user, then it throws the error 'User cannot be found', which is what happens if the user's login has changed and the old value stored in the **'vti_author'** property is no longer valid. Besides, the same story goes for "Modified By" file attribute, which is internally stored as a **'vti_modifiedby'** property.

If you see the error like that when trying to access files in a doc lib, you can quickly run the following script to check the authors logins:

```PowerShell
Add-PSSnapin Microsoft.SharePoint.PowerShell;
$spWeb = Get-SPWeb https://sp.company.com/sites/...;

$spFolder = $spWeb.GetFolder("DocLib/Folder1");
$files = $spFolder.Files;
$files.count;

foreach ($spfile in $files) {
    $spFileItem = $spfile.Item;
    $author = $spFileItem.Properties["vti_author"];
    $modifiedby = $spFileItem.Properties["vti_modifiedby"];

    "$($spfile.Name) author: $author modifiedby: $modifiedby" ;

    $spUser = $spWeb.SiteUsers[$author2]; # Note that there WON'T be a record in this array for invalid logins
    "spUser: $spUser";
};
```
## Solution
In order to solve this problem, I developed a PowerShell script **[fix-author](https://github.com/imatviyenko/Scripts-SharePoint)**, inspired by this [solution](https://blogs.msdn.microsoft.com/valdon/2011/06/16/solving-the-user-cannot-be-found-error-with-spfile-author) for a similar problem.

The script **fix-author.ps1** is best used this way:
1. Run it against a document library where problem files are located in the 'report' mode. The script will check all files in the doc lib for incorrect authors logins and generate a report, containing the list of all checked subfolders and files and the semicolon separated list of all found incorrect/missing logins. By default, the script outputs information in the console and you can also output the subfolder and files report to the standard out stream and further on to a file if you like (-StdOut switch piped to Out-File).
2. Using the list of incorrect/missing logins produced at the first step, find a correct login for each bad one. Create a PowerShell hashtable with the mappings old_login->new_login.
3. Run the script again in the 'fix' mode and feed the mappings hashtable from the step 2 as the -LoginsMapping argument.

Please use Get-Help (i.e. "Get-Help .\fix-author.ps1" or "Get-Help .\fix-author.ps1 -examples") for more details on the usage of this script.
