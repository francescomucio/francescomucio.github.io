---
layout: post
title:  "Monitoring Exasol"
date:   2019-06-19 06:34:56 +0200
tags: [exasol, sql]
---
Who is slowing down this database? Not sure about you, but I got used to the speed of Exasol... except when it is not that fast. 

Once at a conference I saw a booth where they were giving away stickers and t-shirt with "I am only here because my query is running". Now I am a sucker for conference swag, but my reaction was more "What the hell is wrong with you people? Go fix your query!" I didn't get that t-shirt. I am still upset thinking about it.

Having and in-memory MPP database means that queries should return results in an handfull of seconds if you are not exporting huge amount of data or doing something stupid. When someone is doing something stupid on a shared system all the users will suffer, which is not nice. 

Because I had my share of not nice queries to check and kill in the past, I ended up creating an [Exasol Watchdog](https://www.slideshare.net/slideshow/embed_code/key/hQtDjc2JnLdxtO) at Zalando. Now this will be the subject of a future post (when I will probably rewrite the Watchdog in a way that satisfies me more), but I wanted to share a query I wrote recently for a personal project to see what is happening on my database.

I am using it quite often, so I put it in a view and call it even quicker using the Datagrip shortcuts.

The main point for me is to have a very quick overview of the number of queries running, the users running queries and the users with more concurrent sessions. Sorting the session by execution time gives me a good idea of who could be the culprit and I am still not sure the next step is to look at the `temp_db_ram` column.

Once I have identified a bad session, I go checking its profile to see which query step (a join? a grouping operation?) is problematic, then I tried to rewrite it in a better/nicer/more performing way. When I am happy it is finally time to contact my colleague and explain what I did to make it's query faster or to understand the purpose of the query to find a different solution.

```sql
  with sessions as ( select s.session_id,
                            s.stmt_id,
                            s.user_name,
                            s.status,
                            s.command_name,
                            right(s.duration, 2) + 60 * regexp_substr(s.duration, '(?<=:)[0-9]{2}(?=:)') +
                            3600 * regexp_substr(s.duration, '^[0-9]+(?=:)') duration,
                            s.temp_db_ram,
                            s.sql_text,
                            s.priority,
                            s.resources,
                            s.activity
                       from sys.exa_dba_sessions s 
       ),
       checks   as (
              select session_id,
                     stmt_id,
                     user_name,
                     priority,
                     resources,
                     activity,
                     status,
                     command_name,
                     case
                            when status = 'IDLE' then duration
                            else 0
                     end idle_duration,
                     case
                            when status in ('COMPILE', 'PREPARE SQL') then duration
                            else 0
                     end compile_duration,
                     case
                            when status not in ('IDLE', 'COMPILE', 'PREPARE SQL') then duration
                            else 0
                     end query_duration,
                     case
                            when length(sql_text) = 2000000 then 1
                            else 0
                     end passed_query_length,
                     temp_db_ram temp_db_ram,
                     sql_text
                from sessions 
       )
select count(1) over (partition by c.status) cnt_status,
       c.status,
       count(1) over (partition by c.user_name, c.status) cnt_user_status,
       c.user_name,
       count(1) over (partition by c.user_name) cnt_user,
       c.priority,
       c.resources,
       c.activity,
       c.command_name,
       c.query_duration,
       c.temp_db_ram,
       c.idle_duration,
       c.compile_duration,
       c.query_duration,
       c.passed_query_length,
       c.session_id,
       c.stmt_id,
       c.sql_text
  from checks c
 order by c.status,
          c.user_name,
          c.query_duration desc
;
```