

storage/mod -> Storage.start

- storage/txn/scheduler
  - storage/engine
    - rocksdb
      - raftstore/store/engine




### MVCC TXN



- `lock_key`: Put(key, (lock_type, short_value, primary, start_ts))=> CF_LOCK
- `unlock_key`: Delete(key) => CF_LOCK
- `put_value` : Put(key, Value) => CF_DEFAULT
- `delete_value`: Delete(key) => CF_DEFAULT
- `put_write`: Put(key, Value) => CF_WRITE
- `delete_write`: Delete(key) => CF_WRITE



`get`: MvccReader.get(key)



### Store



#### KV Engine DB



##### CF_RAFT 

**Region State:**

- Key: `0x01(LOCAL_PREFIX), 0x03(REGION_META_PREFIX), region_id, 0x01(REGION_STATE_SUFFIX)`  。到`0x01(LOCAL_PREFIX), 0x04(REGION_META_PREFIX+1)` 为止。

- Value:  

  ```protobuf
  message RegionLocalState {
      PeerState state = 1;
      metapb.Region region = 2;
  }
  ```

**Raft Apply State:**

- Key: `0x01(LOCAL_PREFIX), 0x02(REGION_RAFT_PREFIX), region_id,0x03(APPLY_STATE_SUFFIX)`

**Snapshot Raft State:**

- Key: `0x01(LOCAL_PREFIX), 0x02(REGION_RAFT_PREFIX), region_id,0x04(SNAPSHOT_RAFT_STATE_SUFFIX)`
- ​

当 RegionLocalState 中的 peer state 是 Tombstone 时，清除这些数据。

- 删除 KV DB中 该 region_id 的 Region State 数据。
- 删除 KV DB中该 region_id 的 Raft Apply State 数据。
- 删除 Raft DB 中该 region id 的 Raft Log 数据。
- 删除 Raft DB 中的该 region id 的 Raft State 数据。
- 将 Region State 的 peer state 置为 tombstone。？

当 RegionLocalState 中的 peer state 是 Applying 时，?



#### Raft Engine DB

**Raft State:**

- Key: `0x01(LOCAL_PREFIX), 0x02(REGION_RAFT_PREFIX), region_id, 0x02(RAFT_STATE_SUFFIX) `

- Value:

  ```protobuf
  message RaftLocalState {
      eraftpb.HardState hard_state = 1;
      uint64 last_index = 2;
  }
  ```

**Raft Log:**

- Key:  `0x01(LOCAL_PREFIX), 0x02(REGION_RAFT_PREFIX), region_id, 0x01(RAFT_LOG_SUFFIX),raft_index `