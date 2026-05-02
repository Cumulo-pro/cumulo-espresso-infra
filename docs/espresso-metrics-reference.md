# Espresso Network - Metrics Reference

> **By Cumulo** | April 2026  
> Complete reference of available Prometheus metrics exposed by the Espresso sequencer node, with descriptions oriented to node operators.

---

## How to Read This Reference

Each metric entry includes:
- **Metric name** - the Prometheus metric identifier
- **Type** - Gauge, Counter, or Histogram
- **PromQL function** - the recommended function to use in queries
- **Operator description** - what this metric means in practice and when to act on it

---

## 🟢 Node Health

---

### `up`
**Type:** Gauge  
**PromQL:** `up{job="espresso-decaf-cumulo"}`  
**Description:** Standard Prometheus scrape health indicator. Returns `1` if Prometheus can reach the node's metrics endpoint, `0` if the node is unreachable or the metrics endpoint is down. This is the most fundamental health check - if this is `0`, no other metrics will update. Alert immediately if this drops to `0`.

---

### `consensus_build_info`
**Type:** Gauge  
**PromQL:** `consensus_build_info{job=~"$node"}`  
**Description:** Exposes build metadata as labels (version, git commit, build date). Useful to confirm which version of the sequencer binary is running and to detect unintended downgrades after updates.

---

### `consensus_version`
**Type:** Gauge  
**PromQL:** `consensus_version{job=~"$node"}`  
**Description:** Protocol version currently active on the node. Useful to verify that the node has applied a protocol upgrade correctly and is running the expected version after a network upgrade event.

---

### `aggregator_height`
**Type:** Gauge  
**PromQL:** `aggregator_height{job=~"$node"}`  
**Description:** The current block height as seen by the node's aggregator. This is the primary sync indicator - it should increase continuously as new blocks are decided. A value stuck at `1` means the node is not yet participating in consensus (usually because stake has not been delegated). A value that stops increasing on an active validator signals a sync problem.

---

### `consensus_node_index`
**Type:** Gauge  
**PromQL:** `consensus_node_index{job=~"$node"}`  
**Description:** The index assigned to this node within the current active validator set. A value of `-1` or `0` may indicate the node is not yet registered in the active stake table for the current epoch. Useful as a quick check to confirm the node is recognized by the network as an active participant — cross-reference with `consensus_last_voted_view` when diagnosing participation issues.

---

## 📦 Consensus

---

### `consensus_current_view`
**Type:** Gauge  
**PromQL:** `consensus_current_view{job=~"$node"}`  
**Description:** The current HotShot view number the node is processing. Views are the internal time unit of the HotShot consensus protocol - each view corresponds to one block proposal opportunity. This value should increase continuously and closely track `consensus_last_decided_view`. A large gap between current view and last decided view indicates the network is struggling to reach consensus.

---

### `consensus_last_decided_view`
**Type:** Gauge  
**PromQL:** `consensus_last_decided_view{job=~"$node"}`  
**Description:** The view number of the most recently finalized (decided) block. Compare this against `consensus_current_view` - under normal conditions the gap should be small (1-3 views). A growing gap signals consensus is stalling and blocks are not being finalized at the expected rate.

---

### `consensus_last_voted_view`
**Type:** Gauge  
**PromQL:** `consensus_last_voted_view{job=~"$node"}`  
**Description:** The last view in which this node cast a vote. If this value is not advancing while `consensus_current_view` is increasing, the node is receiving proposals but not voting — which means it is **not actively participating in consensus**. The most common cause is that the node is not present in the active stake table for the current epoch: check `consensus_node_index` and verify that stake has been delegated. Compare this metric directly against `consensus_current_view` on a timeseries graph to make the divergence immediately visible. See also the derived query `consensus_current_view - consensus_last_voted_view` in the [Derived Queries](#-derived-queries--useful-promql-combinations) section.

---

### `consensus_last_synced_block_height`
**Type:** Gauge  
**PromQL:** `consensus_last_synced_block_height{job=~"$node"}`  
**Description:** The height of the last block that has been fully synced and stored in the local database. Compare against `aggregator_height` to detect if the storage layer is falling behind the consensus layer.

---

### `consensus_number_of_timeouts`
**Type:** Counter  
**PromQL (rate):** `increase(consensus_number_of_timeouts{job=~"$node"}[5m])`  
**PromQL (total):** `consensus_number_of_timeouts{job=~"$node"}`  
**Description:** Total number of view timeouts since the node started. A timeout occurs when no block is decided within the expected time window and the network moves to the next view without a decision. Occasional timeouts are normal (network jitter, leader unavailability). A consistently high timeout rate indicates network instability or that too many validators are offline.

> **Note on PromQL usage:** Use `increase(...[5m])` in timeseries panels to visualize the rate of new timeouts over time. Use the raw counter value in stat panels when you want to display the cumulative total since node start. Both views are useful: the rate reveals active instability; the total gives a historical summary.

---

### `consensus_number_of_timeouts_as_leader`
**Type:** Counter  
**PromQL:** `increase(consensus_number_of_timeouts_as_leader{job=~"$node"}[5m])`  
**Description:** Number of times this specific node was the designated leader for a view and that view timed out. High values here mean this node is being elected leader but failing to get its proposals decided - check network connectivity, peer count, and L1 provider health.

---

### `consensus_number_of_views_per_decide_event`
**Type:** Gauge  
**PromQL:** `consensus_number_of_views_per_decide_event{job=~"$node"}`  
**Description:** How many views it takes on average to produce one decided block. Ideally this should be close to `1` (one view = one block). Values significantly above `1` mean multiple views are being skipped or timing out before a block is finalized, indicating reduced consensus efficiency.

---

### `consensus_number_of_views_since_last_decide`
**Type:** Gauge  
**PromQL:** `consensus_number_of_views_since_last_decide{job=~"$node"}`  
**Description:** Number of views that have elapsed since the last block was decided. Under normal conditions this should stay low (1-5). A large and growing value is an early warning that the network has stalled and no new blocks are being finalized. This is one of the most useful early-warning metrics: consider alerting if it exceeds 20 for more than a few minutes.

---

### `consensus_number_of_empty_blocks_proposed`
**Type:** Counter  
**PromQL:** `increase(consensus_number_of_empty_blocks_proposed{job=~"$node"}[1h])`  
**Description:** Number of empty blocks (blocks with no transactions) proposed by this node when acting as leader. Some empty blocks are normal during low-activity periods. A very high ratio of empty blocks compared to total blocks may indicate the node is not receiving transactions from the mempool or the builder is not functioning correctly.

---

### `consensus_proposal_to_decide_time`
**Type:** Gauge  
**PromQL:** `consensus_proposal_to_decide_time{job=~"$node"}`  
**Description:** Time in seconds from when a block is proposed to when it is decided (finalized) by the network. This is a key latency metric for the health of the consensus round. Low and stable values indicate a well-performing network. Spikes indicate that quorum is taking longer to form, often due to slow peers or network congestion.

---

### `consensus_previous_proposal_to_proposal_time`
**Type:** Gauge  
**PromQL:** `consensus_previous_proposal_to_proposal_time{job=~"$node"}`  
**Description:** Time elapsed between consecutive block proposals. Reflects the actual block time as experienced by this node. Significant increases here indicate slower-than-expected block production.

---

### `consensus_view_duration_as_leader`
**Type:** Gauge  
**PromQL:** `consensus_view_duration_as_leader{job=~"$node"}`  
**Description:** How long this node spent processing a view when it was the designated leader. Useful to identify if leadership rounds are taking longer than expected, which could impact the node's reputation within the validator set.

---

### `consensus_invalid_qc`
**Type:** Counter  
**PromQL:** `increase(consensus_invalid_qc{job=~"$node"}[1h])`  
**Description:** Number of invalid Quorum Certificates received by this node. QCs are the cryptographic proofs that a supermajority of validators agreed on a block. Receiving invalid QCs can indicate misbehaving peers or a misconfigured node. Should be zero under normal operation.

---

### `consensus_outstanding_transactions`
**Type:** Gauge  
**PromQL:** `consensus_outstanding_transactions{job=~"$node"}`  
**Description:** Number of transactions currently in the node's mempool waiting to be included in a block. A continuously growing value without corresponding block production indicates the node is receiving transactions but not processing them.

---

### `consensus_outstanding_transactions_memory_size`
**Type:** Gauge  
**PromQL:** `consensus_outstanding_transactions_memory_size{job=~"$node"}`  
**Description:** Memory consumed by pending transactions in the mempool, in bytes. Monitor alongside `consensus_outstanding_transactions` to detect abnormally large transactions or mempool bloat.

---

### `consensus_finalized_bytes`
**Type:** Counter  
**PromQL:** `increase(consensus_finalized_bytes{job=~"$node"}[1h])`  
**Description:** Total bytes of transaction data finalized by this node. A throughput indicator - useful to measure how much data the node is processing over time and to compare activity across epochs.

---

## 🌐 Network & Peers

---

### `consensus_libp2p_num_connected_peers`
**Type:** Gauge  
**PromQL:** `consensus_libp2p_num_connected_peers{job=~"$node"}`  
**Description:** Number of peers currently connected via libp2p. This is the direct P2P connectivity metric. A healthy node should maintain connections to a significant portion of the active validator set. Values below 5 should trigger investigation. A drop to 0 means the node is isolated from the network and cannot participate in consensus.

---

### `consensus_libp2p_is_ready`
**Type:** Gauge  
**PromQL:** `consensus_libp2p_is_ready{job=~"$node"}`  
**Description:** Whether the libp2p networking layer has completed initialization and is ready to communicate with peers. Should be `1` shortly after node startup. If it stays at `0`, there is a networking configuration problem (port not open, advertise address incorrect, etc.). Treat this as a secondary `up` check focused specifically on the networking layer.

---

### `consensus_libp2p_num_failed_messages`
**Type:** Counter  
**PromQL:** `increase(consensus_libp2p_num_failed_messages{job=~"$node"}[5m])`  
**Description:** Number of messages that failed to be delivered over libp2p. Some failures are normal (peers going offline, transient connectivity issues). A sustained high rate indicates persistent connectivity problems with specific peers or general network degradation.

---

### `consensus_cdn_num_failed_messages`
**Type:** Counter  
**PromQL:** `increase(consensus_cdn_num_failed_messages{job=~"$node"}[5m])`  
**Description:** Number of messages that failed to be delivered via the Espresso CDN (the relay network used as a fallback alongside libp2p). Failures here combined with libp2p failures indicate broad connectivity problems. Isolated CDN failures with healthy libp2p are less concerning.

---

### `consensus_cliquenet_sequencer_connections`
**Type:** Gauge  
**PromQL:** `consensus_cliquenet_sequencer_connections{job=~"$node"}`  
**Description:** Number of active connections in the CliqueNet overlay (the structured P2P topology used by HotShot). Complements `libp2p_num_connected_peers` - low values here alongside low libp2p peers confirms the node is poorly connected to the validator set.

---

### `consensus_cliquenet_sequencer_iqueue_cap`
**Type:** Gauge  
**PromQL:** `consensus_cliquenet_sequencer_iqueue_cap{job=~"$node"}`  
**Description:** Capacity of the inbound message queue. If this approaches its limit, the node is receiving more messages than it can process - a sign of being overwhelmed by network traffic or a slow processing pipeline.

---

### `consensus_cliquenet_sequencer_oqueue_cap`
**Type:** Gauge  
**PromQL:** `consensus_cliquenet_sequencer_oqueue_cap{job=~"$node"}`  
**Description:** Capacity of the outbound message queue. A saturated outbound queue means the node is trying to send more messages than the network can absorb - check for high timeout rates and peer connectivity issues.

---

### `consensus_internal_event_queue_len`
**Type:** Gauge  
**PromQL:** `consensus_internal_event_queue_len{job=~"$node"}`  
**Description:** Length of the internal HotShot event processing queue. Under normal operation this should be low and stable. A growing queue indicates the node's consensus processing is falling behind the rate of incoming events - usually caused by CPU saturation or slow storage.

---

## 🔗 L1 Connection (Sepolia)

---

### `consensus_l1_head`
**Type:** Gauge  
**PromQL:** `consensus_l1_head{job=~"$node"}`  
**Description:** The latest L1 (Sepolia) block number seen by the node. Should advance continuously as new Sepolia blocks are produced. A stale value means the node has lost connection to its L1 RPC provider - which will prevent stake table updates and may cause consensus issues.

---

### `consensus_l1_finalized`
**Type:** Gauge  
**PromQL:** `consensus_l1_finalized{job=~"$node"}`  
**Description:** The latest finalized L1 block number. Finalized blocks on Sepolia are used as the source of truth for stake table state. This should closely track `consensus_l1_head` with a small lag (~64 blocks on Ethereum). A large gap between head and finalized indicates L1 reorganization activity.

---

### `consensus_l1_failed_requests`
**Type:** Counter  
**PromQL:** `increase(consensus_l1_failed_requests{job=~"$node"}[5m])`  
**Description:** Number of failed RPC requests to the L1 provider. Intermittent failures are expected but should be rare. A sustained high rate means the L1 provider is unreliable - consider switching to a different RPC endpoint. This is particularly critical for stake table reads, which require broad `eth_getLogs` queries.

---

### `consensus_l1_failovers`
**Type:** Counter  
**PromQL:** `increase(consensus_l1_failovers{job=~"$node"}[1h])`  
**Description:** Number of times the node has switched to a fallback L1 provider. Each failover represents a detected failure of the primary provider. Monitor to detect chronic L1 provider instability that may require a permanent provider change.

> **Note on window size:** Use a `[1h]` window rather than `[5m]` for this metric. Failovers are infrequent events — a 5-minute window will almost always return 0 and mask real instability. The same applies to `consensus_l1_stream_reconnects`.

---

### `consensus_l1_stream_reconnects`
**Type:** Counter  
**PromQL:** `increase(consensus_l1_stream_reconnects{job=~"$node"}[1h])`  
**Description:** Number of WebSocket reconnections to the L1 provider. WebSocket connections to RPC providers drop periodically - some reconnects are normal. A very high reconnect rate indicates an unstable WSS provider and may cause missed L1 events. Use a `[1h]` window (not `[5m]`) for meaningful results, as reconnects are sparse events.

---

## 🗄️ Data Availability (DA Scanner)

---

### `scanner_running`
**Type:** Gauge  
**PromQL:** `scanner_running{job=~"$node"}`  
**Description:** Whether the DA scanner is currently running (`1`) or stopped (`0`). The DA scanner is the background process that verifies data availability for historical blocks. Should always be `1` on a DA node.

---

### `scanner_scanned_blocks`
**Type:** Counter  
**PromQL:** `scanner_scanned_blocks{job=~"$node"}`  
**Description:** Total number of blocks scanned by the DA scanner since startup. Should increase continuously. Useful to measure the scanner's progress when catching up from a cold start.

---

### `scanner_scanned_vid`
**Type:** Counter  
**PromQL:** `scanner_scanned_vid{job=~"$node"}`  
**Description:** Total number of VID (Verifiable Information Dispersal) chunks scanned. VID is the erasure-coded data availability scheme used by Espresso. This should advance alongside `scanner_scanned_blocks`.

---

### `scanner_missing_blocks`
**Type:** Gauge  
**PromQL:** `scanner_missing_blocks{job=~"$node"}`  
**Description:** Number of blocks for which the node has not been able to retrieve or verify DA data. This should be `0` on a healthy DA node. Any non-zero value means the node has gaps in its data availability - investigate whether the node is missing data from peers or whether pruning is too aggressive.

---

### `scanner_missing_vid`
**Type:** Gauge  
**PromQL:** `scanner_missing_vid{job=~"$node"}`  
**Description:** Number of VID chunks that are missing or unverified. Should be `0`. Missing VID data means the node cannot serve complete DA proofs for those blocks, reducing its usefulness as a DA node. Often caused by connectivity issues during initial sync.

---

### `scanner_current`
**Type:** Gauge  
**PromQL:** `scanner_current{job=~"$node"}`  
**Description:** The block height currently being scanned by the DA scanner. Compare against `aggregator_height` to determine how far behind the scanner is relative to the chain tip. A large lag is normal during initial sync but should converge over time. Use the derived query `aggregator_height - scanner_current` to express the lag as a single actionable number (see [Derived Queries](#-derived-queries--useful-promql-combinations) below).

---

## 🗃️ Storage & Database (PostgreSQL)

---

### `sql_open_transactions`
**Type:** Gauge  
**PromQL:** `sql_open_transactions{job=~"$node"}`  
**Description:** Number of PostgreSQL transactions currently open. Should be low and stable under normal operation. A sustained high value indicates the node is opening more transactions than it is closing - possibly due to slow queries or database contention.

---

### `sql_committed_transactions`
**Type:** Counter  
**PromQL:** `increase(sql_committed_transactions{job=~"$node"}[5m])`  
**Description:** Number of successfully committed database transactions. The primary measure of write throughput to PostgreSQL. Should be non-zero and stable on an active node. A sudden drop to zero indicates the node has stopped writing to the database.

---

### `sql_reverted_transactions`
**Type:** Counter  
**PromQL:** `increase(sql_reverted_transactions{job=~"$node"}[5m])`  
**Description:** Number of database transactions that were rolled back. Occasional reversions are normal. A high reversion rate indicates database errors or conflicts - check PostgreSQL logs for constraint violations or deadlocks.

---

### `sql_dropped_transactions`
**Type:** Counter  
**PromQL:** `increase(sql_dropped_transactions{job=~"$node"}[5m])`  
**Description:** Number of database transactions that were dropped before being executed. Dropped transactions indicate the node is generating more database writes than PostgreSQL can handle. If this is consistently high, consider tuning PostgreSQL connection pool settings or increasing database resources.

---

### `sql_transaction_duration`
**Type:** Histogram  
**PromQL:** `histogram_quantile(0.95, rate(sql_transaction_duration_bucket{job=~"$node"}[5m]))`  
**Description:** Distribution of database transaction durations in seconds. The p95 latency is the most useful view - high p95 values indicate slow database operations that may be impacting node performance. Values above 1 second warrant investigation of PostgreSQL performance.

---

## 📡 Proposal Fetcher

---

### `consensus_proposal_fetcher_fetched`
**Type:** Counter  
**PromQL:** `increase(consensus_proposal_fetcher_fetched{job=~"$node"}[5m])`  
**Description:** Number of block proposals successfully fetched from peers during catchup. Active during initial sync or after a node restart. Should decrease to zero once the node is fully caught up.

---

### `consensus_proposal_fetcher_failed`
**Type:** Counter  
**PromQL:** `increase(consensus_proposal_fetcher_failed{job=~"$node"}[5m])`  
**Description:** Number of failed proposal fetch attempts. Some failures during catchup are normal. Persistent failures mean the node cannot retrieve historical proposals from peers, which will prevent it from completing the catchup process.

---

### `consensus_proposal_fetcher_queue_len`
**Type:** Gauge  
**PromQL:** `consensus_proposal_fetcher_queue_len{job=~"$node"}`  
**Description:** Number of proposals currently queued for fetching. A large and growing queue during catchup is normal. If this never decreases, the fetcher is stalled and the node will not complete syncing.

---

### `consensus_proposal_fetcher_last_fetched`
**Type:** Gauge  
**PromQL:** `consensus_proposal_fetcher_last_fetched{job=~"$node"}`  
**Description:** View number of the most recently fetched proposal. Compare against `consensus_current_view` to measure how far behind in catchup the node is.

---

### `consensus_proposal_fetcher_last_seen`
**Type:** Gauge  
**PromQL:** `consensus_proposal_fetcher_last_seen{job=~"$node"}`  
**Description:** View number of the most recently seen proposal (received from the network, not necessarily fetched from storage). Should closely match `consensus_current_view` on a well-connected node.

---

## 🔄 Catchup

---

### `consensus_catchup_requests`
**Type:** Counter  
**PromQL:** `increase(consensus_catchup_requests{job=~"$node"}[5m])`  
**Description:** Number of catchup requests sent to peers to retrieve missing state. Active during initial sync. Should drop to zero once the node is fully synced.

---

### `consensus_catchup_request_failures`
**Type:** Counter  
**PromQL:** `increase(consensus_catchup_request_failures{job=~"$node"}[5m])`  
**Description:** Number of catchup requests that failed. A high failure rate during sync indicates that peers are not responding to catchup requests - check peer connectivity and whether the peers themselves are healthy.

---

## 📊 Sync Status

---

### `sync_status_running`
**Type:** Gauge  
**PromQL:** `sync_status_running{job=~"$node"}`  
**Description:** Whether the proactive sync scanner is running. This background process proactively scans for missing data ranges and fetches them from peers.

---

### `sync_status_ranges_scanned`
**Type:** Counter  
**PromQL:** `sync_status_ranges_scanned{job=~"$node"}`  
**Description:** Number of block ranges that have been scanned for missing data. Increases during sync and should stabilize once the node is fully caught up.

---

### `sync_status_avg_time_per_block_scanned`
**Type:** Gauge  
**PromQL:** `sync_status_avg_time_per_block_scanned{job=~"$node"}`  
**Description:** Average time in milliseconds to scan one block for missing data. A useful indicator of sync performance - high values indicate the scanner is slow, possibly due to database or network bottlenecks.

---

### `sync_status_range_size`
**Type:** Gauge  
**PromQL:** `sync_status_range_size{job=~"$node"}`  
**Description:** Size of the block ranges being scanned in each sync iteration. The node dynamically adjusts this based on available resources and network conditions.

---

## 🔢 Derived Queries — Useful PromQL Combinations

These are not standalone metrics but computed expressions that are particularly useful in dashboards and alerts. None of these require additional instrumentation — they combine existing metrics.

---

### DA Scanner lag (blocks behind chain tip)
```promql
aggregator_height{job=~"$node"} - scanner_current{job=~"$node"}
```
The most direct way to see how far the DA scanner is from the current chain tip. During normal operation this should be small and converging toward zero. Alert if it grows continuously for more than 10 minutes after the node has finished initial sync.

---

### Consensus gap (views without a decided block)
```promql
consensus_current_view{job=~"$node"} - consensus_last_decided_view{job=~"$node"}
```
Quantifies how many views are being processed without producing a finalized block. Should be 1–3 under healthy conditions. Values above 10 and growing indicate the network is struggling to reach quorum.

---

### Leader timeout ratio (fraction of own leader slots that fail)
```promql
increase(consensus_number_of_timeouts_as_leader{job=~"$node"}[5m])
  / increase(consensus_number_of_timeouts{job=~"$node"}[5m])
```
Isolates whether timeouts are caused by this node specifically (ratio close to 1) or are a network-wide problem (ratio close to 0). A high ratio points to local issues: connectivity, L1 provider health, or CPU saturation. Add a `> 0` guard on the denominator in alert rules to avoid division-by-zero.

---

### Node voting participation check
```promql
consensus_current_view{job=~"$node"} - consensus_last_voted_view{job=~"$node"}
```
Measures the gap between the current view and the last view in which this node voted. Under normal operation this should be 1–2. A gap larger than 10 and growing means the node has stopped voting — the most common cause is not being present in the active stake table. Cross-check with `consensus_node_index`.

---

*Reference maintained by [Cumulo](https://cumulo.pro) - trusted validator across the blockchain ecosystem.*
