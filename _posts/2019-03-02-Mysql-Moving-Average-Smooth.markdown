---
layout: post
title:  "Moving Average Smooth for Mysql"
date:   2019-03-02 12:00:00
categories:
---

In some cases you might want to have a basic filtering on sql data directly. It is not available in mysql but can make it work with a simple counter logic. Idea is:

* Select full data with counter on first loop with a specific order by section.
* Select the same data again but filter the data with the counter comparing to above.
<br>
Is it the most efficient way? It probably depends on your setup but if you just want to see results or test some data then it works quite well.
<br>
I'll be using `the same data` I used before to show an example which is loaded at  [https://www.db-fiddle.com/f/bbC16J8iMWTLisFKbAypUi/1](https://www.db-fiddle.com/f/bbC16J8iMWTLisFKbAypUi/1). Data contains position data with timestamp and speed for a specific person.  
<br>
Looking at the first part with alias `data_table` we are basically selecting all data and later it will filtered utilizing the counter_table. 
{% highlight sql %}
  select @cnt := @cnt + 1 as cnt, speed
    from rawdata,
       (select @sum := 0, @cnt := 0) tmp
    where event_id = 68162
    and team_id = 1
    and jersey = 55
    order by timestamp
    limit 17823232
{% endhighlight %}
<br>
This part might feel a bit tricky but it just initializes the temporary variables we will be using in the process. 
{% highlight sql %}
from rawdata , (select @sum := 0, @cnt := 0) tmp
{% endhighlight %}
<br>

Next part is counter_table for setting temporary counters and use it while filtering the data. Be aware id can not be used for this since there might be data belongs to other people. Whole point of using temporary counters is creating fake ids which are ordered properly. 
{% highlight sql %}
select @counterTmp := @counterTmp + 1 as counterTmp,   timestamp
      from rawdata ,
           (select @sum := 0, @counterTmp := 0) tmp
      where event_id = 68162
        and team_id = 1
        and jersey = 55
      order by  timestamp
           limit 17823232) counter_table;
{% endhighlight %}
<br>

Also another thing you might notice `limit 17823232` and why we need that exactly. New mysql versions might put some auto limits to inner queries while using order by and if you are querying over a big table it might result in wrong data.
<br>
With merging these two queries and filtering the data as `where data_table.cnt >= counter_table.counterTmp - 7 and data_table.cnt <= counter_table.counterTmp + 7` we get avg of speed data from row 1 to 15 on 8th data. And of course it moves forward as avg of 2 to 16 on 9th etc. Be aware we can not have average of first 7 data points using this.
{% highlight sql %}
   select counter_table.counterTmp, case when
      counter_table.counterTmp >= 8 then (select sum(speed)/count(1)
        from (
               select @cnt := @cnt + 1 as cnt, speed
               from rawdata ,
                    (select @sum := 0, @cnt := 0) tmp
               where event_id = 68162
                 and team_id = 1
                 and jersey = 55
               order by timestamp
                   limit 17823232
             ) data_table
        where data_table.cnt >= counter_table.counterTmp - 7
          and data_table.cnt <= counter_table.counterTmp + 7
       ) else null end as avg,
       counter_table.timestamp
from (select @counterTmp := @counterTmp + 1 as counterTmp,   timestamp
      from rawdata ,
           (select @sum := 0, @counterTmp := 0) tmp
      where event_id = 68162
        and team_id = 1
        and jersey = 55
      order by  timestamp
           limit 17823232) counter_table;
{% endhighlight %}
