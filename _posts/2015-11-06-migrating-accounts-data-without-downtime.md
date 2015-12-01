---
layout: post
category : migrations
tags : [redis, rds, dynamodb]
---

At Practo we’re constantly looking for ways to improve user experience. A major part of it is ensuring that all our services are responsive and highly available. Over the last week we were faced with a challenge. Our Redis server that stores User data, started bottlenecking on performance.

Couple of years back, when we were looking at the possibility of increasing our product base, we created Practo Accounts, a single sign on for all the Practo products and services. At that point in time, we had taken the decision of keeping the user data, though persistence was required, in Redis datastore. The reason for using Redis was simple (and stupid). We assumed that the user data will evolve and putting this data, in a datastore like MySQL would mean a lot of migrations over next few years. And Redis did give some support for persisting that data. The second assumption that we made was Redis is going to be more performant than MySQL. As always, decisions that one takes without any data backing it, could lead to becoming technical debts that will have to be repaid and this migration was one such repayment.

A few weeks ago, we observed regular spikes in Practo Accounts response times. Since it was so regular, the first suspect was our cron jobs. Further investigations revealed that we were backing up/persisting our data using BGSAVE on Redis datastore at regular intervals that matched the spikes on our application performance monitoring systems. RCA revealed that our Redis server, because of the growing size of our data and the frequent [BGSAVE](http://redis.io/commands/bgsave), was misbehaving.


We initially thought of a few workarounds,

- To disable BGSAVE on the master Redis instances and create a read replica of the Redis instance which has BGSAVE enabled and is only used for taking backup.
- Move all our Redis data to DynamoDB.

However, in the long term these did not seem like very good solutions given the fact that the data that we were trying to move had a very normalized SQL structure. The logical step was to move the data to a disk based store like MySQL. Given our data structure and the fact that we use AWS services extensively, RDS was a better choice. And it was time to do it.

The basic principles for any data migration are:

1. Any migration being deployed should be backward compatible on code and data level
2. The data should be consistent, even while it is being modified at source
3. The migration should be downtime free.


**How we accomplished it?**

We triggered a BGSAVE on our Redis master instance and before doing that, started [MONITOR](http://redis.io/commands/MONITOR) on it to capture the update commands for all the keys and pushed those commands (after parsing them to JSON) in the “ProfileEvents” queue of another Redis instance. We chose a Redis queue over AWS SQS or RabbitMQ to ensure FIFO order which isn’t guaranteed by either.

After starting Monitor, we imported the Redis BGSAVE dump to new Redis instance and migrated Redis data into a newly created RDS instance. Once the data migration was complete, a consumer was started to persist all the events from the “ProfileEvents” queue to RDS. This ensured that all changes done to the user data on production were being made to the RDS data as well and helped us maintain consistency between Redis store and RDS data.

We verified the above setup and once we were convinced that this setup was stable, we switched to the new code which used RDS as the data store for our persistent data rather than Redis.

If anything broke on production, as a rollback mechanism we had the following strategy ready, which involved a slight downtime:

1. Take a diff of the current RDS state with a point in time backup from when the code switch was made
2. Replay the diff on Redis (while the Redis monitor is stopped)
3. Start the monitor again to maintain data consistency between Redis and RDS


Thankfully, a rollback wasn’t required and the bugs found on production were very minor and were fixed. All said, this post is not a rant about/against Redis. It has served our purpose and served well and continues to serve in other places within our stack. But, it wasn’t the right choice for the above use case and was a big learning.
