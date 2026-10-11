---
title: Working Mechanism of synchronous_standby_names in MogDB
date: 2024-06-05 11:05:50
tags: MogDB
---

In MogDB, the parameter `synchronous_standby_names` is used to configure synchronous replication settings. When this parameter is set to `'*'`（default setting）, it indicates that any available standby server can be used as a synchronous standby. This configuration allows any currently connected standby to be utilized as a synchronous standby without explicitly specifying the standby's name.

### Working Mechanism

When `synchronous_standby_names` is set to `'*'`, MogDB's synchronous replication mechanism selects the synchronous standby through the following steps:

1. **Initial Connection**: When the primary server starts or the parameter is changed, the primary server accepts all connected standby servers.

2. **Synchronous Standby Confirmation**:
   - The primary server sends a synchronization request to all connected standby servers.
   - Each standby server, upon receiving the request, confirms it and reports its status back to the primary server.

3. **Selecting the Synchronous Standby**:
   - The primary server selects the earliest responding standby server(s) as the synchronous standby.
   - This selection process is dynamic, meaning that if new standbys connect or current synchronous standbys disconnect, the primary server will reselect the synchronous standby.

### Priority and Behavior

- **No Priority Order**: Since `'*'` denotes any standby, all standbys have the same priority. The primary server simply selects the earliest responding standby as the synchronous standby.
- **Dynamic Adjustment**: If a synchronous standby disconnects, the primary server automatically selects the next responding standby as the new synchronous standby, ensuring the continuity and reliability of synchronous replication.
- **Concurrent Management**: If multiple standbys connect simultaneously, the primary server can handle these concurrent connections and select the synchronous standby based on the confirmation of synchronization requests.

### Configuration Example and Usage

Suppose we have one primary server and three standby servers (`standby1`, `standby2`, and `standby3`). In the configuration file `postgresql.conf`, set `synchronous_standby_names` to `'*'`:

```plaintext
synchronous_standby_names = '*'
```

#### Scenario Analysis

1. **Selection at Startup**:
   - When the primary server starts, all connected standby servers send heartbeat signals and wait for the primary server's synchronization request.
   - The primary server selects the earliest responding standby as the synchronous standby.

2. **Runtime Changes**:
   - If the current synchronous standby `standby1` disconnects, the primary server automatically selects the next responding standby `standby2` as the new synchronous standby.
   - If a new standby `standby3` connects to the primary server and there is no current synchronous standby, the primary server selects `standby3` as the synchronous standby.

#### Dynamic Adjustment

If we start the primary server and three standby servers, setting `synchronous_standby_names = '*'`, the following state transitions are possible:

- **Initial State**: All standbys (`standby1`, `standby2`, and `standby3`) connect, and the primary server selects the earliest responding standby `standby1` as the synchronous standby.
- **`standby1` Disconnects**: The primary server automatically selects the next responding standby `standby2` as the new synchronous standby.
- **New Standby Connection**: A new standby `standby4` connects; the primary server will not change the current synchronous standby unless `standby2` disconnects.

### Summary

When `synchronous_standby_names` is set to `'*'`, MogDB dynamically selects any currently available standby as the synchronous standby. This provides a flexible and highly available synchronous replication mechanism without requiring administrators to manually specify the standby names. The selection process is based on standby response times and automatically adjusts during runtime to ensure the continuity and reliability of synchronous replication.
