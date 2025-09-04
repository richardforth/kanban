# Database ideas

## PreAmble
In the book 'Kanban', by David J. Andersen, several tools were recommended which no
longer exist (the book is quite old) so it got me thinking about implementing an
in-house kanban software.

In a later chapter, one thing stood out and really clicked with me, that mentioned
database triggers, that emailed the Project Manager when a free slot became available
in an input queue, or buffer by implementing and comapring actual work in progress, 
(WIP) with agreed WIP Limits.

## First table idea 

So that immediately got me thinking about first ideas for a database table design
that had WIP limits defined for each input queue for example., suppse we have a
database called `kanban`, we could have a table called `wip_limits` which would need
to be designed around the flow model for your specific value stream model, including
any buffers, so these are only ideas here, so lest assume we have 4 value streams
with a buffer for each (assuming you need input buffers), it might look like this:

(Note some buffers, such as the `backlog` (the initial input point for the value
stream) might need to be unlimited, so I have assigned a `wip_limit` here of -1,
representing `unlimited` as we do not want to affect thw rok streams of other parts
of the business or customer interactions)

```
+----+---------------+-----------+
| id | queue_name    | wip_limit |  
+----+---------------+-----------+
| 1  | backlog       |        -1 |
| 2  | triage        |         3 |
| 3  | development   |         3 |
| 4  | test          |         3 |
| 5  | uat           |         5 |
...etc
```

In this way its possible to see how triggers might be placed on other tables holding
these items through the value stream to say when a particular queue is either overloaded
or has capacity to enable a pull from the `triage` queue (if implemented) or directly
from the `backlog`.

## What is WIP?

WIP stands for Work In Progress. By limiting WIP, we improve quality and increase flow.
It has been found that people or workstations with too many concurrent things to juggle
result in poorer quality and longer lead times.

By implementing Kanban and limiting WIP, we can start to see exactly where the bottlenecks
are and improve flow by introdducing new buffers or lmiting WIP up or down stream, or
by addressing the bottlenecks within the value stream to improve cadence and flow.

Also, limiting WIP creates slack, which leads to continuous improvement opportunities.

A common management technique is to eliminate slack to improve efficiency, but this can
lead to unforeseen consequences including the build up of technical debt, and burn out.

Liming WIP enables workstations to clean house, and find improvements which increases
quality and flow, which perhaps for the first time in a long time they have had time
to implement improvements and pay down technical debt.


## Other tables, views etc 

Within each table that represents a stage or workstation in the value stream, they should
maintain the same columns which represent fields of a kanban card that might be found
on a physical kanban card wallm ideas for this include, but not limited to (remember each
value stream and card design might look different, but they must maintain the same structure
throughout the value stream:

```
- id (auto-increment, primary key etc...)
- Date Logged
- Date Due
- Past Due Y/N
- Reference ID No, eg Ticket No
- Short Description
- Designated/Assigned Person (Named Engineer, Developer, Tester etc doing the work)
- Dependency (linkage to other flow items?)
```

## A main queues table that tracks work items through the value stream

Such a database might be able to be mormalised in a way that allows a table determining
which queue a work item is in, rather than moving rows between tables, eg

```
id, reference, queue_name
```

Where `reference` is a foreign key that links to a larger table just called `work_items`
which contains the 7 columns I mentioned above. When a work item is moved from one queue
in the value stream to another, we simply update the `queue_name` in the respective row.

eg

```sql

UPDATE queues SET queue_name = 'develop' where id = 1 and reference = 12345;


```

You could then set up views that only filter rows for specific `queue_name` values.

You might then be able to develop a GUI or web interface that resembles a card wall typical
of Kanban Systems that is powered by this database back-end.

## Triggers

Though I am not personally familiar with databse triggers, you can start to see how it may
be possible to use the `wip_limits` table I discussed earlier to compare against row counts
in a given view for example (or count how mant items are in a given state within the`queues`
table to say, OK, we have a WIP limit of 3, but only two items in develop, so lets send a
notification that there is a free slot in the develop queue (3 - 2 = 1).

## Audit Logs?

Once a work item is complete, it makes no sence to keep it in the `queues` table, so one
might be tempted to remove the work item, and its place in the `queues` table. Which this
might seem like it solves the immediate problem of database growth, you'll lose all the rich
history that may be useful to reporting.

One solution might be to remove the entry from the `queues` table but leave it in `work_items`
but marked as completed in the queue column, and then have an audit log that logs the date
and state change of a work item, by reference, as it flows through teh work stream so that you
can see the history and report on whether it met SLA or not, such an audit table might look
like this (simplified idea at this stage):

```

+-----+------------+-----------+------------+
| id  | date       | reference | queue      |
+-----+------------+-----------+------------+
| 1   | 2025-01-01 |     12345 | backlog    |
...
| 27  | 2025-01-14 |     12345 | triage     |
...
| 317 | 2025-01-29 |     12345 | develop    |
...
| 444 | 2025-02-08 |     12345 | test       |
...
| 527 | 2025-02-15 |     12345 | uat        |
...
| 635 | 2025-04-02 |     12345 | production |

...

It is debatable whether you want a production queue, or simply use completed, that may imply
it has been delivered to production.

Petrhaps a driver for having separate production and completed queues, is to allow passage of
time to ensure that anything delevered to production with no defects reported within say 30 days
automatically gets marked as completed.

## Limiting WIP by imposing timeout limits of the backlog

The idea is that in a kanban pull-system, items from the backlog are triaged and prioritised for
work via prioritisation meetings that occur on a regular cadence, therefore, if an item hasnt been
prioritised for actionable work wihtin a set period of time, one might argue that it is not
important and therefore dropped form eth backlog. If this work becomes important in the future,
it can re-enter the backlog for prioritization. such a work flow might be represented in the audit
log as such:


```

+-----+------------+-----------+------------+
| id  | date       | reference | queue      |
+-----+------------+-----------+------------+
...
|   2 | 2025-01-01  |    12555 |    backlog | 
...
| 987 | 2025-04-01  |    12555 |       drop |
```
```

Without understanding triggers too deeply, one can see how a database trigger could automate this.

If a work item is dropped from the backlog after 90 days, it's creator can re-submit it at a later
date. The idea is that stakeholders attend the prioritisation meetings and/or state their cases
so that the business can make a decision on priorities based on urgency and need. Therefore one
can see that if a work item hasnt met a threshold for urgency or need, wiothin 90 days, it should
be pruned from the system.
 
