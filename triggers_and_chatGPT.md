# Caveats

Please note that all I have done here is tried to format the output with some markdown to aid readability. 

Most of this is Chat GPT wiffle using my previous doc as in input prompt. The big caveat here is that I have
not tested any of this code, it is a high-level CONCEPTUAL conversation, though ChatGPT did get a bit too
enthusiatic and started digging into the weeds a bit and producing code, I would strongly advise (either
future-me or you, dear reader) to apply a healthy dose af skepticism over some of this code, but I do hope
as is the desire of this repo, that you are able to pull out some ideas when implementing a software
based kanban system.

RUN CODE SNIPPETS AT YOUR OWN RISK, but feel free to pick the bones off this thing, I know I will be.

# The conversation



I dont know much about database triggers, but could you read this document and give me some ideas how such a trigger might be introduced:

https://github.com/richardforth/kanban/blob/main/database_driven_kanban_ideas.md


ChatGPT said:
Thought for 19s

I couldn’t load that GitHub page just now (looks like a rendering hiccup on GitHub’s side). If you can paste the text here I’ll tailor this exactly to what’s in your doc. In the meantime, here’s a concrete, battle-tested way to introduce database triggers into a Kanban-style app—plus ready-to-drop snippets for Postgres and MySQL/MariaDB.

## Where triggers shine in a Kanban app

 - Automatic timestamps & who-did-what
 - Ensure created_at, updated_at, created_by, updated_by are always set—no app-side gaps.
 - Write to an activity_log table on key changes (status, assignee, priority, due date).
 - Enforce valid status transitions (workflow guardrails)
 - Only allow transitions in your workflow graph (e.g., todo → doing → review → done).
 - Block illegal jumps (e.g., todo → done) unless a privileged user triggers a “fast-track.”

### WIP limits per column or assignee

Before moving a card into doing, count how many the user/team already has in that column; reject if exceeding the limit.

### Denormalized counters kept consistent

Maintain lists.wip_count, boards.card_count, users.open_card_count without relying on the app layer to remember to update them.

###SLA / due-date escalation hooks

 - When a card moves to doing, set started_at.
 - On update, if now() > due_at and status not in a “terminal” set, mark sla_breached = true and log it.

### Audit & compliance

Immutable append-only card_history rows for every significant update—who, when, what changed (old/new values).

## Notifications

Postgres: use NOTIFY/LISTEN so your worker picks up “card_moved” events instantly.

MySQL/MariaDB: insert into an outbox_events table for a background dispatcher.

## Ordering integrity

Keep position contiguous within a list (no duplicates or gaps) when cards are moved or deleted.

## Soft-delete & cascade cleanup

If you soft-delete a card, also soft-delete its checklists/comments/attachments; restore on undelete.

## Parent/child rollups

If subtasks change status, recompute the parent’s aggregated status/progress fields.

# Suggested schema bits (simplified)
```sql
-- cards
card(id PK, list_id FK, title, status, assignee_id, priority, due_at,
     created_at, created_by, updated_at, updated_by,
     started_at, completed_at, position, sla_breached boolean default false);

-- lists (columns)
list(id PK, board_id FK, name, wip_limit int, wip_count int default 0);

-- history
card_history(id PK, card_id FK, changed_at, changed_by,
             field_name, old_value, new_value);

-- outbox (optional for notifications)
outbox_events(id PK, topic, payload_json, created_at, processed_at null);
```

# PostgreSQL trigger examples
## 1) Set timestamps & basic activity log
```sql
CREATE OR REPLACE FUNCTION trg_cards_set_timestamps()
RETURNS trigger AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    NEW.created_at := COALESCE(NEW.created_at, now());
    NEW.updated_at := COALESCE(NEW.updated_at, now());
  ELSE
    NEW.updated_at := now();
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER cards_set_timestamps
BEFORE INSERT OR UPDATE ON card
FOR EACH ROW EXECUTE FUNCTION trg_cards_set_timestamps();
```
## 2) Valid status transitions
```sql
-- Valid transitions table
CREATE TABLE status_transition (
  from_status text, to_status text, PRIMARY KEY (from_status, to_status)
);

INSERT INTO status_transition VALUES
('todo','doing'),('doing','review'),('review','done'),
('doing','todo'),('review','doing'); -- allow backflow

CREATE OR REPLACE FUNCTION trg_cards_enforce_transition()
RETURNS trigger AS $$
DECLARE ok int;
BEGIN
  IF TG_OP = 'UPDATE' AND NEW.status IS DISTINCT FROM OLD.status THEN
    SELECT 1 INTO ok
    FROM status_transition
    WHERE from_status = OLD.status AND to_status = NEW.status;

    IF ok IS NULL THEN
      RAISE EXCEPTION 'Illegal status move: % -> %', OLD.status, NEW.status;
    END IF;

    -- Set lifecycle timestamps
    IF NEW.status = 'doing' AND OLD.status <> 'doing' AND NEW.started_at IS NULL THEN
      NEW.started_at := now();
    END IF;
    IF NEW.status IN ('done','cancelled') AND OLD.status NOT IN ('done','cancelled') THEN
      NEW.completed_at := now();
    END IF;

    -- SLA breach flag
    IF NEW.due_at IS NOT NULL AND NEW.completed_at IS NULL AND now() > NEW.due_at THEN
      NEW.sla_breached := true;
    END IF;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER cards_enforce_transition
BEFORE UPDATE ON card
FOR EACH ROW EXECUTE FUNCTION trg_cards_enforce_transition();
```
## 3) WIP limits & list counters
```sql
CREATE OR REPLACE FUNCTION trg_cards_wip_and_counters()
RETURNS trigger AS $$
DECLARE current_wip int;
BEGIN
  -- Moving into a list: enforce WIP
  IF (TG_OP = 'INSERT') OR (TG_OP = 'UPDATE' AND NEW.list_id <> OLD.list_id) THEN
    SELECT wip_count INTO current_wip FROM list WHERE id = NEW.list_id FOR UPDATE;
    PERFORM 1 FROM list WHERE id = NEW.list_id AND (wip_limit IS NULL OR current_wip < wip_limit);
    IF NOT FOUND THEN
      RAISE EXCEPTION 'WIP limit reached for list %', NEW.list_id;
    END IF;

    -- increment new list
    UPDATE list SET wip_count = wip_count + 1 WHERE id = NEW.list_id;
    -- decrement old list
    IF TG_OP = 'UPDATE' THEN
      UPDATE list SET wip_count = wip_count - 1 WHERE id = OLD.list_id;
    END IF;
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER cards_wip_and_counters
AFTER INSERT OR UPDATE ON card
FOR EACH ROW EXECUTE FUNCTION trg_cards_wip_and_counters();
```
## 4) Activity history + NOTIFY
```sql
CREATE OR REPLACE FUNCTION trg_cards_history_notify()
RETURNS trigger AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    INSERT INTO card_history(card_id, changed_at, changed_by, field_name, old_value, new_value)
    VALUES (NEW.id, now(), NEW.updated_by, 'create', NULL, NEW.title);
    PERFORM pg_notify('card_events', json_build_object(
      'event','created','card_id',NEW.id,'list_id',NEW.list_id)::text);
  ELSIF TG_OP = 'UPDATE' THEN
    IF NEW.status IS DISTINCT FROM OLD.status THEN
      INSERT INTO card_history(card_id, changed_at, changed_by, field_name, old_value, new_value)
      VALUES (NEW.id, now(), NEW.updated_by, 'status', OLD.status, NEW.status);
      PERFORM pg_notify('card_events', json_build_object(
        'event','status_changed','card_id',NEW.id,'from',OLD.status,'to',NEW.status)::text);
    END IF;
    IF NEW.assignee_id IS DISTINCT FROM OLD.assignee_id THEN
      INSERT INTO card_history(card_id, changed_at, changed_by, field_name, old_value, new_value)
      VALUES (NEW.id, now(), NEW.updated_by, 'assignee', OLD.assignee_id::text, NEW.assignee_id::text);
    END IF;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER cards_history_notify
AFTER INSERT OR UPDATE ON card
FOR EACH ROW EXECUTE FUNCTION trg_cards_history_notify();
```

# MySQL / MariaDB equivalents (condensed)

Note: No built-in NOTIFY/LISTEN. Use an outbox_events table and a worker.

```sql
DELIMITER //

CREATE TRIGGER cards_set_timestamps_bi
BEFORE INSERT ON card
FOR EACH ROW
BEGIN
  SET NEW.created_at = COALESCE(NEW.created_at, NOW());
  SET NEW.updated_at = COALESCE(NEW.updated_at, NOW());
END//

CREATE TRIGGER cards_set_timestamps_bu
BEFORE UPDATE ON card
FOR EACH ROW
BEGIN
  SET NEW.updated_at = NOW();
END//

CREATE TRIGGER cards_enforce_transition_bu
BEFORE UPDATE ON card
FOR EACH ROW
BEGIN
  IF NEW.status <> OLD.status THEN
    IF NOT EXISTS (
      SELECT 1 FROM status_transition
      WHERE from_status = OLD.status AND to_status = NEW.status
    ) THEN
      SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Illegal status transition';
    END IF;

    IF NEW.status = 'doing' AND OLD.status <> 'doing' AND NEW.started_at IS NULL THEN
      SET NEW.started_at = NOW();
    END IF;
    IF NEW.status IN ('done','cancelled') AND OLD.status NOT IN ('done','cancelled') THEN
      SET NEW.completed_at = NOW();
    END IF;
    IF NEW.due_at IS NOT NULL AND NEW.completed_at IS NULL AND NOW() > NEW.due_at THEN
      SET NEW.sla_breached = TRUE;
    END IF;
  END IF;
END//

CREATE TRIGGER cards_wip_and_counters_aiu
AFTER INSERT ON card
FOR EACH ROW
BEGIN
  UPDATE list SET wip_count = wip_count + 1 WHERE id = NEW.list_id;
END//

CREATE TRIGGER cards_wip_and_counters_au
AFTER UPDATE ON card
FOR EACH ROW
BEGIN
  IF NEW.list_id <> OLD.list_id THEN
    UPDATE list SET wip_count = wip_count + 1 WHERE id = NEW.list_id;
    UPDATE list SET wip_count = wip_count - 1 WHERE id = OLD.list_id;
  END IF;
END//

CREATE TRIGGER cards_history_au
AFTER UPDATE ON card
FOR EACH ROW
BEGIN
  IF NEW.status <> OLD.status THEN
    INSERT INTO card_history(card_id, changed_at, changed_by, field_name, old_value, new_value)
    VALUES (NEW.id, NOW(), NEW.updated_by, 'status', OLD.status, NEW.status);
    INSERT INTO outbox_events(topic, payload_json, created_at)
    VALUES ('card.status_changed',
            JSON_OBJECT('card_id', NEW.id, 'from', OLD.status, 'to', NEW.status),
            NOW());
  END IF;
END//

DELIMITER ;
```

# Introduce triggers safely (playbook)

 - Start with read-only triggers (history, counters, outbox). They’re low-risk.
 - Gatekeepers later (status transitions, WIP) once you’re confident you won’t block legitimate flows.
 - Feature-flag via tables, not code: e.g., a settings table with booleans like enforce_wip_limits.
 - Test with fixtures: simulate common scenarios—moving cards, bulk imports, reordering within/ across lists, soft-deletes, and rollbacks.
 - Idempotent workers (for outbox): ensure retries don’t duplicate notifications.
 - Document side-effects so app developers know what the DB will do for them.

## When to avoid triggers

Heavy business logic that changes often → consider app or stored procedures.

Cross-service side effects (send emails, call webhooks) → prefer outbox + worker, not direct trigger calls.

Bulk imports/migrations → consider SET session_replication_role = replica (Postgres) or disable triggers temporarily, then backfill counters/history.

If you drop the doc’s contents here, I’ll align these triggers precisely to your table/column names, workflows, and any rules you’ve already outlined.

You said:
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
database called kanban, we could have a table called wip_limits which would need
to be designed around the flow model for your specific value stream model, including
any buffers, so these are only ideas here, so lest assume we have 4 value streams
with a buffer for each (assuming you need input buffers), it might look like this:

(Note some buffers, such as the backlog (the initial input point for the value
stream) might need to be unlimited, so I have assigned a wip_limit here of -1,
representing unlimited as we do not want to affect thw rok streams of other parts
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
or has capacity to enable a pull from the triage queue (if implemented) or directly
from the backlog.

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

Where reference is a foreign key that links to a larger table just called work_items
which contains the 7 columns I mentioned above. When a work item is moved from one queue
in the value stream to another, we simply update the queue_name in the respective row.

eg


```sql

UPDATE queues SET queue_name = 'develop' where id = 1 and reference = 12345;
```

You could then set up views that only filter rows for specific queue_name values.

You might then be able to develop a GUI or web interface that resembles a card wall typical
of Kanban Systems that is powered by this database back-end.

## Triggers

Though I am not personally familiar with databse triggers, you can start to see how it may
be possible to use the wip_limits table I discussed earlier to compare against row counts
in a given view for example (or count how mant items are in a given state within thequeues
table to say, OK, we have a WIP limit of 3, but only two items in develop, so lets send a
notification that there is a free slot in the develop queue (3 - 2 = 1).

## Audit Logs?

Once a work item is complete, it makes no sence to keep it in the queues table, so one
might be tempted to remove the work item, and its place in the queues table. Which this
might seem like it solves the immediate problem of database growth, you'll lose all the rich
history that may be useful to reporting.

One solution might be to remove the entry from the queues table but leave it in work_items
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
```

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


Without understanding triggers too deeply, one can see how a database trigger could automate this.

If a work item is dropped from the backlog after 90 days, it's creator can re-submit it at a later
date. The idea is that stakeholders attend the prioritisation meetings and/or state their cases
so that the business can make a decision on priorities based on urgency and need. Therefore one
can see that if a work item hasnt met a threshold for urgency or need, wiothin 90 days, it should
be pruned from the system.
 
ChatGPT said:
Thought for 13s

Awesome—thanks for the write-up. You’re already 90% of the way there. Below is a compact, database-first way to add triggers that do exactly what you described: enforce/observe WIP, emit “free slot” notifications, keep an audit trail, and prune stale backlog items. I’ll show Postgres first (best ergonomics) and then note MySQL tweaks.

# Minimal schema to support your ideas

```
-- WIP limits (-1 means unlimited)
CREATE TABLE wip_limits (
  queue_name text PRIMARY KEY,
  wip_limit  int  NOT NULL  -- -1 => unlimited
);

-- Canonical work item data (your 7+ fields)
CREATE TABLE work_items (
  id              bigserial PRIMARY KEY,
  date_logged     timestamptz NOT NULL DEFAULT now(),
  date_due        timestamptz,
  past_due        boolean     GENERATED ALWAYS AS (date_due IS NOT NULL AND now() > date_due) STORED,
  reference       text UNIQUE NOT NULL,      -- e.g., ticket number
  short_desc      text NOT NULL,
  assignee_id     text,                      -- user id/email/etc
  dependency      text
);

-- Where each item is right now in the value stream
CREATE TABLE queues (
  id         bigserial PRIMARY KEY,
  reference  text NOT NULL REFERENCES work_items(reference) ON DELETE CASCADE,
  queue_name text NOT NULL REFERENCES wip_limits(queue_name),
  moved_at   timestamptz NOT NULL DEFAULT now()
);

-- Cheap, consistent counters (avoid SELECT COUNT(*) races)

CREATE TABLE queue_state (
  queue_name  text PRIMARY KEY REFERENCES wip_limits(queue_name),
  current_wip int  NOT NULL DEFAULT 0
);

-- Immutable history (append-only)

CREATE TABLE audit_log (
  id         bigserial PRIMARY KEY,
  occurred_at timestamptz NOT NULL DEFAULT now(),
  reference   text NOT NULL,
  from_queue  text,
  to_queue    text
);


-- Outbox for notifications (your app/worker reads & sends emails)
CREATE TABLE outbox_events (
  id         bigserial PRIMARY KEY,
  topic      text NOT NULL,
  payload    jsonb NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  processed_at timestamptz
);
```

# Seed your limits:

```
INSERT INTO wip_limits(queue_name, wip_limit) VALUES
('backlog', -1), ('triage', 3), ('development', 3), ('test', 3), ('uat', 5);

INSERT INTO queue_state(queue_name, current_wip)
SELECT queue_name, 0 FROM wip_limits
ON CONFLICT (queue_name) DO NOTHING;
```

# Triggers that make it work (PostgreSQL)
## 1) Enforce WIP on move + keep counters correct
```sql
CREATE OR REPLACE FUNCTION trg_queues_enforce_wip_and_count()
RETURNS trigger AS $$
DECLARE
  limit_new int;
  limit_old int;
BEGIN
  IF TG_OP = 'INSERT' THEN
    -- lock the new queue's counter row to prevent races
    SELECT w.wip_limit INTO limit_new
    FROM wip_limits w
    WHERE w.queue_name = NEW.queue_name
    FOR UPDATE;

    -- enforce limit if not unlimited
    IF limit_new <> -1 THEN
      UPDATE queue_state SET current_wip = current_wip + 1
      WHERE queue_name = NEW.queue_name
      RETURNING current_wip INTO limit_old; -- reuse var

      IF limit_old > limit_new THEN
        RAISE EXCEPTION 'WIP limit reached for % (limit %, attempted %)',
          NEW.queue_name, limit_new, limit_old;
      END IF;
    ELSE
      UPDATE queue_state SET current_wip = current_wip + 1
      WHERE queue_name = NEW.queue_name;
    END IF;

    RETURN NEW;
  ELSIF TG_OP = 'UPDATE' AND NEW.queue_name IS DISTINCT FROM OLD.queue_name THEN
    -- leaving old queue
    UPDATE queue_state SET current_wip = current_wip - 1
    WHERE queue_name = OLD.queue_name;

    -- entering new queue (same as insert branch)
    SELECT w.wip_limit INTO limit_new
    FROM wip_limits w
    WHERE w.queue_name = NEW.queue_name
    FOR UPDATE;

    IF limit_new <> -1 THEN
      UPDATE queue_state SET current_wip = current_wip + 1
      WHERE queue_name = NEW.queue_name
      RETURNING current_wip INTO limit_old;

      IF limit_old > limit_new THEN
        RAISE EXCEPTION 'WIP limit reached for % (limit %, attempted %)',
          NEW.queue_name, limit_new, limit_old;
      END IF;
    ELSE
      UPDATE queue_state SET current_wip = current_wip + 1
      WHERE queue_name = NEW.queue_name;
    END IF;

    RETURN NEW;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE queue_state SET current_wip = current_wip - 1
    WHERE queue_name = OLD.queue_name;
    RETURN OLD;
  END IF;

  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER queues_enforce_wip_and_count_biu
BEFORE INSERT OR UPDATE ON queues
FOR EACH ROW EXECUTE FUNCTION trg_queues_enforce_wip_and_count();

CREATE TRIGGER queues_count_bd
BEFORE DELETE ON queues
FOR EACH ROW EXECUTE FUNCTION trg_queues_enforce_wip_and_count();
```

Why BEFORE? So an over-limit move fails atomically, and counters never drift.

## 2) Emit a “free slot” event when capacity appears

We want to notify when a queue drops below its limit (after a delete or a move out). That happens when current_wip becomes < wip_limit. We capture that moment.

```sql
CREATE OR REPLACE FUNCTION trg_queue_state_slot_available()
RETURNS trigger AS $$
DECLARE
  lim int;
BEGIN
  -- Only fire when count decreases
  IF TG_OP = 'UPDATE' AND NEW.current_wip < OLD.current_wip THEN
    SELECT wip_limit INTO lim FROM wip_limits WHERE queue_name = NEW.queue_name;
    IF lim <> -1 AND NEW.current_wip < lim THEN
      INSERT INTO outbox_events(topic, payload)
      VALUES (
        'queue.slot_available',
        jsonb_build_object(
          'queue_name', NEW.queue_name,
          'free_slots', lim - NEW.current_wip,
          'at', now()
        )
      );
      -- Optional: Postgres pub/sub for instant workers
      PERFORM pg_notify('kanban_events',
        json_build_object('type','slot_available','queue',NEW.queue_name,'free',lim-NEW.current_wip)::text);
    END IF;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Fire whenever current_wip changes
CREATE TRIGGER queue_state_slot_available_au
AFTER UPDATE ON queue_state
FOR EACH ROW EXECUTE FUNCTION trg_queue_state_slot_available();
```

This gives you your “email the PM when a free slot appears” hook via an outbox (recommended) or LISTEN/NOTIFY. Your worker sends the actual email.

## 3) Audit log on every move
```
CREATE OR REPLACE FUNCTION trg_queues_audit()
RETURNS trigger AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    INSERT INTO audit_log(reference, from_queue, to_queue)
    VALUES (NEW.reference, NULL, NEW.queue_name);
  ELSIF TG_OP = 'UPDATE' AND NEW.queue_name IS DISTINCT FROM OLD.queue_name THEN
    INSERT INTO audit_log(reference, from_queue, to_queue)
    VALUES (NEW.reference, OLD.queue_name, NEW.queue_name);
  ELSIF TG_OP = 'DELETE' THEN
    INSERT INTO audit_log(reference, from_queue, to_queue)
    VALUES (OLD.reference, OLD.queue_name, NULL);
  END IF;
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER queues_audit_aiud
AFTER INSERT OR UPDATE OR DELETE ON queues
FOR EACH ROW EXECUTE FUNCTION trg_queues_audit();
```

Now you’ve got the immutable journey you showed in your example.

## 4) Backlog auto-drop after 90 days (pruning)

A pure trigger can’t run on a timer. Use a tiny scheduled job that turns findings into data changes, which then flow through the same audit/outbox triggers.

Postgres option (pg_cron extension):
```
-- A view to find “stale” backlog items
CREATE OR REPLACE VIEW backlog_stale AS
SELECT q.id, q.reference
FROM queues q
JOIN wip_limits w USING (queue_name)
WHERE q.queue_name = 'backlog'
  AND q.moved_at < now() - interval '90 days';

-- A stored procedure that moves stale backlog items to 'drop'
CREATE OR REPLACE PROCEDURE prune_stale_backlog()
LANGUAGE plpgsql AS $$
DECLARE r record;
BEGIN
  FOR r IN SELECT * FROM backlog_stale LOOP
    -- Move by updating queues row (fires all the same triggers)
    UPDATE queues SET queue_name = 'drop', moved_at = now() WHERE id = r.id;
  END LOOP;
END $$;

-- Schedule daily at 02:00
-- (once pg_cron is installed: CREATE EXTENSION pg_cron;)
SELECT cron.schedule('kanban-prune-backlog', '0 2 * * *', $$CALL prune_stale_backlog();$$);
```

If you can’t use pg_cron, run CALL prune_stale_backlog(); from any external scheduler (systemd/CRON/GitHub Actions)—same effect.

# How this answers each of your sections

 - WIP limits table: you’ve got wip_limits with -1 = unlimited.
 - Main queues table: you’ve got queues(reference, queue_name, moved_at). You never move rows between stage tables; you just change queue_name.

# Triggers:
 - Enforce limit on insert/move.
 - Maintain `queue_state.current_wip` accurately (no racing `COUNT(*)`).
 - Notify on capacity appearance via `outbox_events` (your mailer reads & emails the PM).

#Audit every move.
 - Backlog timeout: handled via a tiny scheduled procedure that reuses the same triggers (so audits and notifications still happen).
 - SLA/past due: an example computed column shows how to flag, and you can add similar rules in move triggers if you want to log SLA breaches.
 
# Quick MySQL / MariaDB notes

Replace PL/pgSQL with plain SQL triggers. No LISTEN/NOTIFY, so rely on `outbox_events` + a worker.

For a scheduler, use MySQL Event Scheduler:
```sql
CREATE EVENT prune_stale_backlog
ON SCHEDULE EVERY 1 DAY STARTS CURRENT_DATE + INTERVAL 2 HOUR
DO
  UPDATE queues q
  JOIN wip_limits w ON w.queue_name = q.queue_name
  SET q.queue_name = 'drop', q.moved_at = NOW()
  WHERE q.queue_name = 'backlog' AND q.moved_at < NOW() - INTERVAL 90 DAY;

```

MySQL lacks GENERATED ALWAYS AS (...) STORED with now() in expressions—use a trigger to set `past_due` on write or compute it in views/queries.

# Rollout plan (safe & incremental)

 - Create wip_limits, queue_state, and audit_log.
 - Add audit trigger (read-only, low risk).
 - Add counters trigger (no enforcement yet).
 - Flip on enforcement (reject over-limit moves).
 - Add slot-available trigger → connect a tiny worker that emails PMs (reads outbox_events).
 - Add the 90-day prune job (backlog hygiene).

If you’d like, paste your exact column names/types and DB (Postgres vs MySQL), and I’ll tailor the DDL/triggers line-for-line so you can run them as-is.

You said:
when you said:


This gives you your “email the PM when a free slot appears” hook via an outbox (recommended) or LISTEN/NOTIFY. Your worker sends the actual email.


Can you clarify what you mean by a worker? is it some notion of a process that monitors the outbox queue for events and then processes those envents as emails?
ChatGPT said:

Exactly 👍 — by “worker” I mean a background process (could be a service, daemon, or scheduled job) whose job is to watch for new rows in your outbox_events table (or LISTEN/NOTIFY channel) and then act on them.

Think of it as a bridge between the database world and the outside world (email, Slack, Teams, SMS, Jira updates, etc). Triggers are excellent at keeping database state correct, but they shouldn’t directly send emails/webhooks for a few reasons:

 - Portability: your DB doesn’t know how to talk SMTP/HTTP.
 - Stability: you don’t want an email server hiccup to block an INSERT into queues.
 - Retries: a worker can retry failed sends, back off, or mark errors for investigation.

# How it works in practice

Trigger inserts into outbox_events

```sql
INSERT INTO outbox_events(topic, payload)
VALUES ('queue.slot_available',
        jsonb_build_object('queue','development','free_slots',1));
```

Worker polls or listens

Polling: every few seconds, run

```sql
SELECT * FROM outbox_events WHERE processed_at IS NULL ORDER BY id LIMIT 10;
```

then process each event, send the email, and update processed_at.

Listening: in Postgres, you can LISTEN kanban_events; and the trigger’s pg_notify() will wake the worker instantly.

 - Worker sends emails
 - Reads event payload (topic = 'queue.slot_available').
 - Formats: “Development queue has 1 free slot available.”
 - Uses SMTP or an email API (SendGrid, SES, Postmark, etc).
 - Mark as processed

```sql
UPDATE outbox_events SET processed_at = now() WHERE id = $event_id;
```

# Worker technology options

## Simple cron + script:
A Python/Node/Ruby/Go script runs every minute, checks for new events, sends emails.

## Long-running daemon:
A service that connects to the DB, does LISTEN/NOTIFY (Postgres) or polling loop, and reacts instantly.

## Job queues:
Pipe DB events into a queue (RabbitMQ, Kafka, Sidekiq, Celery, etc) for horizontal scale and retries.

## Serverless:
Cloud functions (AWS Lambda, GCP Cloud Function, etc) triggered by DB change stream or polling.

👉 So in short:
The trigger guarantees that a “slot available” event is captured in the database reliably.
The worker is the external process that reads those events and turns them into emails.

Would you like me to sketch out a toy worker in Python that polls the `outbox_events` table and sends a dummy email (e.g. via SMTP)?

Yes please
ChatGPT said:
Thought for 18s

Totally—here’s a small, production-leaning Python worker that:

 - polls your outbox_events table,
 - safely claims events with FOR UPDATE SKIP LOCKED (so multiple workers won’t double-send),
 - sends an email via SMTP,
 - records success/failure with retry counts.

It’s written for PostgreSQL (via psycopg v3), but I note a MySQL tweak at the end.

# 1) Table hardening (recommended)
```sql
-- Add a couple of ops-friendly columns
ALTER TABLE outbox_events
  ADD COLUMN IF NOT EXISTS attempts int NOT NULL DEFAULT 0,
  ADD COLUMN IF NOT EXISTS last_error text;

-- Helpful index for fast picking of unprocessed events
CREATE INDEX IF NOT EXISTS outbox_events_unprocessed_idx
  ON outbox_events (created_at)
  WHERE processed_at IS NULL AND attempts < 10;

```
# 2) Minimal worker (Python)

Save as worker.py.

```python
import os
import json
import time
import smtplib
from email.message import EmailMessage
from contextlib import contextmanager

import psycopg
from psycopg.rows import dict_row

# ---- Config via env vars ----
DB_DSN         = os.getenv("DATABASE_URL", "postgresql://user:pass@localhost:5432/kanban")
SMTP_HOST      = os.getenv("SMTP_HOST", "localhost")
SMTP_PORT      = int(os.getenv("SMTP_PORT", "587"))
SMTP_USER      = os.getenv("SMTP_USER", "")
SMTP_PASS      = os.getenv("SMTP_PASS", "")
FROM_ADDR      = os.getenv("FROM_ADDR", "kanban-bot@example.com")
# Fallback recipient if not in payload
DEFAULT_TO     = os.getenv("DEFAULT_TO", "pm@example.com")

BATCH_SIZE     = int(os.getenv("BATCH_SIZE", "25"))
POLL_INTERVAL  = float(os.getenv("POLL_INTERVAL", "5"))   # seconds between polls
MAX_ATTEMPTS   = int(os.getenv("MAX_ATTEMPTS", "10"))

# ---- Email sending ----
def send_email(subject: str, body: str, to_addr: str):
    msg = EmailMessage()
    msg["Subject"] = subject
    msg["From"] = FROM_ADDR
    msg["To"] = to_addr
    msg.set_content(body)

    with smtplib.SMTP(SMTP_HOST, SMTP_PORT, timeout=30) as s:
        try:
            s.starttls()
        except smtplib.SMTPException:
            # Not all servers require/allow STARTTLS; silently continue
            pass
        if SMTP_USER:
            s.login(SMTP_USER, SMTP_PASS)
        s.send_message(msg)

# ---- DB helpers ----
@contextmanager
def db_conn():
    with psycopg.connect(DB_DSN) as conn:
        conn.execute("SET SESSION CHARACTERISTICS AS TRANSACTION READ WRITE, ISOLATION LEVEL READ COMMITTED;")
        yield conn

def fetch_and_lock_batch(conn):
    """
    Atomically claim a batch using SELECT ... FOR UPDATE SKIP LOCKED.
    Returns list of dict rows: id, topic, payload
    """
    with conn.cursor(row_factory=dict_row) as cur:
        cur.execute(
            f"""
            BEGIN;
            SELECT id, topic, payload
            FROM outbox_events
            WHERE processed_at IS NULL
              AND attempts < %s
            ORDER BY id
            FOR UPDATE SKIP LOCKED
            LIMIT %s;
            """,
            (MAX_ATTEMPTS, BATCH_SIZE),
        )
        rows = cur.fetchall()
        return rows

def mark_success(conn, event_id: int):
    with conn.cursor() as cur:
        cur.execute(
            "UPDATE outbox_events SET processed_at = now(), last_error = NULL WHERE id = %s;",
            (event_id,),
        )

def mark_failure(conn, event_id: int, err: str):
    with conn.cursor() as cur:
        cur.execute(
            "UPDATE outbox_events SET attempts = attempts + 1, last_error = %s WHERE id = %s;",
            (err[:4000], event_id),
        )

def commit(conn):
    conn.commit()

# ---- Event handling ----
def render_email_for_event(topic: str, payload: dict) -> tuple[str, str, str]:
    """
    Decide recipient/subject/body based on topic & payload.
    Payload comes from your trigger (jsonb).
    """
    to_addr = payload.get("notify_email") or DEFAULT_TO

    if topic == "queue.slot_available":
        queue = payload.get("queue_name") or payload.get("queue") or "unknown"
        free  = payload.get("free_slots", 1)
        subject = f"[Kanban] {queue} has {free} free slot(s)"
        body = (
            f"Good news!\n\n"
            f"The '{queue}' queue now has {free} free slot(s).\n"
            f"Consider pulling from triage/backlog.\n\n"
            f"Payload: {json.dumps(payload, indent=2)}\n"
        )
        return to_addr, subject, body

    # Default/unknown topic
    subject = f"[Kanban] Event: {topic}"
    body = f"Payload:\n{json.dumps(payload, indent=2)}\n"
    return to_addr, subject, body

# ---- Main loop ----
def run():
    print("Worker starting. Poll interval:", POLL_INTERVAL, "seconds")
    while True:
        claimed_any = False
        try:
            with db_conn() as conn:
                rows = fetch_and_lock_batch(conn)
                if not rows:
                    conn.execute("COMMIT;")  # end empty transaction cleanly
                for row in rows:
                    claimed_any = True
                    event_id = row["id"]
                    topic    = row["topic"]
                    payload  = row["payload"]

                    # psycopg returns JSON as dict automatically; if not, parse
                    if isinstance(payload, str):
                        payload = json.loads(payload)

                    try:
                        to_addr, subject, body = render_email_for_event(topic, payload)
                        send_email(subject, body, to_addr)
                        mark_success(conn, event_id)
                        print(f"Sent #{event_id} to {to_addr}: {subject}")
                    except Exception as e:
                        mark_failure(conn, event_id, repr(e))
                        print(f"Failed #{event_id}: {e!r}")

                commit(conn)

        except Exception as outer:
            # Log & keep going
            print("Loop error:", repr(outer))

        # Back off if nothing was claimed; otherwise loop immediately
        if not claimed_any:
            time.sleep(POLL_INTERVAL)

if __name__ == "__main__":
    run()
```

Install & run

```bash
python -m venv .venv
source .venv/bin/activate
pip install psycopg
# (optional) pip install python-dotenv  # if you prefer .env files

export DATABASE_URL="postgresql://user:pass@localhost:5432/kanban"
export SMTP_HOST="smtp.example.com"
export SMTP_PORT=587
export SMTP_USER="apikey"          # depends on your provider
export SMTP_PASS="secret"
export FROM_ADDR="kanban-bot@example.com"
export DEFAULT_TO="pm@example.com"

python worker.py
```

Scale-out: you can run multiple worker processes—FOR UPDATE SKIP LOCKED prevents duplicate work.

# 3) Hooking it to your trigger

When you insert into outbox_events, include the recipient if you want:

```sql
INSERT INTO outbox_events(topic, payload)
VALUES (
  'queue.slot_available',
  jsonb_build_object(
    'queue_name', NEW.queue_name,
    'free_slots', lim - NEW.current_wip,
    'notify_email', 'pm@example.com'
  )
);
```


(Otherwise the worker uses DEFAULT_TO.)

Optional niceties

LISTEN/NOTIFY for instant wakeups (Postgres): keep the polling loop, but also LISTEN kanban_events and do a short WAIT FOR NOTIFY before sleeping—best of both worlds.

Dead-letter: if attempts >= 10, treat as a DLQ row for manual inspection.

Per-topic handlers: map topics to functions (e.g., Slack, Teams, Webhooks).

Templating: use Jinja2 for pretty emails.

MySQL/MariaDB tweak

Replace JSONB with JSON; Python mysqlclient/mysql-connector instead of psycopg.

No LISTEN/NOTIFY, so just polling.

Row claiming: `SELECT ... FOR UPDATE` inside a transaction works; MySQL 8.0 also supports `SKIP LOCKED`:

```sql
SELECT id, topic, payload
FROM outbox_events
WHERE processed_at IS NULL AND attempts < 10
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 25;
```

If you want, tell me your DB (Postgres vs MySQL) and where you host email (SES/SendGrid/etc.), and I’ll tailor the script (auth, TLS, ports) and add a systemd unit or Dockerfile.

You said:
it would trypically be MySQL and the email MTA would be postfix on ubuntu linux
ChatGPT said:

Perfect—that’s a clean setup. Here’s a MySQL-flavoured version that works with a local Postfix MTA on Ubuntu (no SMTP auth/TLS needed; the worker just connects to localhost:25).

1) MySQL tables & indexes
```sql
-- Outbox for emails/notifications
CREATE TABLE IF NOT EXISTS outbox_events (
  id           BIGINT PRIMARY KEY AUTO_INCREMENT,
  topic        VARCHAR(128) NOT NULL,
  payload      JSON NOT NULL,
  created_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  processed_at TIMESTAMP NULL DEFAULT NULL,
  attempts     INT NOT NULL DEFAULT 0,
  last_error   TEXT NULL
) ENGINE=InnoDB;

-- Helpful covering index (MySQL has no partial indexes)
CREATE INDEX idx_outbox_unprocessed ON outbox_events (processed_at, attempts, id);

-- Example insert from your trigger or app:
-- INSERT INTO outbox_events(topic, payload)
-- VALUES ('queue.slot_available', JSON_OBJECT('queue_name','development','free_slots',1,'notify_email','pm@example.com'));
```

If you already have this table from earlier examples, just ALTER TABLE to add attempts, last_error, and the index.

2) Python worker for MySQL + local Postfix

Uses mysql-connector-python

Safely claims rows with FOR UPDATE SKIP LOCKED (MySQL 8.0+)

Sends mail via Postfix listening on localhost:25 (default on Ubuntu)

worker.py:

```python
import os
import json
import time
import smtplib
from email.message import EmailMessage

import mysql.connector
from mysql.connector import errorcode

# ---- Config via env ----
DB_HOST      = os.getenv("DB_HOST", "127.0.0.1")
DB_PORT      = int(os.getenv("DB_PORT", "3306"))
DB_USER      = os.getenv("DB_USER", "kanban")
DB_PASS      = os.getenv("DB_PASS", "secret")
DB_NAME      = os.getenv("DB_NAME", "kanban")

BATCH_SIZE   = int(os.getenv("BATCH_SIZE", "25"))
POLL_SECS    = float(os.getenv("POLL_SECS", "5"))
MAX_ATTEMPTS = int(os.getenv("MAX_ATTEMPTS", "10"))

FROM_ADDR    = os.getenv("FROM_ADDR", "kanban-bot@example.com")
DEFAULT_TO   = os.getenv("DEFAULT_TO", "pm@example.com")

SMTP_HOST    = os.getenv("SMTP_HOST", "127.0.0.1")  # Postfix local MTA
SMTP_PORT    = int(os.getenv("SMTP_PORT", "25"))    # No TLS/auth

def db_connect():
    return mysql.connector.connect(
        host=DB_HOST, port=DB_PORT, user=DB_USER, password=DB_PASS,
        database=DB_NAME, autocommit=False
    )

def send_email(subject: str, body: str, to_addr: str):
    msg = EmailMessage()
    msg["Subject"] = subject
    msg["From"] = FROM_ADDR
    msg["To"] = to_addr
    msg.set_content(body)
    with smtplib.SMTP(SMTP_HOST, SMTP_PORT, timeout=30) as s:
        # local Postfix: no STARTTLS/login
        s.send_message(msg)

def render_email(topic: str, payload: dict):
    to_addr = payload.get("notify_email") or DEFAULT_TO
    if topic == "queue.slot_available":
        queue = payload.get("queue_name") or payload.get("queue") or "unknown"
        free  = payload.get("free_slots", 1)
        subject = f"[Kanban] {queue} has {free} free slot(s)"
        body = (
            f"The '{queue}' queue now has {free} free slot(s).\n"
            f"Consider pulling from triage/backlog.\n\n"
            f"Payload:\n{json.dumps(payload, indent=2)}\n"
        )
        return to_addr, subject, body
    # fallback
    subject = f"[Kanban] Event: {topic}"
    body = f"Payload:\n{json.dumps(payload, indent=2)}\n"
    return to_addr, subject, body

def claim_batch(conn):
    """
    Claim a batch atomically so multiple workers don't duplicate work.
    Requires MySQL 8.0+ for SKIP LOCKED.
    """
    cur = conn.cursor(dictionary=True)
    # Start a tx
    cur.execute("SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED")
    conn.start_transaction()
    cur.execute(
        f"""
        SELECT id, topic, payload
        FROM outbox_events
        WHERE processed_at IS NULL
          AND attempts < %s
        ORDER BY id
        FOR UPDATE SKIP LOCKED
        LIMIT %s
        """,
        (MAX_ATTEMPTS, BATCH_SIZE),
    )
    rows = cur.fetchall()
    return cur, rows  # keep cursor/tx open until we finish processing

def mark_success(conn, cur, event_id: int):
    cur.execute("UPDATE outbox_events SET processed_at = NOW(), last_error = NULL WHERE id = %s", (event_id,))

def mark_failure(conn, cur, event_id: int, err: str):
    cur.execute(
        "UPDATE outbox_events SET attempts = attempts + 1, last_error = %s WHERE id = %s",
        (err[:4000], event_id),
    )

def main_loop():
    print("MySQL worker started; polling every", POLL_SECS, "seconds")
    while True:
        claimed_any = False
        try:
            conn = db_connect()
            cur, rows = claim_batch(conn)
            if rows:
                claimed_any = True
            for row in rows:
                event_id = row["id"]
                topic    = row["topic"]
                payload  = row["payload"]
                # mysql-connector returns JSON as str -> parse
                if isinstance(payload, str):
                    try:
                        payload = json.loads(payload)
                    except Exception:
                        payload = {"raw": payload}

                try:
                    to_addr, subject, body = render_email(topic, payload)
                    send_email(subject, body, to_addr)
                    mark_success(conn, cur, event_id)
                    print(f"Sent #{event_id} to {to_addr}: {subject}")
                except Exception as e:
                    mark_failure(conn, cur, event_id, repr(e))
                    print(f"Failed #{event_id}: {e!r}")

            # Commit the whole batch (marks & attempts)
            conn.commit()
            cur.close()
            conn.close()
        except mysql.connector.Error as db_err:
            print("DB error:", db_err)
        except Exception as outer:
            print("Loop error:", repr(outer))

        if not claimed_any:
            time.sleep(POLL_SECS)

if __name__ == "__main__":
    main_loop()

```
requirements.txt:

```
mysql-connector-python>=8.3
```

Run it:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

export DB_HOST=127.0.0.1 DB_PORT=3306 DB_USER=kanban DB_PASS=secret DB_NAME=kanban
export FROM_ADDR="kanban-bot@example.com" DEFAULT_TO="pm@example.com"
# With local Postfix, SMTP_HOST/PORT defaults are fine
python worker.py
```

MySQL < 8.0 fallback: if you can’t use SKIP LOCKED, replace the claim with a “claim flag” pattern:

Add claimed_at TIMESTAMP NULL

```
UPDATE outbox_events SET claimed_at = NOW() WHERE processed_at IS NULL AND attempts < 10 AND claimed_at IS NULL ORDER BY id LIMIT 25;
```
Then `SELECT * FROM outbox_events WHERE claimed_at = NOW() AND ... 
```
(use a unique token rather than NOW() for robustness).

3) MySQL Event Scheduler (optional) for backlog pruning

Enable it:

```sql
SET GLOBAL event_scheduler = ON;
-- Or in /etc/mysql/mysql.conf.d/mysqld.cnf:
-- event_scheduler=ON


Example daily prune job (adjust to your schema):

CREATE EVENT IF NOT EXISTS prune_stale_backlog
ON SCHEDULE EVERY 1 DAY STARTS CURRENT_DATE + INTERVAL 2 HOUR
DO
  UPDATE queues q
  JOIN wip_limits w ON w.queue_name = q.queue_name
  SET q.queue_name = 'drop', q.moved_at = NOW()
  WHERE q.queue_name = 'backlog'
    AND q.moved_at < NOW() - INTERVAL 90 DAY;
```

This will in turn fire your other triggers (audit, counters, etc.).

4) Postfix on Ubuntu: quick, safe defaults

If you just want the worker to hand mail to a local MTA and let Postfix relay (or deliver locally):

```bash
sudo apt-get update
sudo apt-get install -y postfix
# Choose "Internet Site" (or "Local only" if you don't relay externally)
```

Minimal knobs in /etc/postfix/main.cf:

```
myhostname = kanban.local
myorigin = /etc/mailname
inet_interfaces = loopback-only      # only accept from local host
inet_protocols = ipv4
```

If you need to relay all mail via an upstream smart host (e.g., your company SMTP):

```
relayhost = [smtp.yourdomain.tld]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_use_tls = yes
```

Then:

```bash
echo "[smtp.yourdomain.tld]:587 USER:PASS" | sudo tee /etc/postfix/sasl_passwd
sudo postmap /etc/postfix/sasl_passwd
sudo systemctl restart postfix
sudo chmod 600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
```

Test from the worker host:

```bash
python - <<'PY'
import smtplib
from email.message import EmailMessage
m=EmailMessage(); m['From']='kanban-bot@example.com'; m['To']='you@example.com'; m['Subject']='Postfix test'; m.set_content('hi')
s=smtplib.SMTP('127.0.0.1',25,timeout=10); s.send_message(m); s.quit(); print("sent")
PY
```
5) systemd unit (run the worker as a service)

/etc/systemd/system/kanban-outbox-worker.service:

```ini
[Unit]
Description=Kanban Outbox Worker
After=network-online.target mysql.service postfix.service
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/opt/kanban-worker
ExecStart=/opt/kanban-worker/.venv/bin/python /opt/kanban-worker/worker.py
Restart=always
RestartSec=3
Environment=DB_HOST=127.0.0.1
Environment=DB_PORT=3306
Environment=DB_USER=kanban
Environment=DB_PASS=secret
Environment=DB_NAME=kanban
Environment=FROM_ADDR=kanban-bot@example.com
Environment=DEFAULT_TO=pm@example.com
# Optional tuning
Environment=BATCH_SIZE=50
Environment=POLL_SECS=3

[Install]
WantedBy=multi-user.target
```

Then:

```bash
sudo mkdir -p /opt/kanban-worker
sudo cp worker.py /opt/kanban-worker/
python -m venv /opt/kanban-worker/.venv
/opt/kanban-worker/.venv/bin/pip install mysql-connector-python
sudo systemctl daemon-reload
sudo systemctl enable --now kanban-outbox-worker
sudo systemctl status kanban-outbox-worker --no-pager
```

That’s it

Triggers write to outbox_events.

The MySQL worker safely claims and sends emails via local Postfix.

Optional MySQL Event prunes backlog.

systemd keeps it running.

If you want, tell me your exact MySQL version and I can switch the claiming logic to the best pattern for that version (SKIP LOCKED vs claim flag).

You said:
Percona, 84?
ChatGPT said:
Thought for 18s

Got it—you mean Percona Server for MySQL 8.4. Great choice. Short answer:

Yes, Percona 8.4 supports SELECT … FOR UPDATE SKIP LOCKED, JSON columns, and the MySQL Event Scheduler.

So the worker + outbox pattern I gave will work as-is (with the MySQL version of the worker).

Here’s what to know/tweak for Percona 8.4:

Verify features quickly
```sql
SELECT VERSION();                       -- should show 8.4.x-xx Percona
SHOW VARIABLES LIKE 'event_scheduler';  -- ON to use events
```
Row-claiming (safe for multiple workers)

Keep the claim query exactly like this (requires 8.0+ / 8.4):

```sql
SELECT id, topic, payload
FROM outbox_events
WHERE processed_at IS NULL
  AND attempts < 10
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 25;
```

Run it inside a transaction; update processed_at/attempts in the same txn, then COMMIT.

Default isolation (REPEATABLE READ) is fine; if you prefer, set READ COMMITTED per session:

```sql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

JSON & constraints

You can keep payload JSON. If you want a hard guard:

```sql
ALTER TABLE outbox_events
  ADD CONSTRAINT chk_payload_json CHECK (json_valid(payload));
```
Eve
nt Scheduler (for the 90-day prune)

Enable once:

```sql
SET GLOBAL event_scheduler = ON;
-- Or put event_scheduler=ON in mysqld config.
```

Job example (fires your triggers too):

```sql
CREATE EVENT IF NOT EXISTS prune_stale_backlog
ON SCHEDULE EVERY 1 DAY STARTS CURRENT_DATE + INTERVAL 2 HOUR
DO
  UPDATE queues q
  JOIN wip_limits w ON w.queue_name = q.queue_name
  SET q.queue_name = 'drop', q.moved_at = NOW()
  WHERE q.queue_name = 'backlog'
    AND q.moved_at < NOW() - INTERVAL 90 DAY;
```

If you’re on Percona XtraDB Cluster (PXC) 8.4 (Galera)

This still works, but a couple of tips:

Keep batches small (LIMIT 25 is good) and transactions short—cluster replication dislikes long txns.

Avoid using the Event Scheduler for heavy jobs on every node; run it on one designated node or do pruning from an external cron script to reduce cluster churn.

Worker recap (no change needed)

The MySQL worker I gave (with mysql-connector-python) + Postfix on localhost:25 is good to go.

You can run multiple workers; SKIP LOCKED prevents double-sends.

If you want, tell me whether you’re on standalone Percona 8.4 or PXC 8.4 and I’ll drop in a tiny config snippet (systemd + my.cnf knobs) optimized for that setup.
