---
layout: post
title: "Scheduled Events in Mysql"
categories:
- programming
- mysql
tags:
- mysql
comments: true
---

## Justification

There are many options for scheduling events.  Cron in Linux, scheduler in windows, every queueing systems has a built in schedule and then there are homegrown while loops. So why should you consider Mysql event.
* Mysql is often already in your application
* Its OS agnostic. It works the same whether its linux or windows
* If the scheduled tasks is a db maintenance event it makes sense to store it on the db.

In my paticular case a requirement to create a scheduled task that exports a db table to csv on a windows environment. After fumbling with mysql access on a dos shell (Im mostly a unix/bash guy) I looked for alternatives and came across mysql events.  

## Example tasks
Assume we have a table named workjobs that needs to be exported to a csv file every hour how would we do this.

First lets create the look at how to generate an outfile on in mysql

```sql
select * from workjobs
into outfile 'workjobs.csv'
 fields terminated by ','
 enclosed by '\"'
 lines terminated by '\n'
```

This command creates a csv file called workjobs.csv on the server.  On our windows server it drops the file in the data directory of the server.
Its unfortunate that this puts the file in the data directory but there are ways around this.  If we add a path to the file name it will put the file in a subdirectory in the data directory.

```sql
select * from workjobs
into outfile 'exported_files/workjobs.csv'
 fields terminated by ','
 enclosed by '\"'
 lines terminated by '\n'
```
Now we could share the exported_files folder with another machine or create the equivalent of a symbolic link to the folder to another location that is accessible to the user or webserver.

The task is still not scheduled though.  How do we handle that.


```sql
delimiter $$
create event job_dump on schedule every 1 hour do
  begin
    select * from workjobs
      into outfile 'exported_files/workjobs.csv'
        fields terminated by ','
        enclosed by '\"'
        lines terminated by '\n';

  end$$
delimiter |
```
Here we created an event to run every hour. I put the sql in an unnecessary transaction. This is so the reader will have a starting point for more complex tasks.

For this task to be completed we needed a way to keep the workjobs.csv file from overwriting itself.  Normally this is done by adding a date/time stamp to the file name.  We do this by dynamically creating the sql statement and using the date_fomat command to create a file name.
```sql
create event job_dump on schedule every 1 minute do
  begin
    select @myCommand := concat(
        "select * from workjobs into outfile'", DATE_FORMAT(now(),""'exported_files/workjobs-%y%m%d-%H%i%s'),
        "'fields terminated by ',' enclosed by '\""' lines terminated by '\n'"
      );    
    prepare stmt from @MyCommand;
    execute stmt;
  end
```

Here we build the sql statement in @myCommand.  MySQL requires the filenames for 'into outfile' to be static so our work around is to dynamically create the sql command and then execute it.
