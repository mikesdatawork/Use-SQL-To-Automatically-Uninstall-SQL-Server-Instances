![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")        

# Use SQL To Automatically Uninstall SQL Server Instances
**Post Date: November 15, 2016**        



## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process

<p>Recently I had a situation where I needed to test out the limits, and configuration patterns of SQL Server Instance Creations on a Multi-Instance Database Server. In this case I was using SQL Server 2014 Standard, and running install after install. Changing settings (post install), and running through more install scenarios.
As you can imagine; installing over 15 instances can be somewhat tedious; even if you are using unattended installation config files… You still need to ensure the install has a different instance name, and then run through a variety of checks (DBA Wisdom) just to make sure everything was ok.
Instead of doing this I decided to make my process more uniform, and nothing gets better than adding some automation.
The following SQL logic will Automatically create the unattended 'uninstall' configs file, then run through the uninstallation completely.
Note: One of the best things about uninstalling SQL Instances using the unattended config file is that each uninstall does not need to be followed up with rebooting the server. Therefore you can uninstall again, and again without having to restart the machine. Ultimately you'll need to uninstall it, but with this process it can be managed at an appropriate time.
Note: This logic will uninstall 1 instance at a time, and it always targets the most recently installed instance. This process assumes you have a uniform naming convention. In this example we are using SQLSHARE01, 02, 03… It targets the most recent instance which is SQLSHARE03. 
The following actions are taken:
1. Creates a variable table and creates a list of the existing instances.
2. Presents you with the list of instances.
3. Creates the unattended UNINSTALL configuration file using the most recent instance name.
4. Runs the process of uninstalling using QUIET mode.
5. Provides you with a new updated list of the remaining instances.
Suppose you have 16 instances. If you want to keep automatically removing the instance; simply run the below logic again, and again… It will keep removing the instances for as many times you run it.
Hope this is useful.</p>      


## SQL-Logic
```SQL
use master;
set nocount on
 
declare @server_name                varchar(255) = (select @@servername)
declare @run_bcp                varchar(8000) 
declare @silent_uninstall_string    varchar(max)
 
-- create table to hold instance names
declare @sql_instances table
(
    [id]        int identity(0,1)
,   [rootkey]       nvarchar(255)
,   [sql_instances] nvarchar(255)
,   [value]     nvarchar(255)
)
  
-- get instances names using xp_regread
insert into @sql_instances ([rootkey], [sql_instances], [value]) execute xp_regread
    @rootkey        = 'hkey_local_machine'
,   @key            = 'software\microsoft\microsoft sql server'
,   @value_name     = 'installedinstances'
 
-- get list of existing instances
select 'existing_sql_instances' = [sql_instances]   from @sql_instances
 
-- set prefix name for multi-instance environment aka: Shared SQL Environment "SQLSHARE"
-- produce the next instance name.
declare @prefix     varchar(255) = 'SQLSHARE'
declare @latest_instance    varchar(255) = 
(
select
    case
    when max([sql_instances]) = 'MSSQLSERVER' then @@servername
            else max([sql_instances])
    end as [sql_instances]
from
    @sql_instances si
)
-- reference: https://msdn.microsoft.com/en-us/library/ms144259(v=sql.120).aspx
 
-- create uninstall config file
set @silent_uninstall_string    =
'[OPTIONS]
ACTION          ="Uninstall"
ENU         ="True"
QUIET           ="True"
QUIETSIMPLE     ="False"
FEATURES        =SQLENGINE,RS
HELP            ="False"
INDICATEPROGRESS    ="False"
X86         ="False"
INSTANCENAME        ="' + @latest_instance + '"
'
if object_id('tempdb..##silent_uninstall_string') is not null drop table ##silent_uninstall_string
create table ##silent_uninstall_string ([bcpcommand]    varchar(8000)) insert into  ##silent_uninstall_string select @silent_uninstall_string
 
select  @run_bcp = 
'bcp "select [bcpcommand] from ##silent_uninstall_string" ' + 'queryout "e:\sql_silent_uninstall_config_file\unattended_uninstall_sql_2014_standard.ini" -c -t -T -S' + @server_name exec master..xp_cmdshell @run_bcp
 
-- run uninstall of the latest sql instance installed exec master..xp_cmdshell 'e:\sqlt1svgen1_dependencies\sql_server_2014\standard_x64\setup.exe /configurationfile=e:\sql_silent_uninstall_config_file\unattended_uninstall_sql_2014_standard.ini'
 
-- create table to hold instance names
declare @sql_remaining_instances table
(
    [id]        int identity(0,1)
,   [rootkey]       nvarchar(255)
,   [sql_instances] nvarchar(255)
,   [value]     nvarchar(255)
)
  
-- get instances names using xp_regread
insert into @sql_remaining_instances ([rootkey], [sql_instances], [value]) execute xp_regread
    @rootkey        = 'hkey_local_machine'
,   @key            = 'software\microsoft\microsoft sql server'
,   @value_name     = 'installedinstances'
 
-- get list of remaining instances
select 'remaining_sql_instances'    = [sql_instances]   from @sql_remaining_instances
```


[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

     
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")

