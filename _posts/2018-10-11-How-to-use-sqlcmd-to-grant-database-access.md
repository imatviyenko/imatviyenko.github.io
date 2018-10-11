---
layout: post
title: "How to use sqlcmd to grant database access"
description: "You can grant yourself access to SQL database if you know the password for some other Windows login which already has the dbowner access to this database"
date: 2018-10-11
tags: 
    - SQL
---

Suppose you have an IT system which hosts its database on an SQL server instance and you know the service account AD login and password.
If your personal admin account does not have access to the SQL server database, you can give yourself permissions to this database using the instructions below.

There are a few notes:
- You have to be able to connect to the SQL server via RDP with OS admin privileges.
- The service account **must** be dbowner for this method to work, which is usually the case.
- There is a [alternative method](https://adamrehill.com/2017/02/13/how-to-grant-yourself-sysadmin-access-to-a-local-sql-server-database) which involves SQL instance restart with a special command line option.
 

## Step-by-step instructions
1. Connect to SQL server via RDP with your AD account, which **does not have access** to the SQL database

2. Start command prompt with the credentials of the service account that **has** access to the database and enter the password for the service account when prompted:
```
runas.exe /user:domain\service_account_name cmd.exe
```
  

3. Execute "sqlcmd" in the command prompt to start the SQL interactive shell.

4. Execute these TSQL commands in a sequence:  

```SQL
-- Check the current user name - you must be running sqlcmd under the service account credentials
select suser_sname()
go

use *Database_name*
go

create user [domain\your_login_name] for login [domain\your_login_name]
go


-- Add your login to the 'db_datareader' SQL database role in this example to grant read access to the data
exec sp_addrolemember 'db_datareader', 'domain\your_login_name'
go
```

