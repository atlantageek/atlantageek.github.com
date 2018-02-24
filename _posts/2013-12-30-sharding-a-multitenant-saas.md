---
layout: post
title: "Sharding a MultiTenant SaaS app"
categories:
- programming
tags:
- postgres
- postgresql
- databases
comments: true
---

You've finally got some traction with your SaaS project.  Lots of large customers and things are moving along rather well. A few of your customers complain that the system is not as fast as it use to be but that's normal. You up the hardware, add a few indexes and things seem steady. 
All of a sudden your email/support staff is overwhelmed by customers complaining about 404 errors. You look and see that your web sessions are timing out and queries are starting to drag. Postgres Autovacuum is starting at the worse possible times and you are spending a lot of time babysitting your app instead of counting your money. You thought you had more time but now its obvious, its time to shard the database.

There are a few ways to shard the database. Some apps are sharded by time and others are sharded by grouping users. A multi-tenant app lends itself to being sharded by tenant. The idea being that each individual tenant would not want to share data with other tenant. (Notice I said tenant and not users. I assume that users who collaborate would be part of the same tenant.

There are also multiple solutions to how data is sharded. There are three approaches. 

*  webserver
*  app
*  database partitioning

Either solution requires several databases/tables for each shard.

Webserver would be assigning a tenant to a specific database/web server combination. This can work but its somewhat ugly. Basically when the user logs ont the app their server needs to be identified and all their work will be done on this server. nginx sticky-module comes in handy for this solution.

App level is a bit more messy. It requires the most code changes. When the customer logs in the database their db shard is determined and then all datarequests for this user goes through the tenant's shard.

Database partitioning approach is probably the best approach. The goal here is to split up the data across multiple tables but with minimal code changes. This is done by a combination of partitioning and postgres db triggers.

The focus will be on the Database partitioning approach.

Unlike other approaches to sharding, the database partitioning approach does not shard the whole database. Instead you identify certain tables that are the culprit in the drop of the performance and then use partitioning on that table. Partitioning is only beneficial for large tables. If the table is larger than the memory of your database machine then its a canidate for partitioning.

To demonstrate a technique assume that we are working on an accounting application. The largest table is the transaction table. The job_transaction table looks like this

	CREATE SEQUENCE id_seq;
	CREATE TABLE job_transactions (id int NOT NULL DEFAULT nextval('id_seq'), name varchar, description text, tenant_id integer);
	ALTER TABLE job_transactions_seq owned by job_transactions.id;


The job_transactions tabe probably already exists.  rename the old table to something else and create a new empty table with the old name.  We will transfer the data later.

The data needs to be split accross multiple database tables so its time to create those.

	CREATE TABLE job_Transactions0(CHECK ( (tenant_id % 0) = 0)) INHERITS (job_transactions);
	CREATE TABLE job_Transactions1(CHECK ( (tenant_id % 1) = 0)) INHERITS (job_transactions);
	CREATE TABLE job_Transactions2(CHECK ( (tenant_id % 2) = 0)) INHERITS (job_transactions);

So the data is split between these three tables. This uses the mod function to determine which partition. To truly leverage this functionality you might want to add tenant_id to all the indexes and all queries should have tenant_id as part of the query.
 
Now we need to create triggers so that any inserts into the job_transactiosn table will be put in the appropriate partition.

     CREATE OR REPLACE FUNCTION trace_insert()
     RETURNS TRIGGER AS $$
     BEGIN
       IF ((NEW.id % 3) = 0 ) THEN
          INSERT INTO job_transactions0 VALUES (NEW.*);
       ELSEIF ((NEW.id % 3) = 1 ) THEN
       IF INSERT INTO job_transactions1 VALUES (NEW.*);
       ELSEIF ((NEW.id % 3) = 2 ) THEN
          INSERT INTO job_transactions2 VALUES (NEW.*);
       ELSE
          RAISE EXCEPTION 'device_id out of range.  Fix the trace_insert_trigger() function!';
       END IF;
       RETURN NULL;
     END;
     $$
     LANGUAGE plpgsql;

     CREATE TRIGGER trace_insert_trigger
       BEFORE INSERT ON trace_refs
       FOR EACH ROW EXECUTE PROCEDURE trace_insert();

Now any inserts into the job_transactions table will be deployed the the child partitions.  And querying for data in the original table will cause the code to be requested to the individual transactions.

One note, the queries that have the  tenant_id will be fast. Those without must query multiple database tables and kill the whole performance improvements will be killed.

