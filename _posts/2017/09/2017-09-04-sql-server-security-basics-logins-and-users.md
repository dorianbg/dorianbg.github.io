---
title: "SQL Server Security Basics - Logins and Users"
date: "2017-09-04"
categories: 
  - "sql"
---

## Logins and users basics

There are two main levels of security in SQL Server.

1. Access to the server
2. Access to a database

(* there are also schemas on a database/table level but I will not cover them now)

Thus in SQL Server we have to deal with Logins and Users

- Logins can access the SQL Server
- Users within the Server can access specific databases

So there are login roles and user roles. Roles define what you can do.

For instance the most powerful login server roleis the sysadmin role because it can do anything on the server.

But there are also other less powerful roles such as dbcreator which only gives access

On a database level, the most power user database role is the dbowner role which allows the user to anything on a database. But there are other roles such as eg. db_datareader where the user can only read data from a database.

If you are accessing a specific database, that means your login was successfully linked with a user. It also means that your user account has atleast a db_datareader role on the database you are accessing.

A "big data" analogy is thinking about kerberos secured Hadoop cluster.

Your Kerberos account gives you access to HDFS.

Your HDFS account gives you access to specific files inside HDFS.

## Logins and Server roles

2.1 Create logins

The simplest way to create a login with SQL Server authentication is:

CREATE LOGIN [ETL] WITH password='password'

go

2.2 Add login to a server role

After you create a login, you can add a role to it.

Eg. add a login to dbcreator server role:

ALTER SERVER ROLE [dbcreator] ADD MEMBER [ETL]

go

Predefined server roles for logins are [here](https://docs.microsoft.com/en-us/sql/relational-databases/security/authentication-access/server-level-roles)

Also very well explained [here](https://www.mssqltips.com/sqlservertip/1887/understanding-sql-server-fixed-server-roles/)

For a data warehousing user, you should probably only be interested in the bulkadmin and publicroles.

Since public role is automatically granted, you should only add the bulkadmin role if you need to insert data using bulk insert statement.

## Users and Database roles

3.1 Create a user

The simplest way to create a user linked to our login is:

CREATE USER [ETL] FOR LOGIN [ETL]

go

3.2 Add user to a database role

Predefined database roles for users are explained [here](https://www.mssqltips.com/sqlservertip/1900/understanding-sql-server-fixed-database-roles/)

For data warehousing purposes you might want to choose either:

- these four roles: db_backupoperator,  db_ddladmin, db_datawriter and db_datareader

or just a

- db_owner role

The simplest procedure is:

```tsql
Use target_db
go
EXEC sp_addrolemember 'db_owner', [ETL]
go
Or the alternative role variant we mentioned:
Use target_db
go
EXEC sp_addrolemember 'db_backupoperator', [ETL]
go
EXEC sp_addrolemember 'db_ddladmin', [ETL]
go
EXEC sp_addrolemember 'db_datawriter', [ETL]
go
EXEC sp_addrolemember 'db_datareader', [ETL]
go
```

## Summary

Simple code to create a login, create a user linked with login and add a role to the user

```tsql
CREATE LOGIN [ETL] WITH password='password'
go
Use target_db
go
CREATE USER [ETL] FOR LOGIN [ETL]
go
EXEC sp_addrolemember 'db_owner', [ETL]
go
```
