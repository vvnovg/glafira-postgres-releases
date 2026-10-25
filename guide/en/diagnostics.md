# Diagnostics

The **Diagnostics** group sits in the tree of an expanded connection, after the groups with nodes, databases and roles. Each tool opens in its own tab: **Pulse**, **Sessions**, **Idle in transaction**, **Wait profile**, **Locks**, **xmin horizon**, **Replicas** and **Replication slots**; a Patroni cluster also has **Failover readiness** and **Health matrix**. The tools themselves only read: the app runs `SELECT` on PostgreSQL nodes and `GET` on the Patroni REST API. Only actions change the server or the cluster — separate buttons with a preview and a confirmation, see [Actions](diagnostics.md#actions) and [Patroni actions](diagnostics.md#patroni-actions); the last item of the group is **Actions journal** (the [actions journal](diagnostics.md#journal)). Diagnostic sessions are read-only, with `statement_timeout` of 5 seconds and `lock_timeout` of 1 second, and are named `glafira-pg` (`application_name`).

The header is the same in every tab:

- **Node** — which node to read. The default is **Follow the leader**: the node that accepts writes — the Patroni leader (the leader of a standby cluster in that case), the available primary of a host list, or the only server. When the leader changes, the tab moves to the new one and says “Node changed: old → new”. A node chosen from the list stays pinned to the tab. Tools that poll all members themselves (**Pulse**, **Failover readiness**, **Health matrix**) and a single server have no node selector.
- **Interval** — how often to refresh the data: 2, 5, 10 or 30 seconds, or **Pause**. The default is 5 seconds; the Patroni tools are polled every 10 seconds, like the topology. The choice is not kept between launches.
- **Refresh** reads the data right away; next to it is the time when the app received the last report. If a refresh fails, the previous data stays on screen, marked **stale**, with an error message.

A tab polls the server only while it is visible: a hidden tab sends nothing, and when you return to it, data that went stale is read at once. After a login is rejected, polling stops: repeated attempts with a wrong password can lock the account. If the connection is closed, the tab says **The connection is closed** and offers **Connect**.

For the full picture the role needs `pg_monitor` (it includes `pg_read_all_stats`): without it the server hides the query texts of other sessions, replica details and the size of `pg_wal`. The tool says so: the hidden part is listed as grey lines under the findings block, for example “pg_wal size: needs the pg_monitor role”. What the server version lacks is explained the same way, for example “Query ID (query_id): PostgreSQL 14+”. Privileges and the load on the server are described in the administrator guide, in the “Diagnostics” section.

<a id="pulse"></a>

## Pulse

**Pulse** is one table for the whole cluster: a row per node, a column per indicator. All available nodes are polled in parallel: for each one, a query for the indicators and a separate query for the size of `pg_wal` (`pg_ls_waldir()`). An unavailable node stays a row with the reason (**Node unavailable**).

Indicators:

- **Connections** — how many of the connections available to ordinary roles are in use (`max_connections` minus the superuser reserve and, in PostgreSQL 16 and later, minus `reserved_connections`), and the percentage.
- **TPS** — transactions per second; hover to see commits and rollbacks separately.
- **Cache hit ratio** — the share of block reads served from the cache between two refreshes.
- **WAL per second** — the WAL write rate (receive rate on a replica).
- **Longest transaction**, **Longest query** and **Longest idle in transaction** — the longest transaction, active query and idle-in-transaction time among client sessions. The server computes durations from its own `now()`, the clock of your computer is not used, and the collecting session itself is not counted.
- **Waiting for locks · longest** — how many client sessions wait for a lock and how long the oldest of them has waited. Background processes such as autovacuum are not counted here: their waits are shown in **Locks**.
- **XID age** — the largest `datfrozenxid` age across databases and its share of the wraparound limit (2³¹).
- **pg\_wal size** — the size of the `pg_wal` directory; hover to see `max_wal_size`.
- **Replica lag** — replica lag in bytes and in time. It comes from the topology; there is no separate query to the replica.

A cell is colored by the most severe finding of the node for that indicator. Color is not the only cue: the cell has a mark (`i`, `!` or `!!`) and a hidden label for screen readers. Clicking a cell opens the tool where the culprits are listed, straight on that node: connections, queries, transactions and cache — **Sessions**; idle in transaction — **Idle in transaction**; TPS and lock waits — **Locks**; XID age — **xmin horizon**; WAL rate and lag — **Replicas**; `pg_wal` size — **Replication slots**.

The rates (TPS, cache hit, WAL per second) are the difference between two consecutive refreshes, so the first refresh shows “…” instead. The same happens after a server restart or a statistics reset: there is nothing to compare with. The size of `pg_wal` is read by `pg_ls_waldir()`, which needs the `pg_monitor` role; without it the column shows a dash and the reason is given under the findings.

<a id="sessions"></a>

## Sessions and idle in transaction

**Sessions** shows `pg_stat_activity` of the chosen node: a row per session, virtualized for large lists. **Idle in transaction** is the same list with the “Idle in transaction” preset selected in advance.

- Presets: **All**, **Active**, **Idle in transaction** and **Waiting** (waiting for a lock). Active and waiting sessions are sorted by query age, idle in transaction by idle time.
- The filter is a case-insensitive substring of pid, role, application, client, database and query text.
- **Group by** — application, role, client or database: instead of the list you get a table with session counts per state. The preset does not affect the groups, but the filter and **Hide Glafira sessions** do.
- **Hide Glafira sessions** is on by default: the app's own sessions (`application_name = glafira-pg`) are shown neither in the list nor in the groups.
- The **Waits for** column holds the pids of the blocking sessions; it is filled only for sessions waiting for a lock.
- Clicking a row opens the session panel: backend type, start time, age of the transaction, query and state, transaction ID, xmin age, query ID (PostgreSQL 14 and later), the number of locks held, the full query text and the **Copy query** button. The text is limited to 16 KB, and a cut is marked. When the session ends, the panel stays with the last data and a note. On a connection that is not read-only, the panel of a client session has the **Cancel query…** and **Terminate session…** buttons (see [Actions](diagnostics.md#actions)).

The query texts of other sessions are visible to a role with `pg_read_all_stats`. Without it the text is `<insufficient privilege>`, and the “Texts of sessions are hidden” finding suggests granting `pg_monitor` to the role.

A session in the `idle in transaction` state opened a transaction and does nothing. It holds the locks it has acquired (the **Locks held** column) and the xmin horizon (the **xmin age** column): autovacuum cannot remove dead rows newer than its snapshot, and tables bloat. Usually the application is at fault — it did not run `COMMIT` or `ROLLBACK`; grouping by application and role helps to find it. Server-side protection is `idle_in_transaction_session_timeout`: set it for the application role rather than for the whole server. The “Idle in transaction” preset shows the current value of the setting. If it is 0 and a session has been idle for 5 minutes or more, a finding suggests a ready command, `ALTER ROLE … SET idle_in_transaction_session_timeout = '10min';`, with the **Copy command** button; the app does not run this command.

Sessions that have been idle in a transaction for longer than a threshold can be terminated as a batch: above the list of the “Idle in transaction” preset there is an **Older than, min** field and a terminate button (see [Actions](diagnostics.md#actions)).

<a id="waits"></a>

## Wait profile

**Wait profile** answers the question “what do active sessions spend their time on”. It is built from samples collected by the sampler:

- Once a second, one light query to `pg_stat_activity` runs on the chosen node: active sessions of clients, autovacuum and parallel workers. Background processes that wait forever (walsender, checkpointer and the like) are not sampled so they do not distort the picture; the sampler's own session is not sampled either. Query text is taken up to 16 KB.
- The sampler has its own connection (one per node), so a poll every second never waits for catalog queries.
- The sampler runs while the profile tab is open, including while the tab is hidden: history is not interrupted when you switch tabs. Closing the tab stops the sampler. A tab that follows the leader switches to the sampler of the new leader when the leader changes.
- History is kept only in the memory of the app: the last 60 minutes, but not more than 500,000 sample rows (about 40 MB); older data is dropped. It lives until the connection is closed or the node leaves the cluster. If you close the tab and open it again, the history continues, and the time when the sampler did not run is marked as a gap (hatched on the chart).
- A query that has not finished within a second does not overlap with the next one: the tick is skipped. After an error the sampler pauses for 1, 2, 5, 10 and 30 seconds; after a rejected login it stops, and it starts again when you choose **Reconnect** in the connection menu.

**Window** is chosen in the header: 5, 15 or 60 minutes (15 by default). The chart is made of columns with the average number of active sessions per wait class: one column covers 5 seconds for a 5-minute window, 10 seconds for 15 minutes and 40 seconds for 60 minutes. The legend and the table columns list only the classes found in the window; a table with the same numbers is given for screen readers. If the history is shorter than the window, the tab says since when the sampler has been running. The line above the chart shows that the sampler is alive, with the number of active sessions and of sessions waiting for a lock at the last tick.

Wait classes are `wait_event_type` values. An active session with no wait event is counted as the `CPU*` class: it is on the CPU or waits where PostgreSQL does not instrument the wait, hence the asterisk. The other classes:

- `Lock` — waiting for a heavyweight lock; **Locks** shows who blocks.
- `LWLock` — internal lightweight locks of the server, contention for shared structures.
- `IO` — reading and writing data files and WAL.
- `IPC` — waiting for another process, for example a parallel worker or replication.
- `Client` — waiting for the client: the session waits for data from the application or sends a result.
- `Timeout` — waiting for a timer, for example `pg_sleep`.
- `BufferPin`, `Extension`, `Activity` and others.

Below the chart are top waits (class and wait event, share and number of samples), top queries and breakdowns by application, role, database and client; each table has up to 10 rows. A query in the top is identified by `query_id` (PostgreSQL 14 and later, when it exists and is not 0), otherwise by its text; in PostgreSQL 13 always by text. Parallel workers are counted under the query of their leader. For a query its main wait is shown.

The finding about lock waits fires when `Lock` accounts for 20% of the window's samples or more (50% is critical); it is computed only if the window holds at least 30 samples. The profile report reads the buffer in memory and does not query the server itself: the load comes only from the sampler.

<a id="locks"></a>

## Locks

**Locks** shows who holds whom. The data comes from `pg_stat_activity`, `pg_locks` and `pg_blocking_pids`; the list of blockers is requested only for sessions waiting for a lock, and relation sizes are not computed, so the tool puts no lock load on the server.

- The **Lock tree** starts from the root blocker — a session that waits for nothing itself but holds others. The victims are nested under it, and under them those who queue behind the victims. Each node shows the user, application, database, state, transaction age, query, the number of victims in the subtree, and for a waiting session what it requested (lock mode on a relation) and how long it has waited. A victim's parent is the first blocker from `pg_blocking_pids`; the others are listed in the “also waits for” line. Branches collapse.
- If the blocker is not visible — a prepared transaction (pid 0) or a session that ended between queries — the waiting session becomes the root of its own subtree, and the caption explains what it waits for. Look for prepared transactions in **xmin horizon**.
- PostgreSQL 13 does not record when a lock wait began (`pg_locks.waitstart` exists since version 14), so there the wait is estimated from the last state change of the session and marked “≈”.
- A lock on a relation from another database is labeled “oid N in database X”: the name can be read only in the database of the connection.

Above the tree are the **DDL in the queue** cards. This is DDL that waits for a relation lock and holds others back itself: `ALTER`, `DROP`, `TRUNCATE`, `LOCK`, `CREATE TRIGGER`, foreign keys and `CREATE INDEX` without `CONCURRENTLY` ask for `AccessExclusiveLock`, `ShareRowExclusiveLock` or `ShareLock`. This is the most common reason for “everything stopped”: a migration waits for a long transaction, and ordinary `SELECT`s line up behind the migration. A card shows the DDL, whom it waits for, for how long, and how many sessions stand behind it. What to do: cancel the DDL — the queue clears at once — or the query the DDL waits for; run migrations with `lock_timeout` so that the DDL gives up by itself. Cancel a query from the session panel in **Sessions**, and terminate the root blocker with the **Terminate the root blocker…** button in the tree (the `pg_signal_backend` role, or the same role as the session; see [Actions](diagnostics.md#actions)).

The **Wait cycles** section lists sessions that wait for each other. The server resolves such a cycle itself after `deadlock_timeout`; the details of deadlocks the server has already resolved exist only in its log, and a note about this always stands under the tab.

Findings: a lock wait of 30 seconds or more (critical from 5 minutes); a root blocker whose victim waits 10 seconds or more (critical from 60 seconds or with 10 or more victims; one finding per root); queued DDL (critical, one finding per item of the queue); a wait cycle (info).

<a id="horizon"></a>

## xmin horizon

The xmin horizon is the oldest transaction whose snapshot is still needed by someone on the server. `VACUUM` cannot remove dead rows newer than the horizon, so a long-lived holder bloats tables and grows the XID age. **xmin horizon** lists the holders by descending xmin age (in transactions) and shows how long each has held it:

- a session with an open transaction — pid, role, application, database, state; it helps to find the application and close the transaction;
- a prepared transaction — one forgotten after a coordinator failure; below it is a ready command `ROLLBACK PREPARED '…';` with the **Copy command** button (the app does not run it);
- a replication slot — an inactive slot holds `xmin` or `catalog_xmin` (the catalog horizon is marked separately); fix it in **Replication slots**;
- replica feedback: with `hot_standby_feedback = on` a long query on a replica holds the horizon on the primary. The replica is matched to a cluster member, and below its row is a link to the horizon of that member: fix it there, not on the primary. A replica connected through a physical slot is shown the same way.

Autovacuum and lazy `VACUUM` (including a manual one) are not counted as holders: their xmin does not hold back the horizon of other processes; `VACUUM FULL` is an ordinary transaction and is counted. The value of `hot_standby_feedback` is shown above the table. Without `pg_read_all_stats` the server hides the xmin of other sessions — this is stated under the findings.

The “The xmin horizon is held by …” finding describes the oldest holder: 10 million transactions or 1 hour is a warning, 100 million or 6 hours is critical. The XID age of the database itself is in **Pulse**.

<a id="replication"></a>

## Replicas

**Replicas** shows replication from the point of view of the chosen node.

On a primary and on a cascading replica this is the **WAL senders** table — a row per connected replica. A replica is matched to a cluster member by `application_name`: it equals the member name or its host (names are not resolved through DNS). If it cannot be matched, the row is labeled with `application_name` and there are no links to a member. The sent, write, flush and replay lags are shown in bytes from the current WAL position of the node, the replay lag also in time; the write and flush times are on hover. Besides the lags, the table has the client address, state, sync state, time since the replica's last reply, slot and xmin age. Above the table are `synchronous_standby_names` and `synchronous_commit`.

On a replica, the **WAL receiver** card stands at the top: status and source, slot, timeline, time of the last message and of the last position report, the gap between received and applied WAL, replay delay, WAL replay pause state, query conflicts, `hot_standby_feedback` and `max_standby_streaming_delay`. Recovery conflicts — queries cancelled on the replica to apply WAL — are summed over databases; next to the total is the growth since the previous refresh.

A role without `pg_read_all_stats` cannot see the state of senders and of the receiver. Then, instead of the table, only the number of connected WAL senders is shown, the view says so, and no findings are built from them: empty fields do not mean that replication is broken.

Findings: replica lag of 16 MB or 30 seconds (critical from 1 GB or 5 minutes); a replica that is not streaming; a replica silent for more than 60 seconds; `synchronous_standby_names` is set but there is no synchronous replica (critical); the WAL receiver is not running (critical); WAL replay is paused — if the pause is not intentional, resume it with `pg_wal_replay_resume()`; new recovery conflicts (consider `hot_standby_feedback` or `max_standby_streaming_delay`).

<a id="slots"></a>

## Replication slots

**Replication slots** shows the replication slots of the chosen node: name, type (physical or logical, plugin), database, state, `wal_status`, retained WAL, WAL headroom before the slot is lost (with `max_slot_wal_keep_size`), xmin and `catalog_xmin` age, the moment since which the slot has been inactive (PostgreSQL 17 and later), and flags: temporary, failover, synced from the primary, two-phase, conflicting, invalidated. Columns the server version lacks are empty, and the reason is given under the findings. Below the table is the value of `max_slot_wal_keep_size`.

A slot keeps WAL until its consumer has read it. An inactive slot whose consumer is gone piles up WAL, and `pg_wal` can fill the disk. The slot keeps the WAL it needs while `wal_status` is `reserved` or `extended`; `unreserved` means that the WAL is about to be removed, and `lost` that it is gone and the consumer cannot continue: drop the slot and resynchronize the consumer. If the consumer is gone for good, drop the slot (the **Drop…** button in the table, see [Actions](diagnostics.md#actions); only an inactive slot can be dropped); otherwise bring the consumer back and set `max_slot_wal_keep_size`. An active slot piles up WAL while it is being read — that is replica lag, not a forgotten slot; synced slots (PostgreSQL 17 and later, on a standby) are inactive by design and raise no findings.

For a Patroni cluster there is an extra column with the slot mark: a member slot — Patroni creates and removes it itself — or a permanent slot from the `slots` section of the dynamic configuration (`GET /config`). The name of a member slot is built by the Patroni rule: lower case, `-` and `.` become `_`, other characters outside `[a-z0-9_]` become `u` and a four-digit character code, at most 63 characters long. If `/config` could not be read, only member slots are marked, and the reason is given under the findings.

Below the table are the logical replication subscriptions: the apply worker (or “not running”), the received LSN, the time of the last message, the numbers of apply and sync errors (PostgreSQL 15 and later) and their growth since the previous refresh.

Findings: an inactive slot retains 1 GB of WAL or more (critical from 10 GB or with `unreserved`); a slot lost its WAL (critical); new subscription errors. The size of `pg_wal` as a whole is in **Pulse**.

<a id="failover"></a>

## Failover readiness

This tool exists only for a Patroni cluster. **Failover readiness** shows what Patroni is ready for; the decision to switch over is Patroni's own. Every member except the leader gets a verdict, **ready** or **not ready**, with reasons:

- the member is unreachable;
- its state is neither streaming nor running;
- the lag exceeds `maximum_lag_on_failover` or is unknown;
- the `nofailover` tag;
- `failover_priority = 0` (Patroni 4 and later);
- it is not a synchronous replica while `synchronous_mode` is on (in `quorum` mode quorum replicas qualify as well).

A “restart pending” mark does not forbid being a candidate. Above the table is a summary: how many members are ready, the dynamic configuration of Patroni from `GET /config` (`maximum_lag_on_failover`, `synchronous_mode`, `synchronous_mode_strict`, `synchronous_node_count`, `ttl`, `loop_wait`, `retry_timeout`, permanent slots) and a warning if the cluster is paused and there is no automatic failover. If the configuration has no `maximum_lag_on_failover` key, the Patroni default of 1 MB is used. The configuration is read by one request per refresh, without a cache. When `/config` does not answer, the tool shows an error instead of data.

Findings: no ready candidate at all (critical), one ready candidate among three members or more (warning); `synchronous_mode` is on but there is no synchronous replica (critical); the cluster is paused; `ttl` is less than `loop_wait` plus twice `retry_timeout` — the leader key can expire during a DCS hiccup. Scheduled Patroni switchovers and restarts are shown not here but in the cluster overview: as a banner and as a mark in the member's row.

<a id="health"></a>

## Health matrix

This tool exists only for a Patroni cluster. **Health matrix** is a grid of members by REST API addresses: for every member that has an API address, `GET /primary`, `/standby-leader`, `/replica`, `/read-only`, `/sync`, `/async` and `/health` run in parallel (without authorization, 5 seconds per request). A cell holds the HTTP status code, or **no response** if the API did not answer (the reason is in the cell's tooltip). A member without an API address has no cells (**no REST API address**).

The cell color depends not on the code but on whether the response matches the member's role:

- green: the expected 200, such as `/primary` on the leader or `/replica` on a replica;
- gray: the endpoint does not apply to the role, so 503 is normal, such as `/primary` on a replica, `/replica` on the leader or `/sync` on an asynchronous replica;
- yellow: the node works, but the load balancer sends it no traffic for this endpoint, such as a replica answering 503 on `/replica` because of lag or the `noloadbalance` tag;
- red: the response contradicts the role (a replica answers 200 on `/primary`, or the leader answers as a leader nowhere), `/health` is not 200, or the API did not answer.

So a healthy cluster looks green and gray, and red shows at once where to look. The meaning of the color is in the cell's tooltip. This is how a load balancer sees the cluster, for example HAProxy with `option httpchk GET /primary`. Parameters like `/replica?lag=…` are not checked.

Findings: exactly one member must answer 200 on `/primary` (on `/standby-leader` in a standby cluster), otherwise a load balancer sends writes to the wrong place or nowhere (critical); `/health` is not 200 on a member whose API answered — PostgreSQL on that member is not running (critical); a replica whose `/replica` is not 200 is excluded from reads (warning: lag, the `noloadbalance` tag or a stopped replica).

<a id="actions"></a>

## Actions

Diagnostics can do more than show a problem: you can cancel a query, terminate sessions and drop a forgotten replication slot. An action runs only when you press its button, after a preview and a confirmation, and every action goes to the [actions journal](diagnostics.md#journal). The app never changes anything on its own. Actions on a Patroni cluster — switchover, failover, restart and pause — are in [Patroni actions](diagnostics.md#patroni-actions).

**What you can do and where**

- Cancel a query or terminate a session. Click a row in **Sessions**: the session panel gets the **Cancel query…** and **Terminate session…** buttons. Cancel interrupts the current query and the session stays; terminate drops the whole session. The buttons exist only for client sessions: server background processes are never signalled. Only a running query can be cancelled (state `active`): a session waiting for a client command (`idle`, `idle in transaction`) silently ignores a cancel, so its cancel button is inactive with a hint. To end a hanging transaction, terminate the session.
- A batch of idle in transaction sessions. In the **Idle in transaction** set (the **Idle in transaction** tab) there is an **Older than, min** field above the list — a whole number of minutes, at least 1, 5 by default — and a button “Terminate idle in transaction older than N min…”. A batch is always chosen by the condition, not by checkboxes in the table, and terminates every matching session. The finding about idle in transaction offers the same button: “Terminate idle in transaction older than 5 min” (the warning threshold).
- The root blocker. In the lock tree (**Locks**) a root — a session that waits for nothing itself — has the **Terminate the root blocker…** button; the root blocker finding has the same button (“Terminate session N”). Terminating the root frees the queue of victims.
- Dropping a slot. In the **Replication slots** tab every slot has a **Drop…** button.

**Preview.** Every action first opens a panel that reads the data from the server again instead of reusing what the table shows. For sessions it is a table with the PID, role, application, state, transaction age, the start of the query and a note. A session that must not be touched stays in the table in gray and does not take part in the action; the note gives the reason:

- **Superuser session: needs the superuser role** — only a superuser can terminate a superuser session;
- **Another role: needs pg\_signal\_backend** — without being a superuser you can touch another role's session only with the `pg_signal_backend` role;
- **Not a client session**;
- **The app's own connection** — the connection the app itself reads the preview with;
- **No query is running: cancelling does nothing; terminate the session to end its transaction** — for cancel only: the state is checked once more at execution, and if the query has finished meanwhile, the result is **No longer matches**.

The app's own connections (`application_name = glafira-pg`) carry the note **Glafira connection** in the preview: the server allows them, but the app rarely needs to terminate itself. If no session matches, the panel says **No matching sessions** (or, when there are sessions but none can be touched, **No session can be signalled**; for a single session, **This session cannot be terminated** or **This session's query cannot be cancelled**) and only offers to close. The list that was read does not refresh while you confirm.

**Safeguards.** They work inside the app, before it talks to the server, and not only in the window:

- on a read-only connection there are no action buttons, and a direct call is refused;
- on prod the confirmation asks for the connection name; a batch also asks for the number of sessions from the preview;
- dropping a slot is confirmed by typing the slot name in every environment, and on prod the connection name as well;
- a button you have no right to press is inactive, and its tooltip says why.

The privileges for the actions and how the action connection works are described in the administrator guide, section “Diagnostics” → “Actions”.

**Re-checking.** Things may have changed between the preview and the action:

- a session is addressed by its PID and its start time. If the PID has already gone to another session, the answer is “Already gone” and the other session is not hurt;
- a batch checks the condition again in the very statement that sends the signal. A session that has woken up in the meantime is not terminated: “No longer matches”. The root blocker is re-checked the same way — it must still hold up waiting sessions.

**Results.** After the action the panel shows the result for every session:

- **Terminated** — the session disappeared after termination (the app waits up to 5 seconds, polling every 250 ms);
- **Signal sent** — the cancel was sent, or the session is still alive 5 seconds after termination;
- **Already gone** and **No longer matches** — see re-checking;
- **Not allowed** — the server refused for lack of privileges;
- **Failed** — a server error, its text is next to it.

One action covers at most 500 sessions; they go one by one, a batch has to finish within 60 seconds, and sessions left after the deadline get an error. The action runs on a separate connection that is opened for one action and closed after it: it is not read-only, has a `statement_timeout` of 5 seconds and is named `glafira-pg` (`application_name`).

**A replication slot.** The preview shows the slot's type, plugin, database, state, `wal_status` and retained WAL. Only an inactive slot can be dropped: for an active one the button is inactive (“The slot is active”), and if the slot came alive while you were on your way to the window, the window only offers to close. Confirmation is typing the slot name (**slot name**). On a Patroni cluster the preview warns that Patroni will recreate the slot: a member slot while the member is in the cluster (remove the member), a permanent slot while it is listed in `/config` (remove it with `patronictl edit-config`). A logical slot is dropped from a connection to its own database. Dropping a slot needs the `REPLICATION` attribute or superuser; without it the button is inactive with the explanation **Needs the REPLICATION attribute: see the readiness item “Dropping replication slots”**. The risk of that attribute and the advice to give it to a separate role are in the administrator guide, section “Diagnostics” → “Actions”.

<a id="patroni-actions"></a>

## Patroni actions

Patroni actions exist only for a **Patroni cluster** connection and only in the cluster overview: above the member table there are the **Switchover…**, **Failover…** and **Pause** buttons (**Resume** after a pause); in the **Actions** column every member has **Restart…** and, if a restart is scheduled for it, **Cancel restart…**; the banner of a scheduled switchover has **Cancel…**. A read-only connection has none of these buttons.

They need an API login: fill in **API user** and **API password** in the connection profile (the password goes to the system secret store). Without a login the buttons are inactive with the tooltip “Set the Patroni API login in the connection to run actions”; they are also inactive while the connection is not open and while another action is running. If all API addresses in the profile are `https://`, the password is not sent to an `http://` address. Patroni must know the account (`restapi.authentication`); the details are in the administrator guide.

Every action opens a preview. It does not poll the cluster again: the leader, the states of the members and the readiness of the replicas come from the last poll of the topology (it is as old as the polling interval at most — 10 seconds, or 2 seconds while an action is being watched); only the dynamic configuration (`/config`) and the number of client sessions on the leader are read at that moment. The conditions of the actions below are checked against the last poll as well.

- **Switchover** is a planned change of leader. It needs a running leader (one that answers the poll and is in the `running` state); without it the button is inactive and failover is the way out. In the window choose a candidate — **Let Patroni choose** or a ready replica (replicas that are not ready are shown with the reason and cannot be chosen) — and the time: **Now** or **Scheduled** with a date and time in your time zone. The time field is filled at once with a moment about 15 minutes ahead (rounded up to 5 minutes), so that a scheduled action can still be cancelled; a time in the past is not accepted, and when less than a minute is left the window warns that you will hardly have time to cancel. For a switchover “now” the window warns how many client sessions on the leader will break (the app's own `glafira-pg` sessions are not counted, Patroni's own connection is) and that writes will be unavailable for a few seconds; it also says that the old leader will become a replica — for a scheduled switchover this is the only note. The action is sent with the leader the window showed: if the leader has changed meanwhile (an automatic failover, for example), the app refuses (“The leader changed…”) instead of switching over from another leader; a chosen candidate is checked for readiness once more before sending. While the cluster is paused Patroni switches over only to a named candidate, so **Let Patroni choose** is not available and the first ready replica is chosen by default.
- **Cancel switchover.** If a switchover is scheduled, the banner of the overview offers **Cancel…**.
- **Failover** is an emergency change of leader when there is no running leader. While the leader answers and is running, the button is inactive (“The leader is healthy: use switchover”). The candidate is a ready replica (the first one by default), and the command itself enforces this rule: before sending, the app evaluates the candidate's readiness again on the last poll and refuses if it is not ready (“… is not ready to become the leader”, the reasons are in the error details). Patroni itself does not apply the lag limit to an explicitly named candidate, and a lagging replica would become the leader with data loss. If Patroni sees the leader as running and only the app cannot reach it (a closed API port, for example), failover is allowed, but the window warns plainly that failover will demote that leader. Confirmation is typing the candidate name (**candidate name**), and on prod the connection name too.
- **Restart** restarts PostgreSQL on a member, now or on a schedule (the time is chosen as for a switchover), with the **Only if a restart is pending** checkbox (it is checked automatically when the member has a pending restart). Restarting the leader is highlighted with a warning: writes are unavailable until the restart is over. For an unreachable member the button is inactive.
- **Cancel restart** is offered for a member with a scheduled restart.
- **Pause and resume.** While Patroni is paused it stops managing the cluster: no failover and no restarts until the pause is lifted.

The confirmation is the **Run** button; on prod you also type the connection name.

**How the request goes.**

- The request goes to the leader's REST API; restart and its cancellation go only to the API of the member itself. For a request to the leader the addresses are tried in this order: the members whose API answered the last poll (the leader first among them), then the other members, then the addresses from the connection profile; repeated addresses are dropped. This is why, during a failover, when the leader is silent, the request does not wait for it.
- The next address is tried only when the request certainly did not reach Patroni: a refused connection, the connection timeout (3 seconds), TLS or the tunnel.
- If Patroni stops answering after the request was sent — a timeout, a broken connection, a 503 “status unknown” from switchover or failover — the outcome is unknown: the app does not repeat the request, so that the action is not run twice. For a switchover or a restart “now”, a failover, a pause and a resume the window says **Patroni did not answer: whether the action ran will show in the progress below**. Cancellations and scheduled actions are not watched: there the unknown outcome comes as an ordinary error message, and whether the action ran is shown by the overview.
- The app does not follow HTTP redirects: a 3xx answer counts as a refusal.
- The answer timeout is 5 seconds; for switchover, failover and restart “now” it is up to 60 seconds, because Patroni answers after it has done the work. A 2xx answer is success; 401 and 403 mean the credentials were refused; anything else is a Patroni refusal with the code and the beginning of its answer.

**Progress.** After a switchover, a failover and a restart “now”, and after a pause and a resume, a **Progress** panel opens under the window — the steps up to “Done” and the deadline. While it is open the topology is polled every 2 seconds instead of the usual 10. The steps:

- switchover and failover: **Accepted**, **The old leader stepped down**, **The new leader is running** (the chosen candidate, if there is one), **The timeline increased**, **Replicas are streaming**, **Done**; the deadline is 120 seconds;
- restart: **Accepted**, **Restarted** (the PostgreSQL start time that the member's API reports has changed), **The node is running**, with the **Only if a restart is pending** checkbox on — **Pending restart cleared**, **Done**; the deadline is 5 minutes;
- pause and resume: **Accepted**, **The cluster state changed**, **Done**; the deadline is 30 seconds.

If the deadline passes, the panel says **The watch timed out: check the overview** and keeps the last step reached. Closing the panel (**Close**) or the overview tab stops the watching and returns polling to the usual interval; closing the connection stops it too. Scheduled actions are not watched: the overview shows a banner or a mark in the member's row.

<a id="journal"></a>

## Actions journal

Every action that was run is appended to the `actions.jsonl` file in the app data folder (the path is shown next to **Data:** on the **About** screen; per system it is in the administrator guide). Only the owner can access the file (permissions 0600 on macOS and Linux); at 5 MB it is renamed to `actions.1.jsonl`, so two files are kept — the current one and one old copy. The record is written after the action.

The **Actions journal** tab is the last item of the **Diagnostics** group of a connection. It is a table of the last 1000 records of the connection, newest first: time, action, node, object (a slot, the number of sessions or the address of the Patroni request) and the outcome — **done**, **partly** (part of a batch was not done) or **error**. Clicking the action name expands the full record. **Refresh** reads the file again.

What is recorded:

- for any action — the time, the connection (name and environment), the node, whether prod was confirmed, the outcome, the error code on failure and the duration;
- for sessions — one entry per session of the action, including “Already gone” and “No longer matches”: the PID and start time, role, database, application, client, state, transaction age, query identifier, the **query text** and the result. The text comes from `pg_stat_activity`: the first 500 characters, literals included as they are, so that when an incident is reviewed you can see what exactly was terminated. Treat the file as confidential;
- for a slot — the name, type, plugin, database, retained WAL and `wal_status`;
- for Patroni — the method, the request address without credentials, the body, the response code and the first 500 characters of the response. If there was no response (an unknown outcome), the outcome is “error” and there is no response code.

Safeguard refusals (read-only, a wrong confirmation), unmet preconditions and the case “the slot became active or is gone” are not written: nothing happened. Server errors and Patroni refusals are written with the outcome “error”. If the write failed (disk, permissions), the action has still been done, and the window says **The journal was not written**. The journal lives only on the user's computer and is not part of the diagnostics archive.

<a id="findings"></a>

## Findings

Above the data of each tab is the **Findings** block: rules check the data of the tool and name what needs attention. The block collapses; if there are no problems, it says “No problems found”.

A finding has a severity — **Info**, **Warning** or **Critical** (color and a text label), a title with numbers and an expandable **What to do** hint. The buttons under a finding open the right tool on the right node; some findings have a ready command with the **Copy command** button. The app does not run such commands: you copy and run them yourself. The exception is two findings with an action button: the one about idle in transaction (“Terminate idle in transaction older than 5 min”, the warning threshold) and the one about a root blocker (“Terminate session N”). The button opens the same preview as in the tool ([Actions](diagnostics.md#actions)); a read-only connection does not have it. Severity does not depend on the environment of the connection.

A rule that can fire on many objects (long queries, idle transactions, inactive slots, lagging replicas) gives one finding per node: the title shows the worst object, and the number of objects that reached the threshold is in the parameters. The exceptions are the root blocker and queued DDL: those get a finding per root and per item of the queue. In the tools of a cluster (**Pulse**, **Failover readiness**, **Health matrix**) findings belong to different nodes, and the node is shown next to each.

Thresholds: the first value is a warning, the second is critical.

- Connections in use: 80% and 95% of those available.
- Idle in transaction: 5 minutes and 30 minutes; an active query: 15 minutes and 1 hour; a transaction: 1 hour and 6 hours.
- XID age: 50% and 75% of the wraparound limit.
- A lock wait: 30 seconds and 5 minutes; a root blocker — a victim waits 10 seconds and 60 seconds, or 10 victims.
- The size of `pg_wal`: 2 and 4 times `max_wal_size`.
- xmin horizon: 10 million and 100 million transactions, or 1 hour and 6 hours.
- Replica lag: 16 MB and 1 GB, or 30 seconds and 5 minutes; a replica silent for 60 seconds.
- An inactive slot retains 1 GB and 10 GB of WAL.
- The share of `Lock` in the wait profile: 20% and 50% of samples.

Findings with the “Info” severity need no immediate action: a low cache hit in the interval (below 90% with at least 1000 blocks read), new temporary files, `idle_in_transaction_session_timeout` equal to 0, hidden query texts, a cycle of lock waits, recovery conflicts that have accumulated but do not grow. Deadlocks in the interval raise a warning in **Pulse**, and the details exist only in the server log: the application should take locks in one order.

Rates and growth (recovery conflicts, subscription errors, deadlocks) are computed between two consecutive refreshes, so on the first refresh, after a server restart and after a statistics reset there are no findings about growth.
