---
layout: post
title:  "Monitoring Exasol"
date:   2019-06-19 06:34:56 +0200
tags: [exasol, sql]
---




```sql
select count(1) over (partition by ds.status) cnt_status,
       m.status,
       count(1) over (partition by m.user_name, ds.status) cnt_user_status,
       m.user_name,
       ds.priority,
       ds.resources,
       ds.activity,
       m.command_name,
       check_query_duration,
       check_temp_db_ram,
       check_idle_duration,
       check_compile_duration,
       check_query_duration,
       check_passed_query_length,
       m.session_id,
       m.stmt_id,
       m.sql_text
  from zalando_util.swd_session_monitor m
  inner join sys.exa_dba_sessions       ds
             on m.session_id = ds.session_id
 order by ds.status,
          m.user_name,
          m.check_query_duration desc;
  
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
                       from sys.exa_dba_sessions s ),
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
                     end check_idle_duration,
                     case
                            when status in ('COMPILE', 'PREPARE SQL') then duration
                            else 0
                     end check_compile_duration,
                     case
                            when status not in ('IDLE', 'COMPILE', 'PREPARE SQL') then duration
                            else 0
                     end check_query_duration,
                     case
                            when length(sql_text) = 2000000 then 1
                            else 0
                     end check_passed_query_length,
                     temp_db_ram check_temp_db_ram,
                     sql_text
                from sessions )
select count(1) over (partition by c.status) cnt_status,
       c.status,
       count(1) over (partition by c.user_name, c.status) cnt_user_status,
       c.user_name,
       count(1) over (partition by c.user_name) cnt_user,
       c.priority,
       c.resources,
       c.activity,
       c.command_name,
       c.check_query_duration,
       c.check_temp_db_ram,
       c.check_idle_duration,
       c.check_compile_duration,
       c.check_query_duration,
       c.check_passed_query_length,
       c.session_id,
       c.stmt_id,
       c.sql_text
  from checks c
 order by c.status,
          c.user_name,
          c.check_query_duration desc
;
```