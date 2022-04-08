# This is a personal practice of the labs of MIT 6.824 2015 (http://nil.csail.mit.edu/6.824/2015/schedule.html)

## Lab2 Primary/Backup Key/Value Service

Have run the test 100 times without failtures.

Most of the hard parts are in Lab2B, below are challenges I face and how to solve them:

### Handle at-most-once

Each client will have requestId that is increased for each Put/Get requests, each PBServer maintain map of
client's address -> requestId. If a PBServer received a Put request with a lower or equal requestId, just reply OK indicate success. This map will be replicated or transferred to backup

### Keep Primary and Backup in-sync all the time

1. For a PutAppend request, a Primary must replicate to Backup first, if heard nothing or Backup rejects, let client retry without applying to primary's database.

    Otherwise, in an alternative design that Primary applies the change before replicating to Backup, once Backup fails to reach due to network glitches, the next retried client's request will be filtered by Primary's at-most-once map, and the change will never replicated to Backup.

    When Backup receives the replication requests, need to check at-most-once also, due to network glitches, Backup may already apply the change but fail to reply back to Primary, Primary will let client retry without applying, Primary will try to replicate the same command again to Backup, Backup will drop the request, otherwise a duplicate append may happen in Backup.

    In this way, Backup always applied more up-to-date changes than Primary, if Primary fails, Backup will take over as Primary with those changes applied, when clients' retried requests come to Backup (the new Primary), because of at-most-once, Backup will confirm the changes without applying again.

2. Due to network reorder, there is a risk that Backup's applied order is different than Primary's one, consider a case that Primary receives a c1 PutAppend from client1, replicated successfully to Backup but fail to reply back to Primary, Primary will force client to retry. But during this time, client2 requests d2 PutAppend, this is applied in both Backup and Primary successfully and client2 receives confirmation for this d2 request. But here, Backup and Primary is off-sync already, Backup applied c1,d2, but Primary only applied d2, if c1 is retried later in Primary, it will applied in order of d2, c1.

    This is wrong, Primary must have a record tracking which requests are applied by Backup in order. Here my solution is to maintain a log of requests received and also a commitIndex if applied. When Primary replicate to Backup, it passes its current last log index, Backup need to reply back with missing log entries Primary miss. With missing entries replied back, Primary will know the request order and only accept and apply client's requests in that order and for out-of-order requests to retry later. 
    
    In this way, the backup server has the authority on the applied order, this also work if Primary has the authority, say c1 arrives at Primary, it logs it first without commit, replicate to Backup for confirmation, without confirmation, Primary will not proceed d2 requests including don't forward d2 to backup. But I feel this will limit the efficiency, in my above approach, if c1 is not delivered to Backup, and d2 can be delivered and applied first, it is possible to form the order d2, c1 (d2's client request is ACKed before c1 is ACKed), which is still acceptable for concurrent clients.

    Without replied back from Backup with missing entries, Primary will never confirm the change. If Backup fails, Primary will get new view and purge all log entries beyond its commitIndex and transfer everything to the new Backup. Unconfirmed requests (those purged) will be retried again. If Primary fails, Backup will take over as Primary, when clients' retried requests come to Backup, because of at-most-once, Backup will confirm the changes without applying again. BUT here, a later applied requests may be ACKed before a former applied requests, which is fine for concurrent clients, to better help client's view, a recordVersion can be added to a Key, a client read a recordVersion first, if he gets update replies with higher recordVersion, it means some other clients already changes the same key. To make the ordering strict for one client's view, a transaction or locking may be applied, but beyond scope of this lab.

    Noted that this only works if the client keeps retrying until successful, it does not work in the case that one client stop retry and drop the request, in this case, the Primary will keep wait for this client while blocking other requests. A possible way is to let Primary maintain a timer for each logged event but not committed, if some log entry expired, send a special request to backup to erase that and following entries and undo the applied effects.

### Get never reply stale data and PutAppend's changes remain applied forever

When Primary receives Get/PutAppend, it will forward the requests to Backup, if fail to reach or Backup rejects, Primary never confirm the requests. Backup will reject if itself not backup anymore or its viewnum is different than Primary's one.

Is there a case that Client/Primary/Backup's view are all same but stale?

- It is possible that primary will still be the current primary (but store stale viewnum), in this case, it does not matter, primary apply the change and next tick(), it will fetch the latest view and still be the primary and serve clients.

- It is possible that viewservice promotes Backup as primary but Backup not pinged yet, in this case, every PutAppend will be replicated to Backup, who continues to be primary; every Get will return the last confirmed PutAppend.

- It is not possible that current primary is neither Primary nor Backup, by the rule `But the view service must not change views (i.e., return a different view to callers) until the primary from the current view acknowledges that it is operating in the current view (by sending a Ping with the current view number)`
in order for viewservice to promote other servers and increase the view, Backup must acknowledge the view that it being promoted to primary, which also means Backup must store a higher viewnum (than the original stale primary), then it will reject the original stale primary's requests and never confirm any of clients' requests, clients will fetch the new view and retry with latest primary

### In the case of network partition between Primary and Backup, there is no chance that both are Primary serving client's requests

This cannot happen, for every client's reqeust, Primary will consult Backup, if network partition or the Backup is Primary, it will reject the forward requests. The worst case is both Primary and Backup continue pings to viewservice but there is no network between them, in this case, client's requests are all dropped and keep retrying until network resumes or one server stops pinging the viewservice. This only affects the availability but not consistency. In real applications, the Primary-Backup comminucation channels can be monitored or servers send heartbeat to each other

## Lab3A - Paxos-based Key/Value Service (Paxos Protocol)

Understanding Paxos and implementing Paxos are different, especially for multi-paxos, below are some intakes:

- Proposer's prepare/accept/decide calls need to retry if servers are not accessible, otherwise it won't pass the partition unreliable test case, if partition changes, the retried requests can go through to help bring other servers into concensus. But in my implemenation, I make prepare/accept retry only until either majority found or value decided at given instanceIndex (some other concurrent proposers finish deciding the value) and decide retry indefinitely.

- In the proposer's implementation, I passed around a state object to help sync collecting majority and book-keep of other variables, like current highest value ever accepted or when to stop calling other servers.

- According to the paper, the proposal number needs to be unique and globally increasing. It actually does not need to be globally increasing, which is hard to achieve in distributed setup, each proposer just need to select a number and try send them waiting for a prepareReply with a proposal number for correction (if any) and then retry. But it must be unique, why? On initial thought, it may be fine if multiple proposers propose the same proposal number, because prepare requests with equal proposal numbers will be discarded by each server's prepare-handler. But if proposers propose the same proposal numbers at the same time, they will not prepare-accept each other, which never ends. Unique means they have total ordering, in the above scenario, there must have one proposal bigger than others and finally win the acception and block others. So this uniqueness is for liveness rather for correctness.

- To ensure each proposal numbers unique, it shifts the server index into the lowest significant bits of proprosal number

## Lab3B - Paxos-based Key/Value Service (Key/Value Server)

Implement a key-value store on top of multi-paxos of Lab3A, below are some intakes:

- KVServer can accept and process requests from differnt client concurrently and put into its log concurrently, it just keep trying instanceIndex++ until that value is decided and is of the same requestId. Because in Lab3A, once paxos.Start() is called with an instanceIndex, paxos will try indefinitely until there is a decidedValue for this instanceIndex. So we don't need to worry that there are holes within the KVServer's log, these holes are temporary, given enough time, if being within majority, the holes are filled eventually. For being within minority (in case of network partitions), the holes may never get filled, so clients will wait forever until being back into majority.

- Because of above, we don't need a leader or force KVServer fill those holes with NOP oprations (like mentioned in "Paxos Made Simple"). In reality, a timeout needs to be implemented carefully that if the holes are not filled in time, leader or current KVServer may try propose a NOP operation trying to accelarate the agreement of this concensus. Noted that there may already decided value in this instanceIndex, just not received decided requests yet. So a NOP operation will try to re-propose this decided value and reach agreement.

- The application of changes and the response to clients need to be processed in sequence, there should be no holes in between

- Design and implement the at-most-once property of requests is the most challenging part, normally to handle this, server need to maintain a request cache that filters any duplicate requests (e.g. with same requestId). But here in kvpaxos, if one server fails to respond back to client, client may retry the same request (with same requestId) to other servers. Other servers don't have the request cache with this request, so they will accept it, but the first server may already decide and apply this request in paxos layer. This may result in two requests with same requestId stored in log and potentially apply twice if not handled correctly. Because the request cache of different servers can be off-sync, so it won't prevent duplicate requests storing into its own log, so the filtering must happen in apply stage, which means if they can detect requests are seen before, just skip it and don't apply the changes.

- To handle at-most-once property, cache is checked and filtering in apply stage and updated only after applied successfully. For Get requests, only the first Get operation is applied, the second or more duplicate Get operation will be skipped but the current value of the cached Key is returned back to client. This is safe, because client has not received a response from this Get request yet, in this lab's setup, the same client cannot issue other requests while the current Get is pending, so all the log entries before the final Get requests are not coming from this same client. Likewise for Put/Append requests, only the first Put/Append operation is applied, the second or more duplicate Put/Append operation will be skipped but respond to client. Actually like the Key-Value store, the request cache of each server is replicated in a fault-tolerance way as well.

- Noted that for request cache, we cache the key instead of the value, for example, for Get request, this is to ensure that once clients receive response back from Get operation, it will get the lastest status of the database rather than stale data. Another tricky thing is the cache cannot be truncated, because we don't know if there are any duplicate requests coming in (espeically true for some queue-based systems, which has items stuck and pending in the queue for many days or even months, or to redrive for some backfill tasks)

## Lab4A - Sharded Key/Value Service (Shard Master)

For shardmaster, the overall code structure mirrors the kvpaxos of Lab3B including how to call paxos, how to wait, how to apply, how to truncate logs, etc. The only difference is the underlaying storage is a list of configurations instead of kv stores. The challenging part is to achieve minmal shard transfers in join/leave-rebalance. The intuitive understanding is that only transfer shards as necessary, don't transfer more than needed. In other words, for groups with more shards, only transfer out as minimal as possible; for groups with less shards, only transfer in as minimal as possible. With new setup of groups, it is easy to come up with the resulting number of shards per group. Sort them in decreasing order and also sorting the groups in decreasing order in terms of their number of shards, in this way, we can compute how many shards need to be transferred. Compare them one-by-one and note down for each group, how many more shards that need to be transfered out and how may less shards that need to be transfered in. Maintain two list of groups: groups need to transfer out shards and groups need to transfer in shards. Then process these two lists and move shards from `more list` to `less list`. To handle join case, the newGroup needs to be in `less list` and for leave case, the leavingGroup needs to be in `more list`

## Lab4B - Sharded Key/Value Service (Sharded Key/Value Server)

Similar to Lab4A, the overral code structure mirrors that kvpaxos of Lab3B but supporting reconfiguration. All the reconfiguration steps are proposed and decided for certain Paxos instance index. And during the application loop, apply those reconfigurations, same as all other KV queries (in this way each sever can ensure the reconfiguration has been applied in majority of servers before applying current one). But below are some caveats or insights when implement it

- The reconfiguration involves 3 phases: discover/start new configuration (from master), sending/receiving shards of data and clientReq cache (for client dedupe), end new configuration. And these 3 phases must be in serial, which means the next round of new configuration won't start until the previous finish. This can ensure that all the shards are delivered to corresponding groups successfully. To prevent old configurations overwrite newer ones, before apply the configuration, always check the ConfigNumber to ensure it is the current one.

- Because the work of Lab4A, one group can be either shards sending or receiving, not both. So this simplify the work a lot. For servers sending out shards, it needs to wait for ACKs of all the send request before continue to end-configuration phase. Because Paxos is leader-less, each server of one group may send duplicate content, but it does not matter, the receiving servers can detect duplicates (details explain shortly) and still reply back ACK. Actually a leader-based solution may be more efficient, that is one leader sends out shards and notify all others if success.

- For servers receiving shards, they need to maintain an incoming-shards map, whenever it receives some incoming shards, it needs to Paxos propose and wire into Paxos log and update their data in application step. Reply ACK if any of these holds: configNum match and shards update sucessfully; configNum less than current one (this is old duplicates), configNum match but shards already updated (this is current duplicates). Reply ReTry if configNum is larger than current one, the sending servers are too much ahead and receiving server cannot accept, otherwise shards can be overwritten pre-maturally. So sending servers need to retry sending until get ACKs back. Shards received and updated succesfully can serve clients' request for query and updating. Unlike sending servers, if everything works fine, the first one of group serves the "leader" because all sending servers will try sending shards to this one. But once again, a leader-based solution can be more efficient, that is one leader receives shards and broadcast the data to all others.

- Actually all groups are applying re-configuration in a distributed lock-step way: no groups can be 2 versions ahead of the other (too far ahead shards senders will keep retry sending and too far ahead shards receivers will keep waiting for new incoming shards). It can be optimized that re-configuration happens in lock-step for shards instead of groups, from client-side, if the querying/updating shards are up-to-date, they can be fulfilled faster instead of waiting for the whole group ready. And also it will give more throughputs, but it is quite complex and beyond scope of this lab.

- For Server serving client's request, the client's configNum does not matter here, it can be much bigger than the server's one, as long as the shards are ready in the server, it can serve the requests. This will not impact the consistency of the data.

- In implementation, because both KV and reconfiguration operations are multiplexing into Paxos log, so the each log's entry structure needs to accomodate both cases. For request's clientId, it will be the server-id in terms of reconfiguration operations (the final bug is found because I did not putting clientIds at all); for request's requestId, it will be the configNum in terms of reconfiguration operations; for request's value, it will be the serilzied string value of reconfiguration data

## Lab 5 - Persistence

This lab caught two issues of previous KVPaxos:

- Each KVServer need to periodically bump (try to decided a NoOp operation on top of instances) so that its state machine is up-to-date and its paxos logs can be truncated and save space in persistance

- If KVServer cannt find its proposed value decided after some period, it needs to retry propose it (with potentially higher proposalNumber) to unblock the process; a scenario is server-1 try to propose a value, server-2 and server-3 ready to accept server-1's value, but when server-1 try to send accept request for this value, server-3 dead and server-2 proposes another value with higher proposal number, in this case, server-1's accept request will be rejected by server-2 because of lower proposal number. If no-one retries proposal, there is no progress, so either server-1 or server-2 needs to retry proposing its value. To reduce chance of livelock, server needs to wait for a random timeout (less than 2 seconds to pass test cases) before retry

Paxos logs need to be persisted whenever changes to its states and KVServer need to periodically snapshot its states, but it must first snapshot before notify paxos of any done indices, the paxos logs must cover the KVServer's states in case of failure that KVServer's states need to rebuilt from paxos logs.

In case of diskloss, KVServer need to request any live peer for most up-to-date KV and paxos states during initialization. The recipient KVServer need to bump its instances to ensure it gets the most up-to-date KV and paxos states before replying.
