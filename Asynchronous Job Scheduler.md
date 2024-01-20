# Asynchronous task scheduler. Past, present and future.

To avoid cognitive dissonance, let me immediately clarify that hereinafter, by scheduler I will mean a scheduler, not a planner.

## The past or how the planner appeared.

Briefly, the question of why I wrote my asynchronous task scheduler can be answered this way: because I was tired of waiting for someone else to do it.

Since 2007 I have been involved in billing.
Initially it was written in Perl with a Postgres database, later it was partially translated into Django.
Most billing tasks could be solved directly in the database.
For example, calculate something, charge something, turn it on, turn it off.
But they were solved by external scripts in Perl and Python, which essentially executed only SQL queries in the database.
Therefore, when at the next interview in the summer of 2018 they asked me what I was missing in Postgres, without hesitation, I named asynchronous triggers in the first place.
(In second place are foreign keys to partitioned tables.
On the third - a multimaster.)

Asynchronous triggers are like triggers, but they are executed asynchronously.
For example, a client in billing makes a payment to his personal account through a payment gateway.
The payment gateway requests information from billing about the existence of such a personal account and the possibility of replenishing the balance.
If the response is successful, the payment is made.
Next, billing calculates the balance for the personal account and, depending on the result, can take further actions, for example, turn on the phone/Internet.
All these actions are essentially just SQL queries in the database and can be hardwired into stores and triggers.
But regular triggers are synchronous with respect to the request.
Therefore, in this case, the payment gateway would have to wait until the billing completes all the necessary actions, although they are not at all necessary for the payment gateway.
But in the case of asynchronous triggers, these actions can be performed asynchronously, without slowing down the payment gateway, no matter how many there are and no matter how long they take to complete.

First, I looked for what was ready to solve this issue and found
1) [PGQ](https://github.com/pgq/pgq) - asynchronous queues directly in the database
2) [pg_cron](https://github.com/citusdata/pg_cron) - cron directly in the database
3) [pg_background](https://github.com/vibhorkum/pg_background) - executing a request in the background asynchronously

and of course commercial scheduler [pgpro_scheduler](https://habr.com/ru/company/postgrespro/blog/335798/).
Later, when I had already written my own scheduler, I accidentally found another open scheduler [generic-scheduler](https://github.com/okbob/generic-scheduler).

First I tried to use asynchronous queues [PGQ](https://github.com/pgq/pgq).
To work, they require a [ticker manager](https://github.com/pgq/pgqd) - an external program that assigns tasks to be performed by executors - also external programs that constantly poll the database for new tasks for them.
This architecture did not suit me and the first thing I did was move the ticker to the database as a background worker process.
This is how the initial version of my scheduler appeared [pgqbw](https://github.com/RekGRpth/pgqbw).
But all the same, the executors remained external programs and, moreover, a certain number of them should always be running, even if there were no tasks for them at all.
This approach did not suit me; I would like to dynamically launch executors when there are tasks and stop them when there are no tasks.

[pg_cron](https://github.com/citusdata/pg_cron) works in a similar way, but at that time it had many other limitations that did not suit me either.
For example, it checked for new jobs no more than once per minute (my scheduler can check no more than once per millisecond).
Also [pg_cron](https://github.com/citusdata/pg_cron) could not run multiple tasks at the same time (my scheduler does not have such a limitation).
[pg_cron](https://github.com/citusdata/pg_cron) could also be launched in only one database for the entire cluster (my scheduler can be launched in each database, and more than once if necessary).
Also, [pg_cron](https://github.com/citusdata/pg_cron) was just cron, i.e. some periodic tasks, where it was impossible to complete something once on a certain date at a certain time.

Also [pg_background](https://github.com/vibhorkum/pg_background) didn’t work for me, because it’s just asynchronous execution of a request in the background, and almost uncontrollably.
Therefore, because I could not use the open solutions found as I needed, I decided to write my own asynchronous task scheduler and called it [pg_task](https://github.com/RekGRpth/pg_task).

I started working on the scheduler in the fall of 2018, at that time the current version of Postgres was 10, but soon it changed to 11.
Time passed, versions of Postgres changed, the scheduler improved, I tried to keep up with new versions of Postgres.
In the fall of 2021, when the current version of Postgres was already 14, I was asked what versions the scheduler supports.
Then I thought that, in theory, it should support older versions.
Imagine my disappointment when the scheduler did not compile even on the previous (12) version!
I began to study this issue in more detail and gradually, in about a month, I managed to reduce the minimum supported version all the way to 9.4.
Why 9.4?
Yes, because already in 9.3 dynamic background processes have not yet appeared in Postgres, which I very actively use in the scheduler.

At that time, I had not yet heard of Greenplum, so the minimum supported version of the 9.4 scheduler was a pleasant coincidence.
In the summer of 2022, I learned about Greenplum and, when in the fall of 2022 I had already studied it a fair amount, I thought, why not make support for Greenplum in the scheduler.
In just a couple of hours, the scheduler was already working in versions 6 and 7 of Greenplum.
This was done so quickly thanks to the well-thought-out architecture of the scheduler, as well as the fact that I had previously managed to make the scheduler support version 9.4 of Postgres, on the basis of which version 6 of Greenplum works.

In the process of working on the scheduler, I realized that it can execute not only any SQL queries, such as `SELECT`, `UPDATE`, `DELETE`, `VACUUM`, `CREATE DATABASE`, etc., but also more wide possibilities realized through various extensions.
For example, for billing, I attached a template engine [pg_mustach](https://github.com/RekGRpth/pg_mustach) to Postgres, with which I can template subscriber accounts directly in the database, namely, get a ready-made beautiful HTML document.
Then, using HTML to PDF conversion [pg_htmldoc](https://github.com/RekGRpth/pg_htmldoc), I can convert this HTML document into a PDF document directly in the database.
And finally, using [pg_curl](https://github.com/RekGRpth/pg_curl) send this PDF document to the billing subscriber by email.
Yes, `libcurl` can make not only HTTP requests, but also send email.
I even once taught him how to execute commands via SSH so that Postgres could disable and enable the Internet via SSH on Cisco and Mikrotik.
Unfortunately, my pull request was rejected because the author of the curl didn't like the URL syntax.
And therefore, to execute SSH commands from Postgres, I began to use the `PLSH` extension.

Also, I was always interested in seeing in real time what was happening in the table with tasks, so I installed a couple of plugins for `nginx`.
One of them [nginx-push-stream-module](https://github.com/RekGRpth/nginx-push-stream-module) adds websockets to `nginx`, and the second one - [ngx_pq_module](https:/ /github.com/RekGRpth/ngx_pq_module) - connects `nginx` directly to Postgres.
And now I can send asynchronous `LISTEN` / `NOTIFY` notifications, which can be generated by task table triggers, directly to the browser via a web socket.
Thus, having added the necessary frontend in a JavaScript, I can see the execution of tasks in real time, i.e. changes to the corresponding table.

## The present or what the planner can do.

In short, the asynchronous task scheduler [pg_task](https://github.com/RekGRpth/pg_task) can execute any SQL queries at any given time asynchronously.

To start the scheduler, just register it in `shared_preload_libraries`
```conf
shared_preload_libraries = 'pg_task'
```
and restart the Postgres cluster.
In this case, by default, the scheduler will launch a new background worker process `pg_work` in the `postgres` database from the `postgres` user and create a table in it with `task` tasks in the `public` schema, and then check for the availability of tasks for execution in this table every second.

To execute a certain SQL query asynchronously as quickly as possible, just insert it into the `input` text field of the task table:
```sql
INSERT INTO task (input) VALUES ('SELECT now()'); -- execute the request locally as quickly as possible
```
At the same time, during the next check of tasks for execution (which happens every second, by default), the scheduler will see this new task and launch a new background worker process `pg_task` (with the same database and user as `pg_work`), which will execute the request, writing the result back to the table with tasks in the `output` text field.

If the request needs to be executed in a remote database, then the connection parameters to it can be passed to the `remote` text field:
```sql
INSERT INTO task (input, remote) VALUES ('SELECT now()', 'user=user host=host'); -- execute the request remotely as quickly as possible
```
In this case, again, during the next check, the scheduler will connect to the specified remote database, execute a query in it and write the result back to the table.

When executing queries, the scheduler records the current execution status in an enumerable field `state`, the values of which can be
- `PLAN` - the task is scheduled for execution (by default)
- `TAKE` - task taken for execution
- `WORK` - the task is being executed
- `DONE` - task completed
- `STOP` - the task does not need to be performed

Before executing a request, the scheduler records the actual start time of execution in the `start` field, as well as the identifier of the process processing the request in the `pid` integer field.

After the request is executed, the scheduler records the actual end time in the `stop` field.

If some error occurs while executing the request, the scheduler will write an error message in the `error` text field:
```sql
INSERT INTO task (input) VALUES ('SELECT 1/0');
INSERT INTO task (input, remote) VALUES ('SELECT 1/0', 'user=user host=host');
```
To execute a request at a specified time, just specify it in the `plan` field (by default, the current time will be there):
```sql
INSERT INTO task (plan, input) VALUES (now() + '5 min':INTERVAL, 'SELECT now()'); -- complete the request in 5 minutes
INSERT INTO task (plan, input) VALUES ('2029-07-01 12:51:00', 'SELECT now()'); -- execute the request at the specified time on the specified date
```
In this case, the scheduler will execute such requests only when the specified time occurs.

To automatically repeat a request at regular intervals, you can specify the required interval in the `repeat` field:
```sql
INSERT INTO task (repeat, input) VALUES ('5 min', 'SELECT now()'); -- repeat the request every 5 minutes
```
In this case, the scheduler will execute the request every 5 minutes (automatically creating a new task for this, which stores all the necessary fields).

By default, the scheduler executes only one task at a time, but if you need to execute several, this can be specified in the `max` integer field (by default, there will be 0):
```sql
INSERT INTO task (max, input) SELECT 1, 'SELECT pg_sleep(10)' FROM generate_series(1, 10); -- execute 10 queries while executing two queries at a time
```
Here the number 1 in the `max` field means to launch one task in parallel to an already running one, i.e. in total, 2 tasks will be performed simultaneously.

If, while tasks as above are being performed, enter the number 2 in the `max` field
```sql
INSERT INTO task (max, input) VALUES (2, 'SELECT now()'); -- execute a request in parallel to two already running ones
```
then the scheduler will launch a new task in parallel to two already running ones, despite the fact that all tasks that had the number 1 in the `max` field have not yet been completed.
Also, this can be considered as a kind of execution priority in tasks.

Here it is also necessary to mention that the scheduler executes tasks in the order in which they arrive sequentially (all other things being equal), i.e. ordering the table by the auto-incrementing integer field `id`.

Tasks can be grouped by specifying different `group` text fields (by default, there will be a `group` line):
```sql
INSERT INTO task (group, max, input) SELECT 'one', 1, 'SELECT pg_sleep(10)' FROM generate_series(1, 10); -- execute 10 queries while simultaneously executing two queries in one group
INSERT INTO task (group, max, input) SELECT 'two', 1, 'SELECT pg_sleep(10)' FROM generate_series(1, 10); -- execute 10 requests while simultaneously executing two requests in another group
```
Those. 4 requests will be executed simultaneously, 2 for one group and 2 for another.
In fact, the grouping is done by the integer field `hash`, which is calculated as a hash of the `group` and `remote` fields.

That, the scheduler can execute queries sequentially and/or in parallel.
But there is also a third possible option - anti-parallel execution of queries.
This is when requests are executed sequentially one by one (in a group), but with specified pauses between executions of requests.
Of course, this can always be done by specifying the required scheduled execution time for queries, but it is much easier to specify the pause as a negative number of milliseconds in the `max` field:
```sql
INSERT INTO task (max, input) SELECT -5000, 'SELECT pg_sleep(10)' FROM generate_series(1, 10); -- execute 10 queries anti-parallel, i.e. performing them sequentially with a pause of 5 seconds between executions
```
Here, a negative number in the `max` field means the number of milliseconds to pause between requests.

With automatic repetition of queries, as well as with anti-parallel execution of queries with pauses, interesting situations may arise when the duration of the request is longer than the automatic repetition interval or than the pause between anti-parallel executions.
In such situations there are two options:
1) count the next repetition (or pause) after the completion of the request, while the beginning of the next task will constantly shift, “drift”
2) or calculate the repetition (or, accordingly, pause) from the scheduled time of the previous task, so that the beginning of the next task will be clearly separated from the planned time of the previous task by a multiple repetition (pause) interval

These options can be specified in the `drift` boolean field (by default, it will be `false`).

There are several more useful fields in the task table.

The `active` field is used to indicate the interval (relative to the scheduled execution time) when the task is still active, i.e. relevant (by default, there will be 1 hour).
If during this time the scheduler does not have time to complete the task, then in the future (when it comes to its turn), this task will not be executed, and an error will be immediately written that the task is overdue.
In this case, obviously, if the task was repetitive, then the next repetition for the task will be created, which in the future can already be completed in a certain time.

If the boolean field `delete` is specified (and by default it is specified there as `true`) and as a result of executing the request there are no errors and the result itself is empty, then after execution the scheduler will delete the task record from the table with tasks.
And in the same way, if the task was repetitive, then the next repetition for the task will be created.

I can also limit the execution time of a task by specifying a non-negative interval in the `timeout` field (by default there is 0, which means unlimited execution time):
```sql
INSERT INTO task (timeout, input) VALUES ('5 sec', 'SELECT pg_sleep(10)'); -- execute the request with a timeout of 5 seconds
```
In this case, an error will occur, because the request execution time will be longer than the timeout.

If I need to execute many requests in the same group, then it would be more logical not to launch a new background worker process for each request, but to process many requests in one such process.
To control such situations, I use the `live` field, in which I can set the lifetime of the background worker process, and/or the non-negative integer `count` field, in which I can limit the number of requests processed by the background worker process.
A default value of zero in both of these fields means that the background worker process will only process one request.

To custom format the results of query execution, I can use the following fields.

If the boolean field `header` is specified (default is `true`), then headers will be shown in the results.

If the boolean field `string` is specified (default is `true`), then only strings will be quoted in the results.

The `delimiter` field (by default this is a tab character) specifies the delimiter between columns in the results.

The `quote` field (default is an empty string) specifies the quote character for quotation.

The `escape` field (default is an empty string) specifies the character to be escaped.

And finally, the `null` field (by default it is `\N`) specifies the character for NULL.

As mentioned above, the scheduler can be launched simultaneously in several databases.
To control this, the `pg_task.json` variable is used, which can only be set in the configuration file.
But you can apply its changes even simply by reloading the configuration.
This variable stores the scheduler configuration table serialized in `json(b)` form, which consists of the following columns:
- `data` - the name of the database in which the scheduler should be launched (by default it is `postgres`)
- `reset` - interval for resetting stuck tasks (by default it is 1 hour)
- `run` - the maximum number of tasks simultaneously performed by a worker (by default this is 2147483647)
- `schema` - the name of the schema in which the scheduler will create and use the table with tasks (by default it is `public`)
- `table` - the name of the table with tasks (by default it is `task`)
- `sleep` - interval between checks for new tasks (by default it is 1 second)
- `user` - the name of the user under which the scheduler will run (by default it is `postgres`)

Since the `pg_task.json` variable stores the json value of the table, I can omit some columns, which will be filled with default values.
Also, if the specified database and/or user does not exist, they will be created by the scheduler.

For example the following setting
```conf
pg_task.json = '[{"data":"database1"},{"data":"database2","user":"username2"},{"data":"database3","schema":"schema3"},{"data":"database4","table":"table4"},{"data":"database5","sleep":100}]'
```
will launch the scheduler in the database `database1` under the user `database1` (since if a user is not specified, then his name is taken as the database name), in the database `database2` under the user `username2`, in the database `database3` under the user ` database3` in the schema `schema3`, in the database `database4` under the user `database4` with the table `table4`, and finally in the database `database5` under the user `database5` and with a task check interval of 100 milliseconds, with the remaining fields not specified will be taken by default.

The scheduler uses a multi-level system of default values, in some cases it can reach up to 5 levels!
For example, the value of the `drift` field can be set when inserting into a table with tasks, either in the current session, or for the current user, or for the current database, or in the configuration file.

## The future or how the scheduler works.

In short, the asynchronous task scheduler [pg_task](https://github.com/RekGRpth/pg_task) is a background worker process that launches more background worker processes that monitor task tables and launch background worker processes that execute tasks.

The logical structure of my scheduler was largely influenced by the task scheduler from the wonderful and very unusual interesting Python web framework [web2py](https://github.com/web2py/web2py).
In that scheduler, tasks and information about their runs are stored in two different tables, and a recurring task has one entry in the tasks table and many entries in the launches table.
I put everything in one table with tasks, while repeating a task is just a new entry in the table with tasks.
This idea was suggested to me by one person, although he was not talking about a scheduler, but about organizing business processes between departments when transferring tasks between them.
What I really don't like about the [web2py](https://github.com/web2py/web2py) scheduler is that it has to have workers running all the time to perform tasks.
I was never able to make these workers dynamically start and stop depending on the number of tasks, but I managed to do this in my scheduler.

So, the scheduler creates and works with just one task table with the following columns:

| Title | Type | Null? | Default | Description |
| --- | --- | --- | --- | --- |
| id | bigserial | NOT NULL | autoincrement | identifier - primary key |
| parent | bigint | NULL | pg_task.id | ID of the parent task (if there is one), similar to a foreign key, but without the explicit constraint for performance |
| plan | timestamptz | NOT NULL | CURRENT_TIMESTAMP | planned date and time of task launch |
| start | timestamptz | NULL | | current date and time of task launch |
| stop | timestamptz | NULL | | current task completion date and time |
| active | interval | NOT NULL | pg_task.active | positive period after the planned time during which the task is relevant for execution |
| live | interval | NOT NULL | pg_task.live | non-negative lifetime of the current background worker process to perform multiple tasks before exiting |
| repeat | interval | NOT NULL | pg_task.repeat | non-negative period of automatic task repetition |
| timeout | interval | NOT NULL | pg_task.timeout | non-negative period of allowed task execution time |
| count | int | NOT NULL | pg_task.count | non-negative number of tasks performed by the current background worker process before exiting |
| hash | int | NOT NULL | generated by group and remote | hash for grouping tasks |
| max | int | NOT NULL | pg_task.max | the maximum number of tasks simultaneously executed in a group, a negative value means a pause in milliseconds between tasks in the group |
| pid | int | NULL | | ID of the process executing the task |
| state | enum state (PLAN, TAKE, WORK, DONE, STOP) | NOT NULL | PLAN | task status |
| delete | bool | NOT NULL | pg_task.delete | delete a task if there is no result and there is no error? |
| drift | bool | NOT NULL | pg_task.drift | calculate next time relative to completion time instead of planning time? |
| header | bool | NOT NULL | pg_task.header | show titles in results? |
| string | bool | NOT NULL | pg_task.string | quota only rows in results? |
| delimiter | char | NOT NULL | pg_task.delimiter | separator between columns in results |
| escape | char | NOT NULL | pg_task.escape | escape character in results |
| quote | char | NOT NULL | pg_task.quote | quote character in results |
| data | text | NULL | | some user data |
| error | text | NULL | | received errors when executing the request |
| group | text | NOT NULL | pg_task.group | name for task grouping |
| input | text | NOT NULL | | SQL command(s) to execute |
| null | text | NOT NULL | pg_task.null | placeholder for NULL in results |
| output | text | NULL | | obtained result(s) |
| remote | text | NULL | | string for connecting to a remote database (if necessary) |

Almost all default values in the table with tasks are taken from the following variables

| Title | Type | Default | Level | Description |
| --- | --- | --- | --- | --- |
| pg_task.delete | bool | true | configuration, database, user, session | delete a task if there is no result and there is no error? |
| pg_task.drift | bool | false | configuration, database, user, session | calculate next time relative to completion time instead of planning time? |
| pg_task.header | bool | true | configuration, database, user, session | show titles in results? |
| pg_task.string | bool | true | configuration, database, user, session | quota only rows in results? |
| pg_conf.close | int | 60 * 1000 | configuration, database, super-user | close background worker processes in pg_conf within the specified milliseconds |
| pg_conf.fetch | int | 10 | configuration, database, super-user | process the specified number of lines at a time in pg_conf |
| pg_conf.restart | int | 60 | configuration, database, super-user | interval in seconds to restart pg_conf on errors |
| pg_task.count | int | 0 | configuration, database, user, session | non-negative number of tasks performed by the current background worker process before exiting |
| pg_task.fetch | int | 100 | configuration, database, user | process the specified number of tasks at a time |
| pg_task.id | bigint | 0 | session | current task ID (read-only) |
| pg_task.limit | int | 1000 | configuration, database, user | limit the number of tasks processed at a time |
| pg_task.max | int | 0 | configuration, database, user, session | the maximum number of tasks simultaneously executed in a group, a negative value means a pause in milliseconds between tasks in the group |
| pg_task.run | int | 2147483647 | configuration, database, user, session | maximum number of tasks simultaneously performed by a worker |
| pg_task.sleep | int | 1000 | configuration, database, user | period in milliseconds to check for new tasks |
| pg_work.close | int | 60 * 1000 | configuration, database, super-user | close background worker processes in pg_work within the specified milliseconds |
| pg_work.fetch | int | 100 | configuration, database, super-user | process the specified number of lines at a time in pg_work |
| pg_work.restart | int | 60 | configuration, database, super-user | interval in seconds to restart pg_work on errors |
| pg_task.active | interval | 1 hour | configuration, database, user, session | positive period after the planned time during which the task is relevant for execution |
| pg_task.data | text | postgres | configuration | database name for the task table |
| pg_task.delimiter | char | \t | configuration, database, user, session | separator between columns in results |
| pg_task.escape | char | | configuration, database, user, session | escape character in results |
| pg_task.group | text | group | configuration, database, user, session | name for task grouping |
| pg_task.idle | int | 60 | configuration, database, user | number of checks for new tasks before deep sleep in case of inactivity |
| pg_task.json | json | [{"data":"postgres"}] | configuration | json configuration, possible columns: data, reset, schema, table, sleep and user |
| pg_task.live | interval | 0 sec | configuration, database, user, session | non-negative lifetime of the current background worker process to perform multiple tasks before exiting |
| pg_task.null | text | \N | configuration, database, user, session | placeholder for NULL in results (and logs) |
| pg_task.quote | char | | configuration, database, user, session | quote character in results |
| pg_task.repeat | interval | 0 sec | configuration, database, user, session | non-negative period of automatic task repetition |
| pg_task.reset | interval | 1 hour | configuration, database, user | period for resetting stuck tasks |
| pg_task.schema | text | public | configuration, database, user | name of the scheme for the table with tasks |
| pg_task.table | text | task | configuration, database, user | name of the table with tasks |
| pg_task.timeout | interval | 0 sec | configuration, database, user, session | non-negative period of allowed task execution time |
| pg_task.user | text | postgres | configuration | username to run the scheduler |

In fact, table structure is not a strict requirement.
The user can add the columns he needs to the table himself or change the default values of standard columns, or even create a table for tasks himself, the main thing is that there is compliance with standard fields in terms of type casting without explicit indication.

The main principle when designing the scheduler was to use as little code in `C` as possible and, if possible, perform most of the actions in `SQL` (via `SPI`).
Even interprocess communication occurs in `SQL` through the `pg_locks` representation, so it did not have to be adapted for different specific versions of Postgres.
Also, all repeated queries are executed through prepared statements, and where more than one line of results are expected, cursors are used.

The planner implements many unusual and extraordinary solutions.
Because this is actually my first extension for Postgres, I didn’t know many things when I started making my scheduler, and therefore I did a lot of things that, as I realized much later, are usually not done (or cannot be done at all) in extensions, but I - I’ve already done this, it works and it’s quite difficult to give up because of convenience.

For example, I didn't know that to use an extension I must first create it with the `CREATE EXTENSION` command, and therefore the scheduler can be used without this command.
We can say that the scheduler is the only extension that does not use this command to communicate with the user.
Moreover, thanks to this approach, the scheduler can itself create the necessary databases and users, and also run in several copies in each database.

Also, I didn’t know that I can’t start a new background worker process when I change some configuration parameter, but I went ahead and did it.
All other extensions constantly keep a background worker process running, which monitors the necessary configuration parameters and performs any actions when they change.

Physically, in short, the scheduler consists of four main parts:
1) `init` - initialization, auxiliary functions and starting the background configuration worker process `pg_conf`
2) `pg_conf` - a background worker process, which, based on the configuration parameter `pg_task.json`, as well as the parameters specified for databases and users, determines in which databases and under which users and with what parameters to run control `pg_work`
3) `pg_work` - a background worker process that periodically polls the table with tasks and, if there are tasks, starts their execution in a remote database via `libpq` or starts a background worker process `pg_task` to perform a task in a local database
4) `pg_task` - a background worker process that executes a task in a local database (the same as the `pg_work` that launched it)

So, after installing the scheduler in `shared_preload_libraries` and restarting the cluster, the scheduler initializes its many configuration variables and starts a background configuration worker process `pg_conf` (for Greenplum this only happens on the coordinator), which, using locks, first makes sure that it is running only in a single copy.
Then `pg_conf` connects to the `postgres` database under the default user and turns the `pg_task.json` configuration parameter into a table using the `jsonb_to_recordset` function, from which it extracts rows using `SPI` cursors (into a pre-initialized list), defining in which databases and with what parameters the controlling background worker processes `pg_work` should be launched.
At the same time, which processes are already running are determined using locks.
Next, for all lines, a function is launched that first checks the existence of the user and database, and if they do not exist, it creates them, and then launches the corresponding background worker processes `pg_work`.

Parameters (such as database name, user name, schema, table name and others) are transferred to the background worker process through shared memory, which is allocated at startup.

Now the controlling background worker process `pg_work` comes into play and uses the same tactics to pass parameters to its worker processes.
First of all, it connects to the shared memory, the identifier of which was passed to it as an argument by the parent process, and reads the parameters from there.
Then, using a lock, `pg_work` makes sure that only one instance of it is running (the same lock is used in `pg_conf` to check which processes are already running).
Next, `pg_work` connects to the required database with the necessary parameters and creates a schema and a table with tasks in it, if they have not already been created, and starts the main cycle of checking for the presence of new tasks.
Because some tasks can be executed in a remote database via `libpq`, then in the same cycle the asynchronous processing of the corresponding events occurs.

As mentioned above, in the main loop the presence of new tasks is checked periodically, by default, every second.
But recently I made a significant optimization that allows the `pg_work` process to “sleep” after a certain period of no new jobs, by default this is a minute.
In this case, when new entries are inserted into the table with jobs, the `pg_work` manager is awakened by a signal sent through the trigger, and again begins to check for new jobs every second for a minute.
If there are no new tasks, then the manager falls asleep for a long time, and if there are, until the very next time.

So, every second the master process `pg_work` runs a rather complex prepared query using the `SPI` cursor to get jobs to run.
If a task needs to be executed remotely, it is launched through `libpq`, and the corresponding events are added to the main loop for processing.
And if the task needs to be executed locally, then the background worker process `pg_task` is launched, to which parameters are also passed through shared memory.
A separate worker process is necessary for local tasks, because Postgres executes queries synchronously, and also so that multiple jobs can be executed simultaneously.
For remote tasks, a separate process is not necessary, because it is already created when connecting via `libpq`.

Finally, the background worker process `pg_task` executes a task (or several) in the local database as follows.
First, it uses a lock to ensure that only one instance of it is running (the same lock is used in `pg_work` to check which tasks are already running).
Secondly, `pg_task` gets information about the task using a prepared `SPI` request (`pg_work` gets information about a remote task using the same function).
The scheduler executes a query from a task using the `exec_simple_query` function, which in Postgres is private, i.e. declared as `static`.
Therefore, I had to set up copying of the file containing this function to the corresponding version of Postgres (including for Greenplum - the corresponding version of Greenplum).
For those who don't really like this, I created a separate `spi` branch, in which the execution of a task request occurs through `SPI` with appropriate restrictions on the possible requests that can be executed.
To intercept exceptions, the standard `PG_TRY` - `PG_CATCH` construction is used, and the error message itself is intercepted by a custom `emit_log_hook` for recording in the corresponding `error` field of the table with tasks.

I see the near future of the scheduler in doing as many different tests as possible, because the ones I have done so far cover only a very small part of the scheduler’s capabilities.
Also, it is worth working on security and logging.
