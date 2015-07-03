---
layout: post
title:  "Understanding innodb_buffer_pool_size"
date:   2015-06-27 21:54:22
categories:
---
There are lots of parameters when you try to optimize your mysql but the most important one is Innodb Buffer Pool. It is defined as %70 of the your server total RAM but we need more details. Let's first check the definition given from MYSQL:

> InnoDB maintains a storage area called the buffer pool for caching data and indexes in memory. Knowing how the InnoDB buffer > pool works, and taking advantage of it to keep frequently accessed data in memory, is an important aspect of MySQL tuning.

The idea is keeping innodb_buffer_pool_size bigger as possible that doesn't block other OS and programs running properly. This will make InnoDB work like a in-memory database and increase the performance a lot. At first InnoDB will take all the required data from disk and access it from memory in the future queries. The default value for InnoDB is 8MB so you might want to update it as a bigger number which fits well for your system.

Before going exact details there is a wrong assumption about Innodb Buffer Pool. People think that if they give enough space for keys to be in the memory and there will be very low IO accesss. The idea is searching keys fast as much as and retrieve data with index directly. Let's assume you have a query that runs in 600ms and 450 ms is searching keys, 75 ms is getting data from disk and last 75 ms is fetching data from server. Providing proper innodb_buffer_pool_size will make sure you don't access disk for index search and that 450 ms will decrease to 30 ms (it really depends on your keys and structure) so total time for query is 180 ms now. You still have IO at this rate but it is only retrieving data from disk which can be acceptable.

I will not get into details how InnoDB manages the pool and LRU algorithm but directly give info about what value we should choose. Run this query to get an recommended memory:
{% highlight sql %}
SELECT CONCAT(ROUND(KBS/POWER(1024,
IF(PowerOf1024<0,0,IF(PowerOf1024>3,0,PowerOf1024)))+0.49999),
SUBSTR(' KMG',IF(PowerOf1024<0,0,
IF(PowerOf1024>3,0,PowerOf1024))+1,1)) recommended_innodb_buffer_pool_size
FROM (SELECT SUM(data_length+index_length) KBS FROM information_schema.tables
WHERE engine='InnoDB') A,
(SELECT 3 PowerOf1024) B;
{% endhighlight %}
In my system it returns the value:
{% highlight sql %}
recommended_innodb_buffer_pool_size
48G
{% endhighlight %}
Be careful with these values since InnoDB reserves additional memory for buffers and control structures, so that the total allocated space is approximately 10% greater than the specified size. When you have 50 GB RAM and you give 48GB as pool, you will probably get OS paging problems.

This is a simple value that you should better have this value to reach all data properly with the help of Innodb Buffer Pool. Let's assume you set the value 45GB and you want to make sure that these resources are not wasted and buffer pool is used. At this point we should check the buffer pool usage value. It must be run after the all the heavy operations are done for your system. If your system is on high load only on weekends then you should wait for the run this query at least one or two week so you would get at least a minimal idea how the system will respond on load.
{% highlight sql %}
SELECT (PagesData*PageSize)/POWER(1024,3) DataGB FROM
(SELECT variable_value PagesData
FROM information_schema.global_status
WHERE variable_name='Innodb_buffer_pool_pages_data') A,
(SELECT variable_value PageSize
FROM information_schema.global_status
WHERE variable_name='Innodb_page_size') B;
{% endhighlight %}
For my system it returns:
{% highlight sql %}
DataGB
22.5G
{% endhighlight %}
It might be %100 (45G) or some other value depending on the system and you should take action according to this value. It means that even though your system has 45GB of indices you only access 22GB of it at this time area. Hence lowering the value as 25GB should be okay the system to run at almost the same performance. However be careful with the value and keep observing the value for some time and make sure it doesn't go up quickly after for a week etc.

There are some other parameters that needed to be adjusted as well but I will not get into these details. Let's assume you run the query for recommended buffer pool and you got 11GB. However you only have 11.5GB data in your database and it seems a bit odd to have such a big number. The reason for this you probably have too much or redundant index for your tables. You should reconsider the indices and adjust them to a proper state which doesn't make your operations slower and reserve so much space.

One other issue primary keys of very big tables. Let's say we have 20 tables with average 1 million to 5 million rows data but only two of them have 200 million rows data per table. At total you should have 450 million rows and probably with indices it would require around 50GB inno db pool. If you don't access these tables heavily or they are accessed by a big group (like 1.5 million records of logs  belong to one operation) then you should not need a really big buffer pool. However a small amount of pool will be filled with one group data easily and other operations become slower at that time. At least try to give a proper amount of memory to handle the one big operation and it's additional queries. Of course it is totally another discussion topic why don't you use some other systems (Cassandra or a different NOSQL maybe ) to store big tables to create results and put these values into mysql to be access etc.

To sum up; run the query and get a recommended value for your innodb_buffer_pool_size. If you think it is enormous for your database then try to optimize your indices and lower the number of key values. Also be careful with the very high values since it might throttle the system and cause bigger problems like uptimes. Also check other parameters to use the memory you attached more efficiently.
