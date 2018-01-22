# TiKV - HBase done right

![TiDB Family](https://pingcap.com/images/blog/sparkontikv.png)




----------
[x] 概览
[x] Storage - RocksDB
[ ] Replication - Raft protocol
[ ] Transaction - 2PC/MVCC
[ ] Schedule - PD
[ ] Monitoring - Prometheus && Grafana
[ ] Testing
[ ] SQL Layer



----------
## 概览

**HBase 出了什么问题**


- 可用性差
  - GC
  - 恢复时间长
- 无跨行事务 - Mistake of Jeaf Dean
  - 决定了它只合适做一个 KV 数据库

**需要考虑什么**


- 一致性：我们是否需要保证整个系统的线性一致性，还是能容忍短时间的数据不一致，只支持最终一致性。
- 稳定性：我们能否保证系统 7 x 24 小时稳定运行。系统的可用性是 4 个 9，还有 5 个 9？如果出现了机器损坏等灾难情况，系统能否做的自动恢复。
- 扩展性：当数据持续增多，能否通过添加机器就自动做到数据再次平衡，并且不影响外部服务。
- 分布式事务：是否需要提供分布式事务支持，事务隔离等级需要支持到什么程度。
----------



![](http://static.zybuluo.com/zyytop/rmudjvx02boh2g413hhkcqnu/1%E7%9A%84%E5%89%AF%E6%9C%AC.png)


**可用性**


- Rust
  - static language。
  - no GC。
  - Memory safe，avoid dangling pointer，memory leak。
  - Thread safe，no data race。
  - package manager。
  - C bindings，zero-cost。
  - not so easy to learn。
  - not so many libraries as Java or C++。
- 多副本（Raft）
  - 数据写入只有**大多数副本节点写入成功**才算成功。
  - 一个副本节点 down，可以切换到其他副本节点。

**跨行事务 -** 2PC

基于跨行事务，可以实现 SQL-based 的数据库。

**GRPC API**


- Get
- Scan
- BatchGet
- Prewrite
- Commit


----------
## Storage Stack
![Storage Stack](https://pingcap.com/images/blog/storage-stack1.png)

![Key Space](https://pingcap.com/images/blog/key-space.png)

![Storage Stack3](https://pingcap.com/images/blog/storage-stack3.png)

----------
## Leader-based Replication - Raft


- **CAP**: Choose Consistency or Availability when Partitioned
![](https://pingcap.com/images/blog-cn/raft-rocksdb.png)





## Transaction && MVCC


## SQL Layer
![SQL to Key-Value](https://pingcap.com/images/blog/sql-kv.png)



## 编码格式

DB


- `m + DBs + h + DB:[db_id]` => `TiDBInfo`
- `m + DB:[db_id] + h + Table:[tb_id]` => `TiTableInfo`

``` json
    {
     "id":130,
     "db_name":{"O":"global_temp","L":"global_temp"},
     "charset":"utf8","collate":"utf8_bin","state":5
    }
```

``` json
    {
       "id": 42,
       "name": {
          "O": "test",
          "L": "test"
       },
       "charset": "",
       "collate": "",
       "cols": [
          {
             "id": 1,
             "name": {
                "O": "c1",
                "L": "c1"
             },
             "offset": 0,
             "origin_default": null,
             "default": null,
             "type": {
                "Tp": 3,
                "Flag": 139,
                "Flen": 11,
                "Decimal": -1,
                "Charset": "binary",
                "Collate": "binary",
                "Elems": null
             },
             "state": 5,
             "comment": ""
          }
       ],
       "index_info": [],
       "fk_info": null,
       "state": 5,
       "pk_is_handle": true,
       "comment": "",
       "auto_inc_id": 0,
       "max_col_id": 4,
       "max_idx_id": 1
    }
```

**执行计划落地**

Table Scan/Index Scan => Selection/TopN/Aggr/Limit

http://andremouche.github.io/tidb/coprocessor_in_tikv.html


