---
title: Node Shutdown
summary: How to drain, decommission, and shut down a node.
toc: true
---

A node **shutdown** terminates the `cockroach` process on the node.

There are two ways to handle node shutdown:

- To temporarily stop a node and restart it later, **drain** the node before terminating the `cockroach` process. This is done when [upgrading the cluster version](upgrade-cockroach-version.html) or performing cluster maintenance (e.g., upgrading system software).
- To permanently remove the node from the cluster, **decommission** and drain the node before terminating the `cockroach` process. This is done when scaling down a cluster or reacting to hardware failures.

{{site.data.alerts.callout_success}}
This guidance applies to manual deployments. If you have a Kubernetes deployment, terminating the `cockroach` process is handled through the Kubernetes pods. The `kubectl drain` command is used for routine cluster maintenance. For details on this command, see the [Kubernetes documentation](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/).

Also see our documentation for [cluster upgrades](upgrade-cockroachdb-kubernetes.html) and [cluster scaling](scale-cockroachdb-kubernetes.html) on Kubernetes. 
{{site.data.alerts.end}}

This page describes:

- The details of the [node shutdown sequence](#node-shutdown-sequence).
- How to [prepare for graceful shutdown](#prepare-for-graceful-shutdown) by implementing connection retry logic and coordinating load balancer, process manager, and cluster settings.
- How to [perform node shutdown](#perform-node-shutdown) by draining or decommissioning the node.

<div class="filters filters-big clearfix">
    <button class="filter-button" data-scope="drain">Drain</button>
    <button class="filter-button" data-scope="decommission">Decommission</button>
</div>

## Node shutdown sequence

<section class="filter-content" markdown="1" data-scope="drain">
When a node is temporarily stopped, the following stages occur in sequence:
</section>

<section class="filter-content" markdown="1" data-scope="decommission">
When a node is permanently removed, the following stages occur in sequence:
</section>

<section class="filter-content" markdown="1" data-scope="decommission">
#### Decommissioning

An operator [initiates the decommissioning process](#decommission-the-node) on the node.

The node's [`is_decommissioning`](cockroach-node.html#node-status) field is set to `true` and its `membership` status is set to `decommissioning`, which causes all replicas to be slowly transferred to other nodes. The node's [`/health?ready=1` endpoint](monitoring-and-alerting.html#health-ready-1) continues to consider the node "ready" so that the node can function as a gateway to route client connections to relevant data.

{{site.data.alerts.callout_info}}
After this stage, the node is automatically drained. However, we recommend manually draining before decommissioning. For more information, see [Perform node shutdown](#perform-node-shutdown).
{{site.data.alerts.end}}
</section>

#### Draining

<section class="filter-content" markdown="1" data-scope="drain">
An operator [initiates the draining process](#drain-the-node-and-terminate-the-node-process) on the node.
</section>

<section class="filter-content" markdown="1" data-scope="decommission">
After all replicas are transferred, the node is automatically drained.
</section>

Node drain consists of the following consecutive phases:

1. **Wait phase:** The node's [`/health?ready=1` endpoint](monitoring-and-alerting.html#health-ready-1) returns an HTTP `503 Service Unavailable` response code, which causes load balancers and connection managers to reroute traffic to other nodes. This step completes when the [fixed duration set by `server.shutdown.drain_wait`](#server-shutdown-drain_wait) is reached.

1. **Active query phase:** All active transactions and statements on the node are allowed to complete. This includes transactions for which the node is a [gateway](architecture/life-of-a-distributed-transaction.html#gateway) and [distributed statements](architecture/sql-layer.html#distsql) initiated on other gateway nodes. After this step completes, CockroachDB closes all client connections to the node. This step completes either when all transactions have been processed or the [maximum duration set by `server.shutdown.query_wait`](#server-shutdown-query_wait) is reached.

1. **Lease transfer phase:** The node's [`is_draining`](cockroach-node.html#node-status) field is set to `true`, which removes the node as a candidate for replica rebalancing, lease transfers, and query planning. Any replicas holding [range leases](architecture/replication-layer.html#leases) or [Raft leaderships](architecture/replication-layer.html#raft) must be transferred to other nodes. This step completes when all transfers are completed.

    <section class="filter-content" markdown="1" data-scope="drain">
    If the above steps have not completed after 10 minutes by default, node draining will stop and must be restarted to continue. For more information, see [Drain timeout](#drain-timeout).
    </section>

    <section class="filter-content" markdown="1" data-scope="decommission">
    Since all range replicas were already transferred to other nodes during the [decommissioning](#decommissioning) stage, this step immediately resolves.
    </section>

<section class="filter-content" markdown="1" data-scope="decommission">
#### Status change

After decommissioning and draining are both complete, CockroachDB changes the node membership from `decommissioning` to `decommissioned`.
</section>

At this point, the `cockroach` process is still running. It is only stopped by process termination.

#### Process termination

<section class="filter-content" markdown="1" data-scope="decommission">
An operator [terminates the node process](#terminate-the-node-process).
</section>

<section class="filter-content" markdown="1" data-scope="drain">
After draining completes, the node process is automatically terminated.
</section>

A node **process termination** stops the `cockroach` process on the node. The node will stop updating its liveness record.

<section class="filter-content" markdown="1" data-scope="drain">
If the node then stays offline for the duration set by [`server.time_until_store_dead`](#server-time_until_store_dead) (5 minutes by default), the cluster considers the node "dead" and starts to transfer its range replicas to other nodes.

If the node is brought back online, its remaining range replicas will determine whether or not they are still valid members of replica groups. If a range replica is still valid and any data in its range has changed, it will receive updates from another replica in the group. If a range replica is no longer valid, it will be removed from the node.
</section>

<section class="filter-content" markdown="1" data-scope="decommission">
A node that stays offline for the duration set by the `server.time_until_store_dead` [cluster setting](cluster-settings.html) (5 minutes by default) is usually considered "dead" by the cluster. However, a decommissioned node retains **decommissioned** status.
</section>

{{site.data.alerts.callout_info}}
CockroachDB's node shutdown behavior does not match any of the [PostgreSQL server shutdown modes](https://www.postgresql.org/docs/current/server-shutdown.html).
{{site.data.alerts.end}}

## Prepare for graceful shutdown

Each of the [node shutdown steps](#node-shutdown-sequence) is performed in order, with each step commencing once the previous step has completed. However, because some steps can be interrupted, it's best to ensure that all steps complete gracefully.

Before you [perform node shutdown](#perform-node-shutdown), review the following prerequisites to graceful shutdown:

- Configure your [load balancer](#load-balancing) to monitor node health.
- Review and adjust [cluster settings](#cluster-settings) and [drain timeout](#drain-timeout) as needed for your deployment.
- Implement [connection retry logic](#connection-retry-loop) to handle closed connections.
- Configure the [termination grace period](#termination-grace-period) of your process manager or orchestration system.
<section class="filter-content" markdown="1" data-scope="decommission">
- Ensure that the [size and replication factor](#size-and-replication-factor) of your cluster are sufficient to handle decommissioning.
</section>

### Load balancing

Your [load balancer](recommended-production-settings.html#load-balancing) should use the [`/health?ready=1` endpoint](monitoring-and-alerting.html#health-ready-1) to actively monitor node health and direct client connections away from draining nodes.

To handle node shutdown effectively, the load balancer must be given enough time by the `server.shutdown.drain_wait` duration. For details, see [Cluster settings](#cluster-settings).

### Cluster settings

#### `server.shutdown.drain_wait`

`server.shutdown.drain_wait` sets a **fixed** duration for the ["wait phase"](#draining) of node drain. Because a load balancer reroutes connections to non-draining nodes within this duration (`0s` by default), this setting should be coordinated with the load balancer settings.

Increase `server.shutdown.drain_wait` so that your load balancer is able to make adjustments before this step times out. Because the drain process waits unconditionally for the `server.shutdown.drain_wait` duration, do not set this value too high.

For example, [HAProxy](cockroach-gen.html#generate-an-haproxy-config-file) uses the default settings `inter 2000 fall 3` when checking server health. This means that HAProxy considers a node to be down, and temporarily removes the server from the pool, after 3 unsuccessful health checks being run at intervals of 2000 milliseconds. To ensure HAProxy can run 3 conseuctive checks before timeout, set `server.shutdown.drain_wait` to `8s` or greater: 

{% include_cached copy-clipboard.html %}
~~~ sql
SET CLUSTER SETTING server.shutdown.drain_wait = '8s';
~~~

We also recommend setting your [client connection](connection-pooling.html#about-connection-pools) lifetime to be shorter than `server.shutdown.drain_wait`. This will cause your client to close its connections and reconnect on non-draining nodes before CockroachDB forcibly closes all connections to the draining node at the end of the ["active query phase"](#draining).

#### `server.shutdown.query_wait`

`server.shutdown.query_wait` sets the **maximum** duration for the ["active query phase"](#draining) of node drain. Active transactions and statements must complete within this duration (`10s` by default).

Ensure that `server.shutdown.query_wait` is greater than:

- The longest possible transaction in the workload that is expected to complete successfully.
- The `sql.defaults.idle_in_transaction_session_timeout` cluster setting, which controls the duration a session is permitted to idle in a transaction before the session is terminated (`0s` by default).
- The `sql.defaults.statement_timeout` cluster setting, which controls the duration a query is permitted to run before it is canceled (`0s` by default).

`server.shutdown.query_wait` defines the upper bound of the duration, meaning that node drain proceeds to the next step as soon as the last open transaction completes.

{{site.data.alerts.callout_success}}
If there are still open transactions on the draining node when the server closes its connections, you will encounter errors. Your application should handle these errors with a [connection retry loop](#connection-retry-loop).
{{site.data.alerts.end}}

#### `server.shutdown.lease_transfer_wait`

In the ["lease transfer phase"](#draining) of node drain, the server attempts to transfer all range leases and Raft leaderships from the draining node. `server.shutdown.lease_transfer_wait` sets the maximum duration of each iteration of this attempt `server.shutdown.lease_transfer_wait` (`5s` by default). Because this step does not exit until all transfers are completed, changing this value only affects the frequency at which drain progress messages are printed.

<section class="filter-content" markdown="1" data-scope="drain">
In most cases, the default value is suitable. However, do **not** set `server.shutdown.lease_transfer_wait` to a value lower than a few dozen milliseconds. In this case, leases can fail to transfer and node drain will not be able to complete.
</section>

<section class="filter-content" markdown="1" data-scope="decommission">
Since [decommissioning](#decommissioning) a node transfers all of its range replicas to other nodes, no replicas will remain on the node by the time draining begins. Therefore, no iterations occur during this phase. This setting can be left alone.
</section>

{{site.data.alerts.callout_info}}
The sum of [`server.shutdown.drain_wait`](#server-shutdown-drain_wait), [`server.shutdown.query_wait`](#server-shutdown-query_wait), and [`server.shutdown.lease_transfer_wait`](#server-shutdown-lease_transfer_wait) should not be greater than the configured [drain timeout](#drain-timeout).
{{site.data.alerts.end}}

<section class="filter-content" markdown="1" data-scope="drain">
#### `server.time_until_store_dead`

`server.time_until_store_dead` sets the duration after which a node is considered "dead" and its data is rebalanced to other nodes (`5m0s` by default). In the node shutdown sequence, this follows [process termination](#node-shutdown-sequence).

Before temporarily stopping nodes for planned maintenance (e.g., upgrading system software), if you expect any nodes to be offline for longer than 5 minutes, you can prevent the cluster from unnecessarily moving data off the nodes by increasing `server.time_until_store_dead` to match the estimated maintenance window:

{% include_cached copy-clipboard.html %}
~~~ sql
SET CLUSTER SETTING server.time_until_store_dead = '15m0s';
~~~

After completing the maintenance work and [restarting the nodes](cockroach-start.html), you would then change the setting back to its default:

{% include_cached copy-clipboard.html %}
~~~ sql
RESET CLUSTER SETTING server.time_until_store_dead;
~~~
</section>

### Drain timeout

By default, [node drain](#node-shutdown-sequence) must be completed within 10 minutes or it will stop and [must be re-initiated](#perform-node-shutdown) to continue. This can be observed with an `ERROR: drain timeout` message in the terminal output.

When [using `cockroach node drain` to drain manually](#drain-a-node-manually), however, the drain timeout can be configured with the `--drain-wait` flag. 

The value of `--drain-wait` should be greater than the sum of [`server.shutdown.drain_wait`](#server-shutdown-drain_wait), [`server.shutdown.query_wait`](#server-shutdown-query_wait), and [`server.shutdown.lease_transfer_wait`](#server-shutdown-lease_transfer_wait).

### Connection retry loop

At the end of the ["active query phase"](#draining) of node drain, the server forcibly closes all client connections to the node. If any open transactions were interrupted or not admitted by the server because of the connection closure, they will fail with one of the following errors:

- `57P01 server is shutting down`
- `08006 An I/O error occurred while sending to the backend`

These errors indicate that the current connection is no longer usable, and they are an expected occurrence during node shutdown. To be resilient to connection closures, **your application should use a retry loop** to reissue transactions that were open when a connection was closed. This allows procedures such as [rolling upgrades](upgrade-cockroach-version.html) to complete without interrupting your service.

{{site.data.alerts.callout_success}}
You can mitigate connection errors by setting your [client connection](connection-pooling.html#about-connection-pools) lifetime to be shorter than `server.shutdown.drain_wait`. This will cause your client to close its connections and reconnect on non-draining nodes before CockroachDB forcibly closes all connections to the draining node. However, you may still encounter these errors when connections are held open by long-running queries.
{{site.data.alerts.end}}

A connection retry loop should:

- Close the current connection.
- Open a new connection. This will be routed to a non-draining node.
- Reissue the transaction on the new connection.
- Repeat while the connection error persists and the retry count has not exceeded a configured maximum.

A write transaction that enters the [commit phase](architecture/transaction-layer.html#commits-phase-2) on a draining node will run to completion, but the node connection can be closed before your client receives a success message. Therefore, the connection retry logic considers the result of a previously open transaction to be unknown, and reissues the transaction.

When reissuing a write statement that relies on a primary key [`UNIQUE`](unique.html) constraint in your schema, for example, interpret a unique constraint violation message (`ERROR: duplicate key value violates unique constraint "primary"`) as a success.

### Termination grace period

On production deployments, a process manager or orchestration system can disrupt graceful node shutdown if its termination grace period is too short. 

If the `cockroach` process has not terminated at the end of the grace period, a `SIGKILL` signal is sent to perform a "hard" shutdown that bypasses CockroachDB's [node shutdown logic](#node-shutdown-sequence) and forcibly terminates the process. This can corrupt log files and, in certain edge cases, can result in temporary data unavailability, latency spikes, uncertainty errors, ambiguous commit errors, or query timeouts.

- When using [`systemd`](https://www.freedesktop.org/wiki/Software/systemd/) to run CockroachDB as a service, set the termination grace period with [`TimeoutStopSec`](https://www.freedesktop.org/software/systemd/man/systemd.service.html#TimeoutStopSec=) setting in the service file.

- When using [Kubernetes](kubernetes-overview.html) to orchestrate CockroachDB, set the termination grace period with `terminationGracePeriodSeconds` in the [StatefulSet manifest](deploy-cockroachdb-with-kubernetes.html?filters=manual#configure-the-cluster).

To determine an appropriate termination grace period: 

- [Run `cockroach node drain` with `--drain-wait`](#drain-a-node-manually) and observe the amount of time it takes node drain to successfully complete.

- On Kubernetes deployments, it is helpful to set `terminationGracePeriodSeconds` to be 5 seconds longer than the configured [drain timeout](#drain-timeout). This allows Kubernetes to remove a pod only after node drain has completed.

- In general, we recommend setting the termination grace period **between 5 and 10 minutes**. If a node requires more than 10 minutes to drain successfully, this may indicate a technical issue such as inadequate [cluster sizing](recommended-production-settings.html#sizing).

- Increasing the termination grace period does not increase the duration of a node shutdown. However, the termination grace period should not be excessively long, in case an underlying hardware or software issue causes node shutdown to become "stuck". 

<section class="filter-content" markdown="1" data-scope="decommission">
### Size and replication factor

Before decommissioning a node, make sure other nodes are available to take over the range replicas from the node. If no other nodes are available, the decommissioning process will hang indefinitely.

#### 3-node cluster with 3-way replication

In this scenario, each range is replicated 3 times, with each replica on a different node:

<div style="text-align: center;"><img src="{{ 'images/v21.2/decommission-scenario1.1.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

If you try to decommission a node, the process will hang indefinitely because the cluster cannot move the decommissioning node's replicas to the other 2 nodes, which already have a replica of each range:

<div style="text-align: center;"><img src="{{ 'images/v21.2/decommission-scenario1.2.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

To successfully decommission a node in this cluster, you need to **add a 4th node**. The decommissioning process can then complete:

<div style="text-align: center;"><img src="{{ 'images/v21.2/decommission-scenario1.3.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

#### 5-node cluster with 3-way replication

In this scenario, like in the scenario above, each range is replicated 3 times, with each replica on a different node:

<div style="text-align: center;"><img src="{{ 'images/v21.2/decommission-scenario2.1.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

If you decommission a node, the process will run successfully because the cluster will be able to move the node's replicas to other nodes without doubling up any range replicas:

<div style="text-align: center;"><img src="{{ 'images/v21.2/decommission-scenario2.2.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

#### 5-node cluster with 5-way replication for a specific table

In this scenario, a [custom replication zone](configure-replication-zones.html#create-a-replication-zone-for-a-table) has been set to replicate a specific table 5 times (range 6), while all other data is replicated 3 times:

<div style="text-align: center;"><img src="{{ 'images/v21.2/decommission-scenario3.1.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

If you try to decommission a node, the cluster will successfully rebalance all ranges but range 6. Since range 6 requires 5 replicas (based on the table-specific replication zone), and since CockroachDB will not allow more than a single replica of any range on a single node, the decommissioning process will hang indefinitely:

<div style="text-align: center;"><img src="{{ 'images/v21.2/decommission-scenario3.2.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>

To successfully decommission a node in this cluster, you need to **add a 6th node**. The decommissioning process can then complete:

<div style="text-align: center;"><img src="{{ 'images/v21.2/decommission-scenario3.3.png' | relative_url }}" alt="Decommission Scenario 1" style="max-width:50%" /></div>
</section>

## Perform node shutdown

<section class="filter-content" markdown="1" data-scope="drain">
After [preparing for graceful shutdown](#prepare-for-graceful-shutdown), do the following to temporarily stop a node. This both drains the node and terminates the `cockroach` process. To drain the node without process termination, see [Drain a node manually](#drain-a-node-manually).
</section>

<section class="filter-content" markdown="1" data-scope="decommission">
After [preparing for graceful shutdown](#prepare-for-graceful-shutdown), do the following to permanently remove a node.
</section>

{{site.data.alerts.callout_success}}
This guidance applies to manual deployments. If you have a Kubernetes deployment, terminating the `cockroach` process is handled through the Kubernetes pods. The `kubectl drain` command is used for routine cluster maintenance. For details on this command, see the [Kubernetes documentation](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/).

Also see our documentation for [cluster upgrades](upgrade-cockroachdb-kubernetes.html) and [cluster scaling](scale-cockroachdb-kubernetes.html) on Kubernetes. 
{{site.data.alerts.end}}

<section class="filter-content" markdown="1" data-scope="decommission">
### Drain the node

Although [draining automatically follows decommissioning](#draining), we recommend first running [`cockroach node drain`](cockroach-node.html) to manually drain the node of active queries and client connections before decommissioning. This is optional, but prevents possible disruptions in query performance. For specific instructions, see the [example](#drain-a-node-manually).

### Decommission the node

Run [`cockroach node decommission`](cockroach-node.html) to decommission the node and transfer its range replicas. For specific instructions and additional guidelines, see the [example](#remove-nodes).

{{site.data.alerts.callout_danger}}
Do **not** terminate the node process before a decommissioning node has [changed its membership status](#status-change) to `decommissioned`. This will prevent the node from transferring all of its range replicas to other nodes, and will cause issues.
{{site.data.alerts.end}}

### Terminate the node process
</section>

<section class="filter-content" markdown="1" data-scope="drain">
### Drain the node and terminate the node process
</section>

{% include {{ page.version.version }}/prod-deployment/process-termination.md %}

## Monitor shutdown progress

### `stderr`

<section class="filter-content" markdown="1" data-scope="drain">
When CockroachDB receives a signal to [drain and terminate the node process](#drain-the-node-and-terminate-the-node-process), this message is printed to `stderr`:
</section>

<section class="filter-content" markdown="1" data-scope="decommission">
When CockroachDB receives a signal to [terminate the node process](#terminate-the-node-process), this message is printed to `stderr`:
</section>

~~~
initiating graceful shutdown of server
~~~

After the `cockroach` process has stopped, this message is printed to `stderr`:

~~~
server drained and shutdown completed
~~~

### `OPS`

During node shutdown, progress messages are generated in the [`OPS` logging channel](configure-logs.html#ops-channel). The frequency of these messages is configured with [`server.shutdown.lease_transfer_wait`](#server-shutdown-lease_transfer_wait). [By default](configure-logs.html#default-logging-configuration), the `OPS` logs output to a `cockroach.log` file.

<section class="filter-content" markdown="1" data-scope="decommission">
Node decommission progress is reported in [`node_decommissioning`](eventlog.html#node_decommissioning) and [`node_decommissioned`](eventlog.html#node_decommissioned) events:
    
{% include_cached copy-clipboard.html %}
~~~ shell
grep 'decommission' node1/logs/cockroach.log
~~~

~~~
I220211 02:14:30.906726 13931 1@util/log/event_log.go:32 ⋮ [-] 1064 ={"Timestamp":1644545670665746000,"EventType":"node_decommissioning","RequestingNodeID":1,"TargetNodeID":4}
I220211 02:14:31.288715 13931 1@util/log/event_log.go:32 ⋮ [-] 1067 ={"Timestamp":1644545670665746000,"EventType":"node_decommissioning","RequestingNodeID":1,"TargetNodeID":5}
I220211 02:16:39.093251 21514 1@util/log/event_log.go:32 ⋮ [-] 1680 ={"Timestamp":1644545798928274000,"EventType":"node_decommissioned","RequestingNodeID":1,"TargetNodeID":4}
I220211 02:16:39.656225 21514 1@util/log/event_log.go:32 ⋮ [-] 1681 ={"Timestamp":1644545798928274000,"EventType":"node_decommissioned","RequestingNodeID":1,"TargetNodeID":5}
~~~
</section>

Node drain progress is reported in unstructured log messages:

{% include_cached copy-clipboard.html %}
~~~ shell
grep 'drain' node1/logs/cockroach.log
~~~

~~~
I220202 20:51:21.654349 2867 1@server/drain.go:210 ⋮ [n1,server drain process] 299  drain remaining: 15
I220202 20:51:21.654425 2867 1@server/drain.go:212 ⋮ [n1,server drain process] 300  drain details: descriptor leases: 7, liveness record: 1, range lease iterations: 7
I220202 20:51:23.052931 2867 1@server/drain.go:210 ⋮ [n1,server drain process] 309  drain remaining: 1
I220202 20:51:23.053217 2867 1@server/drain.go:212 ⋮ [n1,server drain process] 310  drain details: range lease iterations: 1
W220202 20:51:23.772264 681 sql/stmtdiagnostics/statement_diagnostics.go:162 ⋮ [n1] 313  error polling for statement diagnostics requests: ‹stmt-diag-poll›: cannot acquire lease when draining
E220202 20:51:23.800288 685 jobs/registry.go:715 ⋮ [n1] 314  error expiring job sessions: ‹expire-sessions›: cannot acquire lease when draining
E220202 20:51:23.819957 685 jobs/registry.go:723 ⋮ [n1] 315  failed to serve pause and cancel requests: could not query jobs table: ‹cancel/pause-requested›: cannot acquire lease when draining
I220202 20:51:24.672435 2867 1@server/drain.go:210 ⋮ [n1,server drain process] 320  drain remaining: 0
I220202 20:51:24.984089 1 1@cli/start.go:868 ⋮ [n1] 332  server drained and shutdown completed
~~~

### `cockroach node status`

<section class="filter-content" markdown="1" data-scope="drain">
Draining status is reflected in the [`cockroach node status --decommission`](cockroach-node.html) output:

{% include_cached copy-clipboard.html %}
~~~ shell
cockroach node status --decommission --certs-dir=certs --host={address of any live node}
~~~

~~~
  id |     address     |   sql_address   |              build              |         started_at         |         updated_at         | locality | is_available | is_live | gossiped_replicas | is_decommissioning |   membership   | is_draining
-----+-----------------+-----------------+---------------------------------+----------------------------+----------------------------+----------+--------------+---------+-------------------+--------------------+----------------+--------------
   1 | localhost:26257 | localhost:26257 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:55.07734  | 2022-02-11 02:17:28.202777 |          | true         | true    |                73 | false              | active         | true
   2 | localhost:26258 | localhost:26258 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:56.203535 | 2022-02-11 02:17:29.465841 |          | true         | true    |                73 | false              | active         | false
   3 | localhost:26259 | localhost:26259 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:56.406667 | 2022-02-11 02:17:29.588486 |          | true         | true    |                73 | false              | active         | false
(3 rows)
~~~

- `is_draining == true` indicates that the node is either undergoing or has completed the draining process.
</section>

<section class="filter-content" markdown="1" data-scope="decommission">
Draining and decommissioning statuses are reflected in the [`cockroach node status --decommission`](cockroach-node.html) output:

{% include_cached copy-clipboard.html %}
~~~ shell
cockroach node status --decommission --certs-dir=certs --host={address of any live node}
~~~

~~~
  id |     address     |   sql_address   |              build              |         started_at         |         updated_at         | locality | is_available | is_live | gossiped_replicas | is_decommissioning |   membership   | is_draining
-----+-----------------+-----------------+---------------------------------+----------------------------+----------------------------+----------+--------------+---------+-------------------+--------------------+----------------+--------------
   1 | localhost:26257 | localhost:26257 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:55.07734  | 2022-02-11 02:17:28.202777 |          | true         | true    |                73 | false              | active         | false
   2 | localhost:26258 | localhost:26258 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:56.203535 | 2022-02-11 02:17:29.465841 |          | true         | true    |                73 | false              | active         | false
   3 | localhost:26259 | localhost:26259 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:56.406667 | 2022-02-11 02:17:29.588486 |          | true         | true    |                73 | false              | active         | false
   4 | localhost:26260 | localhost:26260 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:56.914003 | 2022-02-11 02:16:39.032709 |          | false        | false   |                 0 | true               | decommissioned | true
   5 | localhost:26261 | localhost:26261 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:57.613508 | 2022-02-11 02:16:39.615783 |          | false        | false   |                 0 | true               | decommissioned | true
(5 rows)
~~~

- `is_draining == true` indicates that the node is either undergoing or has completed the draining process.
- `is_decommissioning == true` indicates that the node is either undergoing or has completed the decommissioning process.
- When a node completes decommissioning, its `membership` [status changes](#status-change) from `decommissioning` to `decommissioned`.
</section>

## Examples

These examples assume that you have already prepared for a [graceful node shutdown](#prepare-for-graceful-shutdown).

<section class="filter-content" markdown="1" data-scope="drain">
### Stop and restart a node

To drain and shut down a node that was started in the foreground with [`cockroach start`](cockroach-start.html):

1. Press `ctrl-c` in the terminal where the node is running.

    ~~~
    initiating graceful shutdown of server
    server drained and shutdown completed
    ~~~

1. Filter the logs for draining progress messages. [By default](configure-logs.html#default-logging-configuration), the `OPS` logs output to a `cockroach.log` file:
	
	{% include_cached copy-clipboard.html %}
	~~~ shell
	grep 'drain' node1/logs/cockroach.log
	~~~

	~~~
	I220202 20:51:21.654349 2867 1@server/drain.go:210 ⋮ [n1,server drain process] 299  drain remaining: 15
	I220202 20:51:21.654425 2867 1@server/drain.go:212 ⋮ [n1,server drain process] 300  drain details: descriptor leases: 7, liveness record: 1, range lease iterations: 7
	I220202 20:51:23.052931 2867 1@server/drain.go:210 ⋮ [n1,server drain process] 309  drain remaining: 1
	I220202 20:51:23.053217 2867 1@server/drain.go:212 ⋮ [n1,server drain process] 310  drain details: range lease iterations: 1
	W220202 20:51:23.772264 681 sql/stmtdiagnostics/statement_diagnostics.go:162 ⋮ [n1] 313  error polling for statement diagnostics requests: ‹stmt-diag-poll›: cannot acquire lease when draining
	E220202 20:51:23.800288 685 jobs/registry.go:715 ⋮ [n1] 314  error expiring job sessions: ‹expire-sessions›: cannot acquire lease when draining
	E220202 20:51:23.819957 685 jobs/registry.go:723 ⋮ [n1] 315  failed to serve pause and cancel requests: could not query jobs table: ‹cancel/pause-requested›: cannot acquire lease when draining
	I220202 20:51:24.672435 2867 1@server/drain.go:210 ⋮ [n1,server drain process] 320  drain remaining: 0
	I220202 20:51:24.984089 1 1@cli/start.go:868 ⋮ [n1] 332  server drained and shutdown completed
	~~~

	The `server drained and shutdown completed` message indicates that the `cockroach` process has stopped. 

1. Start the node to have it rejoin the cluster.

    Re-run the [`cockroach start`](cockroach-start.html) command that you used to start the node initially. For example:

    {% include_cached copy-clipboard.html %}
    ~~~ shell
    cockroach start \
    --certs-dir=certs \
    --store=node1 \
    --listen-addr=localhost:26257 \
    --http-addr=localhost:8080 \
    --join=localhost:26257,localhost:26258,localhost:26259 \
    ~~~

    ~~~
    CockroachDB node starting at 2022-02-11 06:25:24.922474 +0000 UTC (took 5.1s)
    build:               CCL v22.1.0-alpha.1-791-ga0f0df7927 @ 2022/02/02 20:08:24 (go1.17.6)
    webui:               https://localhost:8080
    sql:                 postgresql://root@localhost:26257/defaultdb?sslcert=certs%2Fclient.root.crt&sslkey=certs%2Fclient.root.key&sslmode=verify-full&sslrootcert=certs%2Fca.crt
    sql (JDBC):          jdbc:postgresql://localhost:26257/defaultdb?sslcert=certs%2Fclient.root.crt&sslkey=certs%2Fclient.root.key&sslmode=verify-full&sslrootcert=certs%2Fca.crt&user=root
    RPC client flags:    cockroach <client cmd> --host=localhost:26257 --certs-dir=certs
    logs:                /Users/ryankuo/node1/logs
    temp dir:            /Users/ryankuo/node1/cockroach-temp2906330099
    external I/O path:   /Users/ryankuo/node1/extern
    store[0]:            path=/Users/ryankuo/node1
    storage engine:      pebble
    clusterID:           b2b33385-bc77-4670-a7c8-79d79967bdd0
    status:              restarted pre-existing node
    nodeID:              1
    ~~~
</section>

### Drain a node manually

You can drain a node separately from decommissioning the node or terminating the node process.

1. Run the [`cockroach node drain`](cockroach-node.html) command, specifying the ID of the node to drain (and optionally a custom [drain timeout](#drain-timeout) to allow draining more time to complete):

    {% include_cached copy-clipboard.html %}
    ~~~
    cockroach node drain 1 --host=localhost:{address of any live node} --drain-wait=15m --certs-dir=certs
    ~~~

    You will see the draining status print to `stderr`:

    ~~~
    node is draining... remaining: 50
    node is draining... remaining: 0 (complete)
    ok
    ~~~

1. Filter the logs for shutdown progress messages. [By default](configure-logs.html#default-logging-configuration), the `OPS` logs output to a `cockroach.log` file:
    
    {% include_cached copy-clipboard.html %}
    ~~~ shell
    grep 'drain' node1/logs/cockroach.log
    ~~~

    ~~~
    I220204 00:08:57.382090 1596 1@server/drain.go:110 ⋮ [n1] 77  drain request received with doDrain = true, shutdown = false
    E220204 00:08:59.732020 590 jobs/registry.go:749 ⋮ [n1] 78  error processing claimed jobs: could not query for claimed jobs: ‹select-running/get-claimed-jobs›: cannot acquire lease when draining
    I220204 00:09:00.711459 1596 kv/kvserver/store.go:1571 ⋮ [drain] 79  waiting for 1 replicas to transfer their lease away
    I220204 00:09:01.103881 1596 1@server/drain.go:210 ⋮ [n1] 80  drain remaining: 50
    I220204 00:09:01.103999 1596 1@server/drain.go:212 ⋮ [n1] 81  drain details: liveness record: 2, range lease iterations: 42, descriptor leases: 6
    I220204 00:09:01.104128 1596 1@server/drain.go:134 ⋮ [n1] 82  drain request completed without server shutdown
    I220204 00:09:01.307629 2150 1@server/drain.go:110 ⋮ [n1] 83  drain request received with doDrain = true, shutdown = false
    I220204 00:09:02.459197 2150 1@server/drain.go:210 ⋮ [n1] 84  drain remaining: 0
    I220204 00:09:02.459272 2150 1@server/drain.go:134 ⋮ [n1] 85  drain request completed without server shutdown
    ~~~

    The `drain request completed without server shutdown` message indicates that the node was drained.

<section class="filter-content" markdown="1" data-scope="decommission">
### Remove nodes

In addition to the [graceful node shutdown](#prepare-for-graceful-shutdown) requirements, observe the following guidelines:

- Before decommissioning nodes, verify that there are no [underreplicated or unavailable ranges](ui-cluster-overview-page.html#cluster-overview-panel) on the cluster.
- When removing nodes, decommission all nodes at once. Do not decommission the nodes one-by-one. This will incur unnecessary data movement costs due to replicas being passed between decommissioning nodes.
- Do **not** terminate the node process before a decommissioning node has [changed its membership status](#status-change) to `decommissioned`. This will prevent the node from transferring all of its range replicas to other nodes, and will cause issues.
- If you have a decommissioning node that appears to be hung, you can [recommission](#recommission-nodes) the node.

#### Step 1. Get the IDs of the nodes to decommission

Open the **Cluster Overview** page of the DB Console and [note the node IDs](ui-cluster-overview-page.html#node-details) of the nodes you want to decommission.

This example assumes you will decommission node IDs `4` and `5` of a 5-node cluster. 

#### Step 2. Drain the nodes manually

Run the [`cockroach node drain`](cockroach-node.html) command for each node to be removed, specifying the ID of the node to drain:

{% include_cached copy-clipboard.html %}
~~~
cockroach node drain 4 --host=localhost:{address of any live node} --certs-dir=certs
~~~

{% include_cached copy-clipboard.html %}
~~~
cockroach node drain 5 --host=localhost:{address of any live node} --certs-dir=certs
~~~

You will see the draining status of each node print to `stderr`:

~~~
node is draining... remaining: 50
node is draining... remaining: 0 (complete)
ok
~~~

Manually draining before decommissioning nodes is optional, but prevents possible disruptions in query performance. 

#### Step 3. Decommission the nodes

Run the [`cockroach node decommission`](cockroach-node.html) command with the IDs of the nodes to decommission:

{% include_cached copy-clipboard.html %}
~~~ shell
$ cockroach node decommission 4 5 --certs-dir=certs --host={address of any live node}
~~~

You'll then see the decommissioning status print to `stderr` as it changes:

~~~
  id | is_live | replicas | is_decommissioning |   membership    | is_draining
-----+---------+----------+--------------------+-----------------+--------------
   4 |  true   |       39 |        true        | decommissioning |    true
   5 |  true   |       34 |        true        | decommissioning |    true
(2 rows)
~~~

The `is_draining` field is `true` because the nodes were previously drained.

Once the nodes have been fully decommissioned, you'll see zero `replicas` and a confirmation:

~~~
  id | is_live | replicas | is_decommissioning |   membership    | is_draining
-----+---------+----------+--------------------+-----------------+--------------
   4 |  true   |        0 |        true        | decommissioning |    true
   5 |  true   |        0 |        true        | decommissioning |    true
(2 rows)

No more data reported on target nodes. Please verify cluster health before removing the nodes.
~~~

The `is_decommissioning` field remains `true` after all replicas have been transferred from each node.

{{site.data.alerts.callout_danger}}
Do **not** terminate the node process before a decommissioning node has [changed its membership status](#status-change) to `decommissioned`. This will prevent the node from transferring all of its range replicas to other nodes, and will cause issues.
{{site.data.alerts.end}}

#### Step 4. Confirm the nodes are decommissioned

Check the status of the decommissioned nodes:

{% include_cached copy-clipboard.html %}
~~~ shell
$ cockroach node status --decommission --certs-dir=certs --host={address of any live node}
~~~

~~~
  id |     address     |   sql_address   |              build              |         started_at         |         updated_at         | locality | is_available | is_live | gossiped_replicas | is_decommissioning |   membership   | is_draining
-----+-----------------+-----------------+---------------------------------+----------------------------+----------------------------+----------+--------------+---------+-------------------+--------------------+----------------+--------------
   1 | localhost:26257 | localhost:26257 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:55.07734  | 2022-02-11 02:17:28.202777 |          | true         | true    |                73 | false              | active         | false
   2 | localhost:26258 | localhost:26258 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:56.203535 | 2022-02-11 02:17:29.465841 |          | true         | true    |                73 | false              | active         | false
   3 | localhost:26259 | localhost:26259 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:56.406667 | 2022-02-11 02:17:29.588486 |          | true         | true    |                73 | false              | active         | false
   4 | localhost:26260 | localhost:26260 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:56.914003 | 2022-02-11 02:16:39.032709 |          | false        | false   |                 0 | true               | decommissioned | true
   5 | localhost:26261 | localhost:26261 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:57.613508 | 2022-02-11 02:16:39.615783 |          | false        | false   |                 0 | true               | decommissioned | true
(5 rows)
~~~

- Membership on the decommissioned nodes should have changed from `decommissioning` to `decommissioned`.
- `0` replicas should remain on these nodes.

The decommissioned nodes will no longer be visible in the DB Console.

#### Step 5. Terminate the process on decommissioned nodes

{% include {{ page.version.version }}/prod-deployment/process-termination.md %}

The following messages will be printed:

~~~
initiating graceful shutdown of server
server drained and shutdown completed
~~~

### Remove a dead node

If a node is offline for the duration set by `server.time_until_store_dead` (5 minutes by default), the cluster considers the node "dead" and starts to transfer its range replicas to other nodes. 

However, if the dead node is restarted, the cluster will rebalance replicas and leases onto the node. To prevent the cluster from rebalancing data to a dead node that comes back online, do the following:

#### Step 1. Confirm the node is dead

Check the status of your nodes:

{% include_cached copy-clipboard.html %}
~~~ shell
$ cockroach node status --decommission --certs-dir=certs --host={address of any live node}
~~~

~~~
  id |     address     |   sql_address   |              build              |         started_at         |         updated_at         | locality | is_available | is_live | gossiped_replicas | is_decommissioning |   membership   | is_draining
-----+-----------------+-----------------+---------------------------------+----------------------------+----------------------------+----------+--------------+---------+-------------------+--------------------+----------------+--------------
   1 | localhost:26257 | localhost:26257 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:45:45.970862 | 2022-02-11 05:32:43.233458 |          | false        | false   |                 0 | false              | active         | true
   2 | localhost:26258 | localhost:26258 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:46:40.32999  | 2022-02-11 05:42:28.577662 |          | true         | true    |                73 | false              | active         | false
   3 | localhost:26259 | localhost:26259 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:46:47.20388  | 2022-02-11 05:42:27.467766 |          | true         | true    |                73 | false              | active         | false
   4 | localhost:26260 | localhost:26260 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:56.914003 | 2022-02-11 02:16:39.032709 |          | true         | true    |                73 | false              | active         | false
(4 rows)
~~~

The `is_live` field of the dead node will be `false`.

Alternatively, open the **Cluster Overview** page of the DB Console and check that the [node status](ui-cluster-overview-page.html#node-status) of the node is `DEAD`.

#### Step 2. Decommission the dead node

Run the [`cockroach node decommission`](cockroach-node.html) command against the address of any live node, specifying the ID of the dead node:

{% include_cached copy-clipboard.html %}
~~~ shell
$ cockroach node decommission 1 --certs-dir=certs --host={address of any live node}
~~~

~~~
  id | is_live | replicas | is_decommissioning |   membership    | is_draining
-----+---------+----------+--------------------+-----------------+--------------
   1 |  false  |        0 |        true        | decommissioning |    true
(1 row)

No more data reported on target nodes. Please verify cluster health before removing the nodes.
~~~

#### Step 3. Confirm the node is decommissioned

Check the status of the decommissioned node:

{% include_cached copy-clipboard.html %}
~~~ shell
$ cockroach node status --decommission --certs-dir=certs --host={address of any live node}
~~~

~~~
  id |     address     |   sql_address   |              build              |         started_at         |         updated_at         | locality | is_available | is_live | gossiped_replicas | is_decommissioning |   membership   | is_draining
-----+-----------------+-----------------+---------------------------------+----------------------------+----------------------------+----------+--------------+---------+-------------------+--------------------+----------------+--------------
   1 | localhost:26257 | localhost:26257 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:45:45.970862 | 2022-02-11 06:07:40.697734 |          | false        | false   |                 0 | true               | decommissioned | true
   2 | localhost:26258 | localhost:26258 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:46:40.32999  | 2022-02-11 05:42:28.577662 |          | true         | true    |                73 | false              | active         | false
   3 | localhost:26259 | localhost:26259 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:46:47.20388  | 2022-02-11 05:42:27.467766 |          | true         | true    |                73 | false              | active         | false
   4 | localhost:26260 | localhost:26260 | v22.1.0-alpha.1-791-ga0f0df7927 | 2022-02-11 02:11:56.914003 | 2022-02-11 02:16:39.032709 |          | true         | true    |                73 | false              | active         | false
(4 rows)
~~~

- Membership on the decommissioned node should have changed from `active` to `decommissioned`.

The decommissioned node will no longer be visible in the DB Console.

### Recommission nodes

If you accidentally started decommissioning a node, or have a node with a hung decommissioning process, you can recommission the node. This cancels replica transfer from the decommissioning node.

Recommissioning can only cancel an active decommissioning process. If a node has [completed decommissioning](#status-change), you must start a new node. A fully decommissioned node is permanently decommissioned, and cannot be recommissioned.

#### Step 1. Cancel the decommissioning process

Press `ctrl-c` in each terminal with an ongoing decommissioning process that you want to cancel.

#### Step 2. Recommission the decommissioning nodes

Run the [`cockroach node recommission`](cockroach-node.html) command with the ID of the node to recommission:

{% include_cached copy-clipboard.html %}
~~~ shell
$ cockroach node recommission 1 --certs-dir=certs --host={address of any live node}
~~~

The value of `is_decommissioning` will change back to `false`:

~~~
  id | is_live | replicas | is_decommissioning | membership | is_draining
-----+---------+----------+--------------------+------------+--------------
   1 |  false  |       73 |       false        |   active   |    true  
(1 row)
~~~

On the **Cluster Overview** page of the DB Console, the [node status](ui-cluster-overview-page.html#node-status) of the node should be `LIVE`. After a few minutes, you should see replicas rebalanced to the nodes.
</section>

## See also

- [Upgrade CockroachDB](upgrade-cockroach-version.html)
- [`cockroach node`](cockroach-node.html)
- [`cockroach start`](cockroach-start.html)
- [Node status](ui-cluster-overview-page.html#node-status)