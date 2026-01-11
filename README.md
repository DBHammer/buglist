# Bugs found in Database Management Systems

We have successfully discovered 49 bugs (26 fixed, 39 confirmed and 10 open reported) from real-world production-level DBMSs, including 5 bugs in MySQL, 2 bugs in PostgreSQL, 20 bugs in TiDB, 3 bugs in OpenGauss, 3 bugs in Oceanbase, and 16 bugs in DB-X, a commercial DBMS whose name is anonymized at the request of our collaborator.

We are thankful to the DBMS developers for responding to our bug reports and fixing the bugs that we found. Because the nondeterministic interleavings among operations challenges the reproducibility of the isolation-related bugs, there are 10 bugs that can not be reproduced but are open-reported. In the future, we will aim to the research question about reducing bug cases such that the bug can be easily reproduced.

We have anonymized the corresponding issue and other personal information due to the double-blind reviewing constraint.

[TOC]



## Confirmed bugs

## TiDB

### Isolation-related Bugs

#### Bug#1 Dirty write in pessimistic transaction mode

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**     |
| ----------------------- | ------------------------------------------------------------ | ------------- |
| Mode Setting            | Set global tidb\_txn\_mode = 'pessimistic';                  | Success       |
| Schema Creation         | Create table table\_7\_2(a int primary key, b int, c double); | Success       |
| Database Initialization | Insert into table\_7\_2 values(676,-5012153, 2240641.4);     | Success       |
| 739                     | Begin transaction;                                           | Success       |
| 739                     | Update table\_7\_2 set b=-5012153, c=2240641.4 where a=676   | Success       |
| 723                     | Begin transaction;                                           | Success       |
| **723**                 | **Update table\_7\_2 set b=852150 where a=676**              | **Success X** |
| 739                     | Commit                                                       | Success       |

**Bug Description**

It first set transaction mode as pessimistic. Transaction 739 updates a record 676 in table\_ 7\_2 and holds an exclusive lock on the record 676. Before transaction 739 releases the exclusive lock on record 676, transaction 723 also successfully acquires an exclusive lock on record 676 and updates it, which results dirty write anomalies should be avoided in pessimistic transaction mode as described in TiDB official website.

#### Bug#2 Read inconsistency in snapshot isolation

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                        |
| ----------------------- | ------------------------------------------------------------ | -------------------------------- |
| Schema Creation         | Create table table\_7\_2(primarykey int primary key, attribute1 double,attribute6 double); | Success                          |
| Database Initialization | Insert into table\_7\_2 values(3873, 0.213, 0.234);          | Success                          |
| 904                     | Begin transaction;                                           | Success                          |
| 904                     | Update table\_8\_2 set attribute6=-0.386 where primarykey=3873 | Success                          |
| 904                     | Commit                                                       | Success                          |
| 914                     | Set @@global.tx\_isolation='REPEATABLE-READ';                | Success                          |
| 907                     | Begin transaction;                                           | Success                          |
| 907                     | Update table\_8\_2 set attribute6=0.484 where primarykey=3873 | Success                          |
| 907                     | Commit                                                       | Success                          |
| **914**                 | **Select attribute6 from table\_8\_2 where primarykey=3873** | **Success(attribute6=-0.368) X** |

**Bug Description**

There are two historical versions on attribute6 of record 3873 in table\_ 8\_2. The first one is -0.386 created by transaction 904; the second one is 0.484 created by transaction 907. Since the select operation of transaction 914 happens after the committing of transaction 907, transaction 914 should see the second one, i.e., 0.484, instead of the first one, i.e., -0.368. However, the select operation of transaction 914 returns the first version -0.368, which indicates there is a defect about the implementation of consistent read in TiDB.

#### Bug#3 Violating mutual exclusion

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State** |
| ----------------------- | ------------------------------------------------------------ | --------- |
| Schema Creation         | create table t1(a int primary key, b  int); <br>create table t2(a int primary key, b int,  constraint fk1 foreign key(b) references t1(a));  <br/>create view view0(t2_a,t2_b,t1_b) as  select t2.a,t2.b,t1.b from t2,t1 where t2.b=t1.a; | Success   |
| Database Initialization | insert into t1 values(1,2);  <br/>insert into t1 values(2,3);  <br/>insert into t1 values(3,4);  <br/>insert into t1 values(4,5);  <br/>insert into t1 values(5,6);     <br/>insert into t2 values(1,2);  <br/>insert into t2 values(2,3);  <br/>insert into t2 values(3,4);  <br/>insert into t2 values(4,5);  <br/>insert into t2 values(5,1); | Success   |

| Operation ID | **Session1**                  | **Session2**                                        | **State**                                 |
| ------------ | ----------------------------- | --------------------------------------------------- | ----------------------------------------- |
| 1            | Begin transaction;            |                                                     | Success                                   |
| 2            |                               | Begin transaction;                                  | Success                                   |
| 3            | update t1 set b=12 where a=1; |                                                     | Success                                   |
| **4**        |                               | **select \* from view0 where t2\_a\>3 for update;** | **Success<br>Query Result(5,1,2)(4,5,6)** |
| 5            |                               | Commit;(Success)                                    | Success                                   |
| 6            | Commit;(Success)              |                                                     | Success                                   |

**Bug Description**

Operation 3 holds an exclusive lock on a record 1 in table t1 until operation 6 releases the lock. Due to the nature of exclusive locks, operation 4 attempts to acquire an exclusive lock on record 1 in table t1, which should be blocked. However, TiDB grants operation 4 an exclusive lock on record 1 in table t1, which indicates a locking violation.

#### Bug#4 Unnecessary locking a non-existing record

**Test Case**

| **Transaction ID**      | **Operation Detail**                      | **State** |
| ----------------------- | ----------------------------------------- | --------- |
| Schema Creation         | Create table t(a int primary key, b int); | Success   |
| Database Initialization |                                           | Success   |

| Operation ID | **Session1**                  | **Session2**                   | **State**                  |
| ------------ | ----------------------------- | ------------------------------ | -------------------------- |
| 1            | Begin transaction;            |                                | Success                    |
| 2            |                               | Begin transaction;             | Success                    |
| 3            | Update t set b=314 where a=1; |                                | Success with row count = 0 |
| **4**        |                               | **Insert into t values(1,3);** | **blocking X**             |
| 5            | Commit;                       |                                | Success                    |
| 6            |                               | Insert into t values(1,3);     | Success with row count = 1 |
| 7            |                               | Commit;                        | Success                    |


**Bug Description**

After investigating TiDB official website, the write operations of TiDB only locks the record that satisfies the conditions. Notice that the read operation of TiDB can avoid the phantom by the way of MVCC.

However, as shown in above test case, the update operation (Operation ID=3) locks a non-existing record that dose not satisfy its where condition. Additionally, the update operation (Operation ID=3) blocks the insertion operation of another transaction (Operation ID=6), which may lead to some performance issues about locking.

#### Bug#5 Query in transaction may return rows with same unique index column value

See [Query in transaction may return rows with same unique index column value · Issue #24195 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/24195) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State** |
| ----------------------- | ------------------------------------------------------------ | --------- |
| Mode Setting            | Set global tidb\_txn\_mode = 'pessimistic';                  | Success   |
| Schema Creation         | create table t (c1 varchar(10), c2 int, c3 char(20), primary key (c1, c2)); | Success   |
| Database Initialization | insert into t values ('tag', 10, 't'), ('cat', 20, 'c');     | Success   |
| 2                       | Begin transaction;                                           | Success   |
| 2                       | update t set c1=reverse(c1) where c1='tag';                  | Success   |
| 4                       | Begin transaction;                                           | Success   |
| 4                       | insert into t values('dress',40,'d'),('tag', 10, 't');       | Success  |
| 2                       | Commit                                                       | Success   |
| 4                       | select \* from t use index(primary) order by c1,c2;          |           |

When primary key is clustered index, query returns

| C1    | C2   | C3   |
| ----- | ---- | ---- |
| cat   | 20   | c    |
| dress | 40   | d    |
| tag   | 10   | t    |

When primary key is nonclustered index, query returns

| C1    | C2   | C3   |
| ----- | ---- | ---- |
| cat   | 20   | c    |
| dress | 40   | d    |
| tag   | 10   | t    |
| tag   | 10   | t    |

**Bug Description**

The above test case runs on pessimistic transaction mode in TiDB. Session 1 initializes a table with two record. One of them is ('tag','10',t). Session 2 acquires a exclusive lock to update the record ('tag','10',t) as ('gat','10',t). Before update committed, session 4 successfully acquires a exclusive lock to insert a new record ('tag','10',t), which violates the principle of long duration lock.

After that, session 4 issues a query to see all record in table t. When primary key is clustered index, query return the newly inserted record ('tag','10',t). However, when primary key is nonclustered index, query return two identical records ('tag','10',t), which indicates a bug hidden in TiDB. We have reported such a test case in TiDB community. TiDB developers have acknowledged it and fixed it.

#### Bug#6 Select under repeatable read isolation level returns stale version of data

 [SELECT under repeatable read isolation level returns stale version of data · Issue #36718 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/36718) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                      |
| ----------------------- | ------------------------------------------------------------ | ------------------------------ |
| Schema Creation         | create table table0 (pkId integer, pkAttr0 integer, pkAttr1 integer, pkAttr2 integer, coAttr0\_0 integer, primary key(pkAttr0, pkAttr1, pkAttr2)); | Success                        |
| Database Initialization | insert into t values (412,409,258,17702);                    | Success                        |
| 281                     | SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ      | Success                        |
| 282                     | SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE         | Success                        |
| 282                     | Begin transaction;                                           | Success                        |
| 282                     | update table0 set coAttr0\_0 = 40569 where ( pkAttr0 = 412 ) and ( pkAttr1 = 409 ) and ( pkAttr2 = 258 ); | Success                        |
| 282                     | commit;                                                      | Success                        |
| 281                     | Begin transaction;                                           | Success                        |
| 281                     | select pkAttr0, pkAttr1, pkAttr2, coAttr0\_0 from table0 where ( coAttr0\_0 = 89665 ) | Success                        |
| 281                     | update table0 set coAttr0\_0 = 40569 where ( pkAttr0 = 412 ) and ( pkAttr1 = 409 ) and ( pkAttr2 = 258 ) | Success                        |
| **281**                 | **select pkAttr0, pkAttr1, pkAttr2, coAttr0\_0 from table0 where ( coAttr0\_0 = 17702 )** | **Return (412,409,258,17702)** |
| 281                     | commit                                                       | Success                        |

**Bug Description**

Though Session-282 update the only row into (412,409,258, 40569). The last select query still returns a stale version of data for Key\<412,409,258\>, i.e. (412,409,258,17702).

#### Bug#7 Transaction can't read its own update in repeatable read.

See  [Transaction can't read its own update in repeatable read. · Issue #42487 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/42487) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                                     |
| ----------------------- | ------------------------------------------------------------ | --------------------------------------------- |
| Schema Creation         | create table table1 (pkId integer, pkAttr0 integer, pkAttr1 integer, pkAttr2 integer, coAttr0\_0 integer, primary key(pkAttr0, pkAttr1, pkAttr2) NONCLUSTERED); | Success                                       |
| Database Initialization | insert into table1 values (1,1,1,1,1);                       | Success                                       |
| 1                       | SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ      | Success                                       |
| 2                       | SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ      | Success                                       |
| 2                       | Begin transaction;                                           | Success                                       |
| 1                       | Begin transaction;                                           | Success                                       |
| 2                       | update table1 set coAttr0\_0 = 2 where pkAttr0=1 and pkAttr1=1 and pkAttr2=1; | Success                                       |
| 2                       | commit                                                       | Success                                       |
| 1                       | update table1 set coAttr0\_0 = 2 where pkAttr0=1 and pkAttr1=1 and pkAttr2=1; | Success                                       |
| **1**                   | **select \* from table1 where pkAttr0=1 and pkAttr1=1 and pkAttr2=1;** | **Return (1,1,1,1,2) instead of (1,1,1,1,1)** |

**Bug Description**

The last select query of session1 cannot get its own update.

When we update a different value in session2, the last query can get correct value, i.e., when we execute the transaction below:

It seems that some opt(skipping update) when two transactions update a same value may go wrong because the select and update hold different snapshot.

#### Bug#8 DROP DATABASE is not blocked while executing transaction.

See  [DROP DATABASE is not blocked while executing transaction · Issue #46943 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/46943) 

**Test Case**

| **Transaction ID** | **Operation Detail**     | **State** |
| ------------------ | ------------------------ | --------- |
| Schema Creation    | create table t (a int);  | Success   |
| 2                  | Begin transaction;       | Success   |
| 2                  | insert into t values(1); | Success   |
| 1                  | drop database test;      | No block  |

**Bug Description**

DROP DATABASE can be executed without blocking. And the idling transaction can also be committed without warning/error.

We tested DROP TABLE in TiDB and found this DDL will be blocked. And we also test DROP DATABASE in MySQL, which is also be blocked. So it seems obviously that DROP DATABASE should also be blocked.

Maybe locking all tables while altering database can help.

#### Bug#9 Transaction cannot be committed/rollbacked via binary protocol after TTL manager timed out

See  [Transaction cannot be committed/rollbacked via binary protocol after TTL manager timed out · Issue #49151 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/49151) 
**Test Case**

```js
  const c = await mysql.createConnection({ host: TIDB_HOST, port: 4000, user: "root", database: 'test' });
  try {
    await c.query("set @@tidb_general_log=1");
    await c.query("drop table if exists t1");
    await c.query("create table t1 (a int, b int)");
    await c.query("insert into t1 values (3, 2), (2, 3)");
    log("show config", await pp(c.query("show config where name like '%max-txn-ttl'")));
    await c.query("begin");
    await c.query("update t1 set a=4 where a=3");
    log("select(1):", await pp(c.query("select * from t1")));
    log("wait 45 seconds");
    await sleep(45*1000);
    log("continue");
    log("select(2):", await pp(c.query("select * from t1")));
    log("rollback(bin):", await pp(c.execute("rollback")));
    log("commit(bin):  ", await pp(c.execute("commit")));
    log("commit(txt):  ", await pp(c.query("commit")));
  } finally {
    await close(c);
  }
```

**Bug Description**

Both commit and rollback fail.

```bash
2023-12-04T10:49:43.457Z show config [{"Type":"tidb","Instance":"127.0.0.1:4000","Name":"performance.max-txn-ttl","Value":"30000"}]
2023-12-04T10:49:43.467Z select(1): [{"a":4,"b":2},{"a":2,"b":3}]
2023-12-04T10:49:43.467Z wait 45 seconds
2023-12-04T10:50:28.476Z continue
2023-12-04T10:50:28.478Z select(2): TTL manager has timed out, pessimistic locks may expire, please commit or rollback this transaction
2023-12-04T10:50:28.481Z rollback(bin): TTL manager has timed out, pessimistic locks may expire, please commit or rollback this transaction
2023-12-04T10:50:28.482Z commit(bin):   TTL manager has timed out, pessimistic locks may expire, please commit or rollback this transaction
2023-12-04T10:50:28.486Z commit(txt):   {"fieldCount":0,"affectedRows":0,"insertId":0,"info":"","serverStatus":2,"warningStatus":0}
```

It's because binary protocal cannot finish an aborted transaction.



### Other types of bugs

#### Bug#10 Schema version check error

**Test Case**

| **Transaction ID** | **Operation Detail**                                         | **State**                                     |
| ------------------ | ------------------------------------------------------------ | --------------------------------------------- |
|                    | Drop db0.table\_1\_2                                         | Success                                       |
| **723**            | **Update db1.table\_5\_1 set attribute2=8132130 where primarykey=6123** | **Exception:Information schema is changed X** |

**Bug Description**

During verifying the read committed isolation level recently developed by TiDB team, we find there is a checking schema problem. Specifically, the first line in above table modifies db0's schema information, while the second line in transaction 723 modifies db1's table with exception "information schema is changed". However, there is no modification on db1's schema information, which indicates a bug hidden in checking schema version.

#### Bug#11 Timestamp acquisition mechanism error in read committed

**Test Case**

| **Transaction ID** | **Operation Detail**                                 | **State**                |
| ------------------ | ---------------------------------------------------- | ------------------------ |
| **232**            | **Select \* from table\_2\_1 where primarykey=4323** | **Stall(never respond)** |

**Bug Description**

As definition, read committed isolation level sees a consistent view of database as of the beginning of an operation. Thus, each read operation in read committed isolation level should acquire a timestamp. Under the read committed isolation level recently developed by TiDB team, in order to optimize the performance of timestamp acquisition, asynchronous timestamp acquisition mechanism is adopted, but there are internal problems in this mechanism, as shown in the above table. Specifically, the read operation in above table never be responded from timestamp sever. We have help their developers fix this bugs.

#### Bug#12 Update BLOB data error

**Test Case**

| **Operation ID**        | **Operation Detail**                                         | **State**                             |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------- |
| Schema Creation         | create table t(pk0 int, pk1 int, pk2 int, attr3 blob);       | Success                               |
| Database Initialization | insert into t values(1,1,1,null);                            |                                       |
| 1                       | Update t set attr3=FILE("./data\_case/obj/12obj\_file.obj") where pk0=1 and pk1=1 and pk2= 1 | Success                               |
| 2                       | Update t set attr3=FILE("./data\_case/obj/12obj\_file.obj") and other column where pk0 = 1 and pk1 = 1 and pk2 = 1 | Success                               |
| **3**                   | **Select attr3 from t where pk0 = 1 and pk1 = 1 and pk2 = 1 for update** | **Success <br>Return attr3 = NULL X** |

**Bug Description**

For BLOB data type, when the new value and the old value written by the update operation are for the same binary file, the value actually written is null and success is returned, which indicates a BLOB-related bug hidden in TiDB.

#### Bug#13 Bug in Start Transaction

**Test Case**

| **Transaction ID** | **Operation Detail**                                     | **State** |
| ------------------ | -------------------------------------------------------- | --------- |
| **1**              | **Start transaction read only with consistent snapshot** | **fail**  |

**Bug Description**

As for the start transaction statement, the TiDB official websiate shows that it supports the keywords "with consistent snapshot" and "read only". However, in fact, we found that TiDB cannot support these two keywords at the same time.

#### Bug#14 JDBC ResultSetMetaData.getColumnName for view query returns the attribute name defined in the table instead of the one defined in the view

See  [JDBC ResultSetMetaData.getColumnName for view query returns the attribute name defined in the table instead of the one defined in the view · Issue #24227 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/24227) 

**Test Case**

 Just query a view with renamed attribute and print `ResultSetMetaData.getColumnName`. 

```java
import lombok.Cleanup;

import java.sql.*;

public class Main {
    public static void main(String[] args) throws SQLException {
        String url = "jdbc:mysql://localhost:3306?serverTimezone=UTC&useServerPrepStmts=true&cachePrepStmts=true";
        String username = "root";
        String password = "root";

        String dropDatabase = "drop database if exists RSMDTestDB;";
        String createDatabase = "create database RSMDTestDB;";
        String useDatabase = "use RSMDTestDB;";
        String createTable0 = "create table table0(t0c0 int, t0c1 int, t0c2 int);";
        String createView0 = "create view view0(v0c0, v0c1, v0c2) as select t0c0, t0c1, t0c2 from table0;";
        String selectView0 = "select * from view0;";
        String selectView0WithAlias = "select v0c0 as alias_v0c0, v0c1 as alias_v0c1, v0c2 as alias_v0c2 from view0;";

        @Cleanup
        Connection conn = DriverManager.getConnection(url, username, password);
        Statement stat = conn.createStatement();
        stat.execute(dropDatabase);
        stat.execute(createDatabase);
        stat.execute(useDatabase);
        stat.execute(createTable0);
        stat.execute(createView0);

        // print DBMS info
        DatabaseMetaData dbMD = conn.getMetaData();
        System.out.println(dbMD.getDatabaseProductVersion());

        // select from view without alias
        ResultSet rs = stat.executeQuery(selectView0);
        ResultSetMetaData md = rs.getMetaData();
        System.out.println("SELECT WITHOUT ALIAS");
        System.out.printf("%-10s\t%-10s\t%-10s\n", "Index", "ColName", "ColLabel");
        for(int ind=1; ind <= md.getColumnCount(); ind++) {
            System.out.printf("%-10d\t%-10s\t%-10s\n", ind, md.getColumnName(ind), md.getColumnLabel(ind));
        }
        rs.close();

        // select from view with alias
        rs = stat.executeQuery(selectView0WithAlias);
        md = rs.getMetaData();
        System.out.println("SELECT WITH ALIAS");
        System.out.printf("%-10s\t%-10s\t%-10s\n", "Index", "ColName", "ColLabel");
        for(int ind=1; ind <= md.getColumnCount(); ind++) {
            System.out.printf("%-10d\t%-10s\t%-10s\n", ind, md.getColumnName(ind), md.getColumnLabel(ind));
        }
        rs.close();

        // drop database
        stat.execute(dropDatabase);
    }
}
```

**Bug Description**

For MySQL, ResultSetMetaData.getColumnName returns the name of a column defined in the view, and ResultSetMetaData.getColumnLabel returns the alias given in the query.

However, in TiDB, ResultSetMetaData.getColumnName returns the name of a column defined in the table instead of in the view, while ResultSetMetaData.getColumnLabel returns the alias given in the query.

However, considering that when users query a view, they treat it as an abstract table, and do not care whether it is a table or a view. Perhaps it is more reasonable to return the name of the attribute listed in the view definition. This is the behavior of MySQL.

Moreover, if the user uses the as keyword to alias a attribute when querying the view, it seems that the attribute name defined in the view cannot be obtained from ResultSetMetaData.

#### Bug#15 Query Error in information\_schema.slow\_query

See  [Query Error in information_schema.slow_query · Issue #28069 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/28069) 

**Test Case**

| **Transaction ID** | **Operation Detail**                                         | **State**              |
| ------------------ | ------------------------------------------------------------ | ---------------------- |
| Schema Creation    | Use information\_schema                                      | Success                |
| 1                  | select Txn\_start\_ts,time from slow\_query where time \> '2021-08-13 16:18:37.313976' limit 10; | Success returns 1 rows |
| **1**              | **select Txn\_start\_ts,time from slow\_query where time \> '2021-08-10 16:18:37.313976' limit 10;** | **Empty set**          |

**Bug Description**

Since the last query select a larger scale(\>2021-08-10) tha the first one (2021-08-13), it should not return less result.

#### Bug#16 Min/Max function applied on partition key causes more than 5000x performance degradation

See  [Min/Max function applied on partition key causes more than 5000x performance degradation · Issue #41462 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/41462) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                |
| ----------------------- | ------------------------------------------------------------ | ------------------------ |
| Schema Creation         | CREATE TABLE IF NOT EXISTS test\_table ( a INT NOT NULL, b INT NOT NULL, c INT NOT NULL, PRIMARY KEY (a,b,c)) PARTITION BY HASH(a) PARTITIONS 5; | Success                  |
| Database Initialization | Insert 10k tuples.For any tuple, column a and b have same value, generated from [1-10], and column c is increasing by 1. | Success                  |
| **1**                   | **select min(a) from test\_table;**                          | **Success(1min 54 sec)** |
| **1**                   | **select max(a) from test\_table;**                          | **Success(2min 3 sec)**  |
| 1                       | select distinct(a) from test\_table order by a limit 1;      | Success(21ms)            |
| 1                       | select min(b) from test\_table;                              | Success(14ms)            |

**Bug Description**

Expect to see that four queries should take a similar amount of time.

But first two queries took a very long time to execute(1 min 54.51 sec, 2 min 3.46 sec). While the last two act quickly(21.3ms, 14.4ms). There is a 5000x performance degradation between them.

And we found that the first two queries used Limit but no TopN on tikv, which may lead to a large data transfer from tikv to root. This may make query slow(even on the same machine).

#### Bug#17 Create index is blocked when no /tmp/tidb/tmp\_ddl-4000.

See  [Create index is blocked when no /tmp/tidb/tmp_ddl-4000 · Issue #45624 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/45624) 

**Test Case**

| **Transaction ID**  | **Operation Detail**       | **State** |
| ------------------- | -------------------------- | --------- |
| Schema Creation     | Create table t(a int)      | Success   |
| **Schema Creation** | **Create index t1 on (a)** | **block** |

**Bug Description**

The ddl is blocked. When kill this ddl, return error mes cannot get disk capacity at /tmp/tidb/tmp\_ddl-4000: no such file or directory. And there is no tidb under /tmp. Index can be created after making dir /tmp/tidb/tmp\_ddl-4000 manually.

It is because TiDB does not build such dir automatically after checking.

#### Bug#18 Join between blob type with matching returns incorrect result.

See  [Join between blob type with matching returns incorrect result. · Issue #50393 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/50393) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State** |
| ----------------------- | ------------------------------------------------------------ | --------- |
| Schema Creation         | create table t2(a blob);<br> create table t3(a blob);        | Success   |
| Database Initialization | insert into t2 values(0xC2A0);<br> insert into t3 values(0xC2); | block     |
| **1**                   | **select * from t2,t3 where t2.a like concat("%",t3.a,"%");** |           |

In MySQL(v8.0.33), the last query returns one row.

```sql
mysql> select * from t2,t3 where t2.a like concat("%",t3.a,"%");
+------------+------------+
| a          | a          |
+------------+------------+
| 0xC2A0     | 0xC2       |
+------------+------------+
1 row in set (0.01 sec)
```

However, TiDB returns an empty set.

```sql
> select * from t2,t3 where t2.a like concat("%",t3.a,"%");
Empty set (0.00 sec)
```

 It looks weird since in the case below, TiDB works well. 

```sql
create table t2(a blob);
create table t3(a blob);
insert into t2 values(0xC2A020);
insert into t3 values(0xC2A0);
select * from t2,t3 where t2.a like concat("%",t3.a,"%");
```

In this case, two DBMSs both returns one row. The charsets of TiDB and MySQL are the same(utf8mb4). 

This bug may be relative to how match operation(like) compare two objects. When taking two bytes(as java's char) as a compare unit, the second object in the first case are different from the first one instead of contained by the first object. However, in the second case, the second object is contained by the first one. Moreover, when taking one byte as a compare unit(as c's char), the second object is contained by the first one in both cases.

## MySQL

### Isolation-related Bugs

#### Bug#19 Select under repeatable read isolation level returns stale version of data

See  [MySQL Bugs: #108015: select under repeatable read isolation level returns stale version of data](https://bugs.mysql.com/bug.php?id=108015) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                      |
| ----------------------- | ------------------------------------------------------------ | ------------------------------ |
| Schema Creation         | create table table0 (pkId integer, pkAttr0 integer, pkAttr1 integer, pkAttr2 integer, coAttr0\_0 integer, primary key(pkAttr0, pkAttr1, pkAttr2)); | Success                        |
| Database Initialization | insert into t values (412,409,258,17702);                    | Success                        |
| 281                     | SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ      | Success                        |
| 282                     | SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE         | Success                        |
| 282                     | Begin transaction;                                           | Success                        |
| 282                     | update table0 set coAttr0\_0 = 40569 where ( pkAttr0 = 412 ) and ( pkAttr1 = 409 ) and ( pkAttr2 = 258 ); | Success                        |
| 282                     | commit;                                                      | Success                        |
| 281                     | Begin transaction;                                           | Success                        |
| 281                     | select pkAttr0, pkAttr1, pkAttr2, coAttr0\_0 from table0 where ( coAttr0\_0 = 89665 ) | Success                        |
| 281                     | update table0 set coAttr0\_0 = 40569 where ( pkAttr0 = 412 ) and ( pkAttr1 = 409 ) and ( pkAttr2 = 258 ) | Success                        |
| **281**                 | **select pkAttr0, pkAttr1, pkAttr2, coAttr0\_0 from table0 where ( coAttr0\_0 = 17702 )** | **Return (412,409,258,17702)** |
| 281                     | commit                                                       | Success                        |

**Bug Description**

Though Session-282 update the only row into (412,409,258, 40569). The last select query still returns a stale version of data for Key\<412,409,258\>, i.e. (412,409,258,17702).

#### Bug#20 Two parallel threads trigger error code '1032 Can't find record in 'table'

See  [MySQL Bugs: #103891: Two parallel threads trigger error code '1032 Can't find record in 'table'](https://bugs.mysql.com/bug.php?id=103891) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                                        |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------ |
| Schema Creation         | create table t(pkattr0 integer, coattr1 varchar(100), coattr2 varchar(100), coattr3 varchar(100), coattr4 integer, coattr5 varchar(100), coattr6 varchar(100), primary key(pkattr0)); | Success                                          |
| Database Initialization | delimiter \$<br>create procedure p()<br/>begin<br/>declare done int default false;<br/>declare v\_pkattr0 integer;<br/>declare v\_coattr1 varchar(100);<br/>declare v\_coattr2 varchar(100);<br/>declare v\_coattr3 varchar(100);<br/>declare v\_coattr4 varchar(100);<br/>declare v\_coattr5 varchar(100);<br/>declare v\_coattr6 varchar(100);<br/>declare cur1 cursor for select pkattr0, coattr1, coattr2, coattr3, coattr4, coattr5, coattr6 from t order by pkattr0, coattr1, coattr2, coattr3, coattr4, coattr5, coattr6;<br/>-- declare continue handler for sqlexception begin end;<br/>declare continue handler for not found set done = true;<br/>repeat<br/>    set session transaction isolation level read uncommitted;<br/>    if rand()\<0.5 <br/>          then start transaction; <br/>    end if;<br/>open cur1;<br/>     read\_loop: loop<br/>         fetch cur1 into v\_pkattr0, v\_coattr1, v\_coattr2, v\_coattr3, v\_coattr4, v\_coattr5, v\_coattr6;<br/>         if done then<br/>             leave read\_loop;<br/>         end if;<br/>      end loop;<br/>close cur1;<br/>if rand()\<0.5 then commit; end if;<br/>if rand()\<0.5 then start transaction; end if;<br/>if rand()\<0.5 then delete from t where pkattr0 = 1; end if;<br/>if rand()\<0.5 then commit; end if;set session transaction isolation level read uncommitted;<br/>if rand()\<0.5 then start transaction; end if;<br/>if rand()\<0.5 then replace into t(pkattr0, coattr1, coattr2, coattr3, coattr4, coattr5, coattr6) values("1", "varchar1", "varchar2", "varchar3", "2021", "varchar5", "varchar6"); end if;<br/>if rand()\<0.5 then commit; end if;<br/>until 1=2 end repeat; <br/>end ​\$ <br/>delimiter ; | Success                                          |
| 1                       | call p();                                                    | Success                                          |
| 2                       | call p();                                                    | Success                                          |
| **3**                   | **call p();**                                                | **ERROR 1032 (HY000): Can't find record in 't'** |

**Bug Description**

The expected result should not throw '1032' error.

## Oceanbase

### Other types of bugs

#### Bug#21 Create View Error

**Test Case**

| **Transaction ID**  | **Operation Detail**                                         | **State**                        |
| ------------------- | ------------------------------------------------------------ | -------------------------------- |
| Schema Creation     | create table table0(pkId integer,pkAttr0 integer,pkAttr1 integer,pkAttr2 integer,pkAttr3 integer,pkAttr4 integer,coAttr0\_0 integer,coAttr0\_1 decimal(10, 0),coAttr0\_2 varchar(100),primary key (pkAttr0, pkAttr1, pkAttr2, pkAttr3, pkAttr4)); | Success                          |
| Schema Creation     | alter table table0 add index index\_pk(pkAttr0, pkAttr1, pkAttr2, pkAttr3, pkAttr4); | Success                          |
| Schema Creation     | create table table1(pkId integer,pkAttr0 integer,pkAttr1 integer,coAttr0\_0 varchar(100),coAttr0\_1 integer,coAttr0\_2 varchar(100),primary key (pkAttr0, pkAttr1)); | Success                          |
| Schema Creation     | alter table table1 add index index\_pk(pkAttr0, pkAttr1);    | Success                          |
| Schema Creation     | alter table table1 add index index\_commAttr0(coAttr0\_0, coAttr0\_1, coAttr0\_2); | Success                          |
| Schema Creation     | create table table2(pkId integer,pkAttr0 integer,pkAttr1 integer,pkAttr2 integer,pkAttr3 integer,pkAttr4 integer,pkAttr5 integer,pkAttr6 integer,coAttr0\_0 decimal(10, 0),coAttr0\_1 varchar(100),coAttr0\_2 varchar(100),fkAttr0\_0 integer,fkAttr0\_1 integer,fkAttr0\_2 integer,fkAttr0\_3 integer,fkAttr0\_4 integer,fkAttr1\_0 integer,fkAttr1\_1 integer,primary key (pkAttr0, pkAttr1, pkAttr2, pkAttr3, pkAttr4, pkAttr5, pkAttr6),foreign key (fkAttr0\_0, fkAttr0\_1, fkAttr0\_2, fkAttr0\_3, fkAttr0\_4) references table0 (pkAttr0, pkAttr1, pkAttr2, pkAttr3, pkAttr4),foreign key (fkAttr1\_0, fkAttr1\_1) references table1 (pkAttr0, pkAttr1)); | Success                          |
| Schema Creation     | alter table table2 add index index\_pk(pkAttr0, pkAttr1, pkAttr2, pkAttr3, pkAttr4, pkAttr5, pkAttr6); | Success                          |
| Schema Creation     | alter table table2 add index index\_fk0(fkAttr0\_0, fkAttr0\_1, fkAttr0\_2, fkAttr0\_3, fkAttr0\_4); | Success                          |
| Schema Creation     | alter table table2 add index index\_fk1(fkAttr1\_0, fkAttr1\_1); | Success                          |
| **Schema Creation** | **create view view2 (pkAttr0, pkAttr1, pkAttr2, pkAttr3, pkAttr4, pkAttr5, pkAttr6, fkAttr0\_0, fkAttr0\_1, fkAttr0\_2, fkAttr0\_3, fkAttr0\_4, fkAttr1\_0, fkAttr1\_1, coAttr0\_0, coAttr0\_1, coAttr0\_2, coAttr1\_0, coAttr1\_1, coAttr1\_2)asselect table2.pkAttr0,table2.pkAttr1,table2.pkAttr2,table2.pkAttr3,table2.pkAttr4,table2.pkAttr5,table2.pkAttr6,table2.fkAttr0\_0,table2.fkAttr0\_1,table2.fkAttr0\_2,table2.fkAttr0\_3,table2.fkAttr0\_4,table2.fkAttr1\_0,table2.fkAttr1\_1,table2.coAttr0\_0,table2.coAttr0\_1,table2.coAttr0\_2,table0.coAttr0\_0,table0.coAttr0\_1,table0.coAttr0\_2from table2,table0where table2.fkAttr0\_0 = table0.pkAttr0and table2.fkAttr0\_1 = table0.pkAttr1and table2.fkAttr0\_2 = table0.pkAttr2and table2.fkAttr0\_3 = table0.pkAttr3and table2.fkAttr0\_4 = table0.pkAttr4** | **Error: duplicate column name** |

**Bug Description**

When creating a correct view defined above, an error that cannot be imported may occur, and the error message "duplicate column name" will be reported. However, after careful inspection, there are no duplicate column names in the statement, and the same DDL statement can run normally on MySQL 5.7, which indicates a schema-related bug hidden in Oceanbase.

#### Bug#22 Different join results when change calculate order.

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**              |
| ----------------------- | ------------------------------------------------------------ | ---------------------- |
| Schema Creation         | create table t0 (a decimal(5,3));create table t1 (a double); | Success                |
| Database Initialization | insert into t0 values(0.01);insert into t1 values(0.1);      | Success                |
| **281**                 | **select \* from t0,t1 where t0.a = t1.a - t1.a + t0.a;**    | **Returns (0.01,0.1)** |
| **282**                 | **select \* from t0,t1 where t0.a = t0.a + t1.a - t1.a;**    | **Returns Empty set**  |

**Bug Description**

The two queries are expected to have the same result. The different results are caused by the float deviation in different calculation order.

## OpenGauss

### Isolation-related bugs

#### Bug#23 JDBC Get A Wrong Snapshot When Start

**Test Case**


| **Transaction ID**      | **Operation Detail**    | **State**       |
| ----------------------- | ----------------------- | --------------- |
| Schema Creation         | create table t (a int); | Success         |
| Database Initialization | insert into t values(1) | Success         |
| 281(jdbc)               | begin;                  | Success         |
| 282(jdbc)               | update t set a = 2;     | Success         |
| **281(jdbc)**           | **select \* from t;**   | **Returns (1)** |

**Bug Description**

The last query is expected to return (2), as that does in CLI, but it returns (1).

It is because OpenGauss is designed to get snapshot at the first non-transaction-control statement of each transaction, but jdbc requires a snapshot at begin.



## DB-X

### Isolation-related bugs

#### Bug#24 Server Crash in deadlock scenario involving partitioned table

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                                                    |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Schema Creation         | CREATE TABLE tbl_1 (col_1_1 TINYINT, col_1_2 INT, col_1_3 INT, col_1_4 TIMESTAMP, pkid_1 INT, version_1 VARCHAR(200), PRIMARY KEY idx_1_1 (col_1_2)) PARTITION BY LIST (col_1_2) (PARTITION p_low VALUES IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11), PARTITION p_high VALUES IN (12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22));<br>CREATE TABLE tbl_2 (col_2_1 INT, col_2_2 MEDIUMINT, col_2_3 INT, col_2_4 VARCHAR(100), pkid_2 INT, version_2 VARCHAR(200), PRIMARY KEY idx_2_1 (col_2_3)); | Success                                                      |
| Database Initialization | INSERT INTO tbl_1 VALUES ('', 1, 20, 1, 8, {ts '2025-12-23 19:08:33.0'});<br>INSERT INTO tbl_2 VALUES (8, 1, 'str_002', '', 19, 1);<br>INSERT INTO tbl_1 VALUES ('', 2, 2, 1, 10, {ts '2025-12-23 19:08:33.0'});<br>INSERT INTO tbl_2 VALUES (10, 1, 'str_002', '', 22, 1); | Success                                                      |
| Session 1               | BEGIN;                                                       | Success                                                      |
| Session 1               | SELECT tbl_2.col_2_3, tbl_2.pkid_2, tbl_2.version_2 FROM tbl_2 WHERE tbl_2.col_2_4 = 'str_003' FOR SHARE; | Success                                                      |
| Session 2               | BEGIN;                                                       | Success                                                      |
| Session 2               | DELETE FROM tbl_2 WHERE tbl_2.col_2_4 = 'str_003';           | Blocked                                                      |
| **Session 1**           | **UPDATE tbl_1 SET tbl_1.col_1_2 = 7, tbl_1.version_1 = CONCAT(tbl_1.version_1, 'T0') WHERE tbl_1.col_1_4 = {ts '2025-12-23 19:08:33.0'};** | **ERROR 2013: Lost connection to MySQL server during query X** |

**Bug Description**

In this test case, Session 1 first acquires a shared lock (FOR SHARE) on `tbl_2` for a non-existent value 'str_003'. Session 2 attempts to delete records with the same condition 'str_003' from `tbl_2` and gets blocked, waiting for the lock held by Session 1 (likely due to gap locking).

Subsequently, Session 1 attempts to update `tbl_1`. Note that `tbl_1` is a partitioned table, and the update operation modifies `col_1_2`, which is the partition key (changing the value from 20 to 7). This update requires moving the row from partition `p_high` to `p_low`. Instead of completing the transaction or detecting a deadlock, the server crashes, and the client loses the connection with `ERROR 2013`. This indicates a critical stability issue when handling partition key updates under specific locking conditions.



#### Bug#25 Unexpected deadlock with SELECT FOR SHARE and ALTER TABLE

See [MySQL Bugs: #119521: Deadlock with SELECT FOR SHARE but not FOR UPDATE](https://bugs.mysql.com/bug.php?id=119521)

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                                                    |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Schema Creation         | drop table if exists table1;<br>create table table1 (pkId int primary key, colA int); | Success                                                      |
| Database Initialization | insert into table1 values (5, 100), (15, 200), (30, 100);    | Success                                                      |
| Session 1               | begin;                                                       | Success                                                      |
| Session 2               | begin;                                                       | Success                                                      |
| Session 1               | select pkId, colA from table1 where pkId between 10 and 20 LOCK IN SHARE MODE; | Success                                                      |
| Session 2               | update table1 set colA = 50 where pkId = 30;                 | Success                                                      |
| Session 3               | ALTER TABLE table1 ADD UNIQUE (colA);                        | Blocked                                                      |
| **Session 1**           | **DELETE FROM table1 WHERE pkId = 30;**                      | **ERROR 1213 (40001): Deadlock found when trying to get lock X** |

**Bug Description**

In this scenario, Session 1 holds a shared lock on a range, Session 2 updates a row outside that range, and Session 3 attempts an `ALTER TABLE` to add a unique index (which waits for metadata locks). When Session 1 subsequently attempts to delete the row updated by Session 2, a deadlock error occurs immediately.

However, if the operation in Session 1 is changed from `LOCK IN SHARE MODE` (Shared Lock) to `FOR UPDATE` (Exclusive Lock), the deadlock does not occur. Instead, the `DELETE` operation in Session 1 is merely blocked, waiting for locks to be released. Once Session 2 commits, Session 1 proceeds successfully, and finally Session 3 completes. The fact that a weaker lock (`SHARE`) triggers a deadlock while a stronger lock (`UPDATE`) does not indicates an anomaly in lock dependency handling or metadata locking logic in MySQL 8.4.



#### Bug#26 Server crash when using PreparedStatement with IN clause on partitioned table

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                            |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------ |
| Schema Creation         | create table table0 (pkId integer, pkAttr0 decimal(10, 0), primary key(pkAttr0)) PARTITION BY key() PARTITIONS 43;<br>alter table table0 add index table0index_pk(pkAttr0); | Success                              |
| Database Initialization | insert into table0 (pkId,pkAttr0) values ("0","3.0");<br>insert into table0 (pkId,pkAttr0) values ("1","8.0");<br>... (Insert total 30 rows) ...<br>insert into table0 (pkId,pkAttr0) values ("29","145.0"); | Success                              |
| JDBC Session            | conn.setAutoCommit(false);                                   | Success                              |
| **JDBC Session**        | **PreparedStatement ps = conn.prepareStatement("SELECT * FROM table0 WHERE pkAttr0 IN (?, ?)");<br>ps.setBigDecimal(1, new BigDecimal("3.0"));<br>ps.setBigDecimal(2, new BigDecimal("8.0"));<br>ps.executeQuery();** | **Connection Lost / Server Crash X** |

**Bug Description**

When using a JDBC connection to execute a transaction, if a `PreparedStatement` is used to query a table defined with partitions (specifically `PARTITION BY key()`), and the query condition involves an `IN` clause with at least two parameters on the partition key (e.g., `pkAttr0 IN (?, ?)`), the database connection is unexpectedly dropped. This behavior suggests a potential server crash triggered by the specific combination of prepared statements, `IN` clause evaluation, and partitioning logic.



#### Bug#27 JDBC connection broken after deadlock in specific versions

**Test Case**

| **Transaction ID**      | **Operation Detail**                               | **State**                                                    |
| ----------------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| Schema Creation         | create table t (k integer primary key, v integer); | Success                                                      |
| Database Initialization | insert into t values(1,1),(2,2);                   | Success                                                      |
| Session 1               | BEGIN;                                             | Success                                                      |
| Session 2               | BEGIN;                                             | Success                                                      |
| Session 1               | update t set v=v+10 where k=1;                     | Success                                                      |
| Session 2               | update t set v=v+10 where k=2;                     | Success                                                      |
| Session 1               | update t set v=v+10 where k=2;                     | Blocked (Waiting for Session 2)                              |
| Session 2               | update t set v=v+10 where k=1;                     | Deadlock found, transaction rolled back (Exception thrown)   |
| **Session 2**           | **rollback() OR close()**                          | **SQLNonTransientConnectionException: Communications link failure X** |

**Bug Description**

This bug relates to how DB-X handles JDBC connections after a deadlock occurs. The test case simulates a standard deadlock scenario using two parallel sessions updating records in reverse order.

In DB-X v2.0 and v3.0, when the deadlock is detected and an exception is thrown for Session 2, the underlying JDBC connection remains valid, allowing the application to execute a `rollback()` or proceed with other operations.

However, in DB-X v2.5, triggering the deadlock causes the JDBC connection to break entirely. Any subsequent operation on that connection object, including standard error handling routines like `rollback()` or `close()`, results in a `SQLNonTransientConnectionException` or `Communications link failure`. This prevents applications from gracefully recovering from deadlock exceptions.



#### Bug#28 Inconsistent update in Read Committed when Primary Key is modified concurrently

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                                                    |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Global Setting          | set global DB-X_no_range_lock_on_rc = true;                  | Success                                                      |
| Schema Creation         | create table t (a int , b int,c int, primary key(a,b));<br>alter table t add index ic(c); | Success                                                      |
| Database Initialization | insert into t values(1,1,null),(1,2,null),(3,3,null);        | Success                                                      |
| Session 1               | BEGIN;                                                       | Success                                                      |
| Session 2               | BEGIN;                                                       | Success                                                      |
| Session 1               | update t set a=2 where b=3;                                  | Success                                                      |
| Session 2               | update t set a=4 where c is null;                            | Blocked                                                      |
| Session 1               | Commit;                                                      | Success                                                      |
| **Session 2**           | **... (Unblocked) -> Commit**                                | **Success (Rows matched: 2, Changed: 2)**                    |
| **Session 2**           | **select * from t;**                                         | **Returns <2,3,null>,<4,1,null>,<4,2,null>** <br> (Expected: <4,3,null>,<4,1,null>,<4,2,null>) |

**Bug Description**

In Read Committed (RC) isolation level, a transaction may fail to update a row if that row's primary key was concurrently modified by another transaction, even if the row still satisfies the update condition.

In the test case above, Session 2 intends to update all rows where `c is null` (which includes all 3 rows). It first scans the data but gets blocked on the row `<3,3,null>` because Session 1 holds a lock on it. Session 1 modifies the primary key of this row from `<3,3>` to `<2,3>` and commits.

When Session 1 commits, Session 2 resumes. However, because the primary key of the blocked row has changed, Session 2 likely fails to locate or re-fetch the row using the old primary key it originally scanned, resulting in that row being skipped.

DB-X updates only 2 rows, leaving the row modified by Session 1 as `<2,3,null>` (with `a=2` from Session 1, not `a=4` from Session 2). In contrast, MySQL (in this specific scenario involving a secondary index) correctly updates all 3 rows (`Rows matched: 3`).

*Note: A similar anomaly exists in MySQL when no secondary index is used (MySQL Bug #118923), but the behavior described here for DB-X occurs even when secondary indexes are present, deviating from standard MySQL behavior.*



#### Bug#29 Potential Server Core Dump with complex INSERT/REPLACE sequence

**Test Case**

| **Transaction ID** | **Operation Detail**                                         | **State**               |
| ------------------ | ------------------------------------------------------------ | ----------------------- |
| Schema Creation    | DROP DATABASE IF EXISTS database43;<br>CREATE DATABASE database43;<br>USE database43;<br>CREATE TABLE IF NOT EXISTS t0(c0 MEDIUMINT(70) ZEROFILL  COLUMN_FORMAT DEFAULT COMMENT 'asdf'  UNIQUE KEY PRIMARY KEY NOT NULL STORAGE DISK); | Success                 |
| Data Manipulation  | UPDATE t0 SET c0=INET_ATON((("e") = (NULL)) && (( EXISTS (SELECT 1 WHERE FALSE)) NOT IN (t0.c0))) WHERE (((t0.c0) IS NOT FALSE) <= ((')i') NOT IN (t0.c0, t0.c0, 1341945352))) < ((+ (('h?F*pCHH') IN (0.8245236965404992)))); | Success                 |
| Data Manipulation  | REPLACE INTO t0(c0) VALUES(1341945352);<br>REPLACE LOW_PRIORITY INTO t0(c0) VALUES(1580004396);<br>INSERT HIGH_PRIORITY INTO t0(c0) VALUES(-1958477645);<br>INSERT INTO t0(c0) VALUES(1341945352);<br>REPLACE INTO t0(c0) VALUES(-781030407), (-1170603897), (2142965007), (-1), (-993046491);<br>REPLACE DELAYED INTO t0(c0) VALUES(233032385), (233032385), (-929048787);<br>REPLACE LOW_PRIORITY INTO t0(c0) VALUES(1184887975);<br>REPLACE DELAYED INTO t0(c0) VALUES(NULL);<br>REPLACE DELAYED INTO t0(c0) VALUES(NULL);<br>INSERT LOW_PRIORITY IGNORE INTO t0(c0) VALUES(-348976803);<br>INSERT INTO t0(c0) VALUES(-993046491);<br>REPLACE LOW_PRIORITY INTO t0(c0) VALUES(1702262481);<br>INSERT HIGH_PRIORITY IGNORE INTO t0(c0) VALUES(-1604196711); | Success                 |
| **Session 1**      | **INSERT IGNORE INTO t0(c0) VALUES(-1183122001), (2017470206), (1063044896);** | **Core Dump / Crash X** |

**Bug Description**

Executing a specific sequence of SQL statements involving `MEDIUMINT ZEROFILL` columns, complex conditional `UPDATE` statements, and various types of write operations (`REPLACE`, `INSERT`, `DELAYED`, `LOW_PRIORITY`, `IGNORE`) may cause the DB-X server to core dump (crash).

The crash typically triggers on the final `INSERT IGNORE` statement in the sequence provided above. It is noted that this bug is difficult to reproduce consistently, suggesting it may involve a race condition or a specific memory corruption state triggered by the combination of `ZEROFILL` attributes, storage engine properties, and mixed DML operations.



#### Bug#30 Inconsistent data and Primary Key violation after INSERT IGNORE

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                                                    |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Schema Creation         | DROP TABLE IF EXISTS t0;<br>CREATE TABLE t0(c1 INT  PRIMARY KEY , c0 DATE) ;<br>CREATE INDEX i0 ON t0(c1, c0 ); | Success                                                      |
| Database Initialization | **INSERT IGNORE INTO t0(c1, c0) VALUES(2, '2025-01-21'), (2, '1742-08-30'), (2, '2026-05-28');** | **Success (2 rows affected, 1 warning) X** <br>(Expected: 1 row affected) |
| Session 1               | set htap_routing_strategy='default';                         | Success                                                      |
| **Session 1**           | **select * from t0;**                                        | **Returns 2 rows: (2025-01-21, 2), (2026-05-28, 2) X**       |
| **Session 1**           | **select count(*) from t0;**                                 | **Returns 1 X**                                              |
| Session 1               | set htap_routing_strategy='vector_engine';                   | Success                                                      |
| **Session 1**           | **select * from t0;**                                        | **Returns 1 row: (2026-05-28, 2) X**                         |

**Bug Description**

This bug highlights a violation of the Primary Key uniqueness constraint and inconsistent read results across different execution engines in DB-X.

The table `t0` has a Primary Key on column `c1`. When executing `INSERT IGNORE` with three records containing the same duplicate key `c1=2`, the database should strictly insert only the first record and discard the others (Standard MySQL behavior results in 1 row: `2025-01-21`).

However, DB-X exhibits the following anomalies:

1.  **Constraint Violation**: The `INSERT IGNORE` statement successfully inserts 2 rows, violating the primary key constraint.
2.  **Inconsistent Reads (Default Engine)**: A simple `SELECT *` scan returns 2 rows (showing duplicate PKs), whereas `SELECT COUNT(*)` returns only 1 row.
3.  **Engine Discrepancy (Vector Engine)**: When switching to the Vector Engine, `SELECT *` returns 1 row, but it returns the *last* inserted value (`2026-05-28`) instead of the first one (`2025-01-21`).

This issue appears to be specific to tables containing `DATE` and `INT` columns with a secondary covering index (`i0`).



#### Bug#31 Transaction implicitly rolled back when JDBC Query Timeout occurs in a Deadlock scenario

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                                                    |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Schema Creation         | create table table0 (pkId integer, pkAttr0 varchar(10), pkAttr1 varchar(10), coAttr0_0 integer, primary key(pkAttr0, pkAttr1) ) ; | Success                                                      |
| Database Initialization | insert into table0 values (0,'vc0','vc2',479),(1,'vc7','vc8',646)...; | Success                                                      |
| Session 1               | BEGIN;                                                       | Success                                                      |
| Session 2               | BEGIN;                                                       | Success                                                      |
| Session 1               | update table0 set coAttr0_0 = 347 where ( pkAttr0 = 'vc7' ); | Success (Locks Row A)                                        |
| Session 2               | insert into table0 (..., pkId=9, pkAttr0='vc45'...) values(...); | Success (Creates Row B)                                      |
| Session 2               | update table0 set coAttr0_0 = 348 where ( pkAttr0 = 'vc7' ); | Blocked (Waiting for Lock A held by S1)                      |
| Session 1               | **(JDBC setQueryTimeout = 2s)**<br>select ... from table0 where ( pkId > 5 ) for update; | Blocked (Waiting for Lock B held by S2)                      |
| **Session 1**           | **... (After 2 seconds)**                                    | **MySQLTimeoutException: Statement cancelled due to timeout** |
| Session 1               | Commit;                                                      | Success                                                      |
| Session 1               | select coAttr0_0 ... where pkAttr0 = 'vc7';                  | **Returns 646 (Old Value)** <br>Expected: 347 (Updated Value) |

**Bug Description**

This bug occurs when using the JDBC `setQueryTimeout` method in a concurrency scenario that effectively forms a deadlock.

1.  **Session 1** updates a row (Row A).
2.  **Session 2** inserts a new row (Row B) and then attempts to update Row A, getting blocked by Session 1.
3.  **Session 1** attempts to read Row B with `FOR UPDATE`. This forms a circular dependency (Deadlock): Session 1 holds A, wants B; Session 2 holds B, wants A.

If Session 1 has a short query timeout (e.g., 2 seconds) set via JDBC:

*   **MySQL Behavior**: It typically detects the deadlock immediately and rolls back the victim transaction (usually Session 2), or if the timeout triggers first, it cancels the statement.
*   **DB-X Behavior**: Session 1 throws a `MySQLTimeoutException` (Statement cancelled). The application catches this and executes `COMMIT`. However, the data updated by Session 1 earlier (Row A = 347) is **lost**.

This implies that although DB-X threw a "Timeout" exception to the client, it internally performed a full transaction rollback (likely due to deadlock detection logic interfering with the timeout logic). Since the client received a Timeout exception rather than a Deadlock exception, it assumed the transaction was still valid and committed, leading to an unexpected loss of written data.



#### Bug#32 Server crash when parsing PreparedStatement with parameter in subquery projection

**Test Case**

| **Transaction ID** | **Operation Detail**                                         | **State**                                                    |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| JDBC Configuration | URL: `jdbc:mysql://...?useServerPrepStmts=true&cachePrepStmts=true` | Success                                                      |
| Schema Creation    | drop table if exists table0;<br>create table table0 (k integer, v integer); | Success                                                      |
| JDBC Session       | PreparedStatement ps = conn.prepareStatement("select * from `table0` where ( `v` = ( select ? ) )"); | Success                                                      |
| **JDBC Session**   | **ps.setInt(1, 10);<br>ps.executeQuery();**                  | **Communications link failure (Server Crash / Core Dump) X** |

**Bug Description**

When using JDBC with server-side prepared statements (`useServerPrepStmts=true`), executing a query where a parameter placeholder (`?`) appears in the projection of a subquery (e.g., `( select ? )`) causes the DB-X server to crash (Core Dump).

The client receives a `Communications link failure` exception. The server-side stack trace indicates a segmentation fault in `Protocol_classic::send_field_metadata` -> `__strlen_sse2_pminub`. This suggests that during the `COM_STMT_PREPARE` phase, when the server attempts to send result set metadata back to the client, it encounters a NULL pointer when trying to determine the column name or metadata for the un-typed `?` in the subquery.

This behavior differs from MySQL 8.4.3, which handles this syntax correctly without crashing.



#### Bug#33 Unique index violation in Read Committed with concurrent updates

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                                                    |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Global Setting          | set global DB-X_no_range_lock_on_rc = true;                  | Success                                                      |
| Schema Creation         | create table t (a double primary key, b double, c double unique, d double);<br>alter table t add index i(d); | Success                                                      |
| Database Initialization | insert into t values(1,1,1,1),(3,3,3,3);                     | Success                                                      |
| Session 1               | BEGIN;                                                       | Success                                                      |
| Session 2               | BEGIN;                                                       | Success                                                      |
| Session 1               | update t set c=2 where a=1;                                  | Success                                                      |
| Session 2               | update t set c=2 where d > 2;                                | **Success (Affected 1 row) X** <br>(Expected: Blocked / Error) |
| Session 1               | Commit;                                                      | Success                                                      |
| Session 2               | Commit;                                                      | Success                                                      |
| **Session 2**           | **select * from t;**                                         | **Returns <1,1,2,1>, <3,3,2,3> X** <br> (Duplicate entry '2' for unique column 'c') |

**Bug Description**

In Read Committed (RC) isolation level (with range locks disabled), DB-X fails to enforce unique constraints under specific concurrent update scenarios involving different access paths (Primary Key vs. Secondary Index).

Transaction 1 updates a row via the primary key, setting the unique column `c` to `2`. Concurrently, Transaction 2 updates a different row (found via secondary index `d`) setting the same unique column `c` to `2`.

In MySQL, Transaction 2 would be blocked waiting for the lock on the unique index entry `2`, and upon Transaction 1's commit, Transaction 2 would fail with `ERROR 1062 (Duplicate entry)`. However, DB-X allows both transactions to succeed and commit. This results in a corrupted state where two different rows hold the value `2` in column `c`, violating the UNIQUE constraint.





#### Bug#34 Inconsistent query results between Unique and Non-Unique indexes after concurrent PK update

**Test Case**

| **Transaction ID**       | **Operation Detail**                                       | **State**                           |
| ------------------------ | ---------------------------------------------------------- | ----------------------------------- |
| Schema Creation (Case A) | create table test (a int,b int, primary key(a),unique(b)); | Success                             |
| Schema Creation (Case B) | create table test (a int,b int, primary key(a),key(b));    | Success                             |
| Database Initialization  | insert into test values(1,1);                              | Success                             |
| Session 1                | BEGIN;                                                     | Success                             |
| Session 1                | select * from test;                                        | Success                             |
| Session 2                | update test set a = 2; (and Commit)                        | Success                             |
| Session 1                | update test set a=3 where a=2;                             | Success                             |
| **Session 1 (Case A)**   | **select * from test where b = 1;**                        | **Returns <1,1> (1 row)**           |
| **Session 1 (Case A)**   | **select * from test where b >= 1;**                       | **Returns <1,1>, <3,1> (2 rows) X** |
| **Session 1 (Case B)**   | **select * from test where b = 1;**                        | **Returns <1,1>, <3,1> (2 rows)**   |
| **Session 1 (Case B)**   | **select * from test where b >= 1;**                       | **Returns <1,1>, <3,1> (2 rows)**   |

**Bug Description**

This bug reveals an inconsistency in query results depending on whether a secondary index is defined as `UNIQUE` or not, specifically after a concurrent transaction modifies the Primary Key.

The scenario involves Transaction 1 starting, then Transaction 2 updates the Primary Key of a row from 1 to 2. Subsequently, Transaction 1 updates that *same* row (using the new PK 2) to a new PK 3.

*   **Case A (Unique Index on `b`)**:
    *   A point query (`b=1`) returns only the old version `<1,1>`.
    *   A range query (`b>=1`) returns **both** the old version `<1,1>` and the newly updated version `<3,1>`. This internal inconsistency (Point scan vs. Range scan) suggests an issue with how unique index lookups handle visibility in the write-set of the current transaction combined with history versions.

*   **Case B (Non-Unique Index on `b`)**:
    *   Both point and range queries return duplicate versions `<1,1>` and `<3,1>`.

While returning both versions (Ghost Rows) might be an anomaly in itself (violating standard snapshot visibility where one should arguably see the latest state of the transaction), the fact that **Unique Index** behaves inconsistently between point and range selects makes this a distinct bug.



#### Bug#35 Non-deterministic read results in repeated transactions with Partition Key updates

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                                                    |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Schema Creation         | CREATE TABLE tbl_1 (col_1_1 INT, col_1_4 INT, pkid_1 INT) PARTITION BY RANGE (col_1_1) SUBPARTITION BY HASH (col_1_1) (PARTITION p0 VALUES LESS THAN (2) (SUBPARTITION s0, SUBPARTITION s1), PARTITION p1 VALUES LESS THAN (4) (SUBPARTITION s2, SUBPARTITION s3)); | Success                                                      |
| Database Initialization | INSERT INTO tbl_1 (col_1_4, pkid_1, col_1_1) VALUES (1, 2, 1);<br>INSERT INTO tbl_1 (col_1_4, pkid_1, col_1_1) VALUES (1, 3, 0); | Success                                                      |
| **Transaction 1**       | BEGIN;                                                       | Success                                                      |
| Transaction 1           | DELETE FROM tbl_1 WHERE (tbl_1.col_1_4 < 1 );                | Success (0 rows affected)                                    |
| Transaction 1           | UPDATE tbl_1 SET tbl_1.col_1_4 = 0, tbl_1.col_1_1 = 0 WHERE tbl_1.col_1_1 > 0; | Success (Updates Row 1: col_1_1 changes 1->0, col_1_4 changes 1->0) |
| **Transaction 1**       | **SELECT tbl_1.col_1_4, tbl_1.pkid_1 FROM tbl_1 WHERE tbl_1.col_1_4 = 1;** | **Returns (1,3), (1,2) X** <br> (Row (1,2) should not match because col_1_4 became 0) |
| Transaction 1           | ROLLBACK;                                                    | Success                                                      |
| **Transaction 2**       | BEGIN;                                                       | Success                                                      |
| Transaction 2           | DELETE FROM tbl_1 WHERE (tbl_1.col_1_4 < 1 );                | Success                                                      |
| Transaction 2           | UPDATE tbl_1 SET tbl_1.col_1_4 = 0, tbl_1.col_1_1 = 0 WHERE tbl_1.col_1_1 > 0; | Success                                                      |
| **Transaction 2**       | **SELECT tbl_1.col_1_4, tbl_1.pkid_1 FROM tbl_1 WHERE tbl_1.col_1_4 = 1;** | **Returns (1,3)** <br> (Correct result)                      |
| Transaction 2           | ROLLBACK;                                                    | Success                                                      |

**Bug Description**

This bug demonstrates inconsistent read behavior when executing the exact same transaction logic twice, separated by a rollback.

The transaction performs an `UPDATE` that modifies the Partition Key (`col_1_1`) and another column (`col_1_4`) of a row.

1.  **First Execution**: The subsequent `SELECT` query incorrectly returns the old version of the updated row (`1,2`), even though `col_1_4` was updated to 0 (which should result in the row being filtered out). This suggests the query in the first transaction failed to see the effects of its own partition-key update.
2.  **Second Execution**: After rolling back and retrying the exact same operations, the `SELECT` query correctly returns only the non-updated row (`1,3`).

Since the first transaction was rolled back, the database state should be identical for the start of the second transaction. The difference in results indicates a state contamination or caching issue related to Partition Key updates within the engine.



#### Bug#36 Ghost rows returned when updating Partition Key in transaction

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                                                    |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Schema Creation         | DROP TABLE IF EXISTS tbl_1;<br>CREATE TABLE tbl_1 (col_1_1 INT, col_1_2 INT, pkid_1 INT, PRIMARY KEY idx_1_1 (col_1_1, col_1_2)) PARTITION BY HASH (col_1_2) PARTITIONS 4; | Success                                                      |
| Database Initialization | INSERT INTO tbl_1 (col_1_1, col_1_2, pkid_1) VALUES (0, 0, 0); | Success                                                      |
| Session 1               | BEGIN;                                                       | Success                                                      |
| Session 1               | DELETE FROM tbl_1 WHERE ((tbl_1.col_1_2 = -1));              | Success (0 rows affected)                                    |
| Session 1               | UPDATE tbl_1 SET tbl_1.col_1_1 = -1, tbl_1.col_1_2 = -1 WHERE tbl_1.col_1_2 = 0; | Success (Updates partition key)                              |
| **Session 1**           | **SELECT * FROM tbl_1;**                                     | **Returns 2 rows: (0,0,0) and (-1,-1,0) X** <br> (Expected: Only (-1,-1,0)) |
| Session 1               | COMMIT;                                                      | Success                                                      |
| Session 1               | SELECT * FROM tbl_1;                                         | Returns 1 row: (-1,-1,0)                                     |

**Bug Description**

In a partitioned table, updating a row's partition key (which involves moving data across partitions or logical delete+insert) causes a subsequent query within the same transaction to return duplicate versions of the same row: the old version (before update) and the new version (after update).

This anomaly is triggered only if the `UPDATE` is preceded by a `DELETE` statement (even if that delete affects 0 rows). This suggests a defect in how the transaction write-set or cursor visibility is handled during partition key updates when combined with previous write intentions in the same transaction. The issue disappears after the transaction is committed.



#### Bug#37 Inconsistent visibility of "no-op" updates in Repeatable Read isolation

**Test Case**

| **Transaction ID**      | **Operation Detail**                   | **State**                                                    |
| ----------------------- | -------------------------------------- | ------------------------------------------------------------ |
| Schema Creation         | create table t (id int, a int);        | Success                                                      |
| Database Initialization | insert into t values(1,2),(2,2),(3,3); | Success                                                      |
| Session 1               | BEGIN;                                 | Success                                                      |
| Session 1               | select * from t;                       | Returns (1,2), (2,2), (3,3)                                  |
| Session 2               | update t set a=a+1;                    | Success (Rows: 3, Changed: 3)<br>Data becomes (1,3), (2,3), (3,4) |
| Session 1               | update t set a=3;                      | Success (Rows matched: 3, **Changed: 1**)<br>Row 1 & 2 are already 3, so optimization skips them. Row 3 changes 4->3. |
| **Session 1**           | **select * from t;**                   | **Returns (1,3), (2,2), (3,3) X**                            |
| Session 1               | Commit;                                | Success                                                      |
| Session 1               | select * from t;                       | Returns (1,3), (2,3), (3,3)                                  |

**Bug Description**

This bug highlights an inconsistency in how "no-op" updates (updates where the new value equals the old value) affect data visibility within a Repeatable Read transaction.

1.  **Context**: Session 1 starts and views a snapshot. Session 2 updates all rows to `a+1` (Row 1: `2->3`, Row 2: `2->3`, Row 3: `3->4`).
2.  **The Trigger**: Session 1 attempts to update `a=3`.
    *   For Row 1 (`a` is 3) and Row 2 (`a` is 3), the value matches the target. The engine optimizes this by not performing a physical update (`Changed: 0` for these rows).
    *   For Row 3 (`a` is 4), the value changes to 3 (`Changed: 1`).
3.  **The Anomaly**: When Session 1 selects the data again:
    *   **MySQL Behavior**: It returns the *snapshot* version for rows that were not physically changed by the transaction (Row 1: `2`, Row 2: `2`) and the *new* version for the row that was changed (Row 3: `3`).
    *   **DB-X Behavior**: It acts inconsistently. It returns the **new** version for Row 1 (`3`), but the **snapshot** version for Row 2 (`2`). Both rows were treated identically by the logic (skipped updates), yet they exhibit different visibility rules. This randomness indicates a bug in how the read view interacts with skipped update optimizations.



#### Bug#38 Skipped Snapshot Creation due to "Impossible WHERE" optimization in Repeatable Read

**Test Case**

| **Transaction ID**      | **Operation Detail**                                      | **State**                                                    |
| ----------------------- | --------------------------------------------------------- | ------------------------------------------------------------ |
| Schema Creation         | create table t1(a int);<br>alter table t1 add index i(a); | Success                                                      |
| Database Initialization | (Table t1 is empty)                                       | Success                                                      |
| Session 1               | BEGIN;                                                    | Success                                                      |
| Session 2               | BEGIN;                                                    | Success                                                      |
| Session 1               | insert into t1 values(1);                                 | Success                                                      |
| **Session 2**           | **select * from t1 where a = null;**                      | **Success (Empty set)** <br> *(Trigger: Query optimization detects "Impossible WHERE" and skips execution/snapshot creation)* |
| Session 1               | Commit;                                                   | Success                                                      |
| **Session 2**           | **select * from t1;**                                     | **Returns (1) X** <br> (Expected: Empty set, due to Repeatable Read) |

**Bug Description**

In Repeatable Read isolation level, the first read operation in a transaction should establish a consistent snapshot. However, if that first read query contains a condition that is statically determined to be impossible (e.g., `a = null` which is always False in SQL, vs `a is null`), the database engine optimizes the execution by directly returning an empty set without actually accessing the storage engine or creating a transaction read view (snapshot).

Consequently, when the transaction executes a second query later, it creates the snapshot *at that moment*. If another transaction (Session 1) has committed data in the interim, this delayed snapshot creation causes the current transaction (Session 2) to see data committed *after* its logical start, violating the expected Repeatable Read behavior.

This issue requires an index on the column used in the `WHERE` clause to trigger the specific optimization path. While present in MySQL 8.0/8.4, it is also observed in DB-X, differing from systems like TiDB which handle this correctly.





## Open reported bugs

## TiDB

### Isolation-related bugs

#### Bug#39 Update with sub query uses incorrect snapshot.

See  [Update with sub query uses incorrect snapshot in RR isolation level · Issue #45677 · pingcap/tidb (github.com)](https://github.com/pingcap/tidb/issues/45677) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                      | **State**                          |
| ----------------------- | --------------------------------------------------------- | ---------------------------------- |
| Schema Creation         | create table t(a int, b int);                             | Success                            |
| Database Initialization | insert into t values(1,1);                                | Success                            |
| 1                       | begin;                                                    | Success                            |
| 2                       | update t set b=2 where a=1;                               | Success                            |
| **1**                   | **update t set b=3 where b=(select b from t where b=2);** | **Success, affects 0 row**         |
| 1                       | commit                                                    | Success                            |
| **1**                   | **select \* from t;**                                     | **Returns (1,2) instead of (1,3)** |

**Bug Description**

Update is designed to read the last committed data. Thus the update in session 1 should read the data version (1,2). However, the sub query in update is executed as a single query and uses snapshot read, which read a earlier version(1,1).The snapshot read by sub query is different from that read by update and breaks the consistency in this single update SQL(with sub query), because a single SQL reads two snapshots.We test the same case in MySQL(v8.0.33), whose sub query read consistent snapshot as update.

Thus this execution is incompatible with MySQL.

## MySQL

### Isolation-related bugs

#### Bug#40 Predicate Lock ERROR

See  [MySQL Bugs: #105988: The problem about predicate lock in Serializable isolation level](https://bugs.mysql.com/bug.php?id=105988) 

**Test Case**

| **Transaction ID** | **Operation Detail**                                         | **State**     |
| ------------------ | ------------------------------------------------------------ | ------------- |
| Schema Creation    | Create table table0 (pkId integer, pkAttr0 integer, coAttr0\_0 integer, primary key(pkAttr0)); | Success       |
| Mode Setting       | Set autocommit = 0;                                          | Success       |
| 69264              | SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;        | Success       |
| 69269              | SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;        | Success       |
| 69264              | START TRANSACTION;                                           | Success       |
| 69264              | Insert into `table0`(`pkId`, `pkAttr0`, `coAttr0_0`) values(225, 225, 35704); | Success       |
| 69269              | START TRANSACTION;                                           | Success       |
| **69269**          | **Select `pkAttr0`, `coAttr0_0` from `table0` where ( `pkAttr0` = 225 );** | **Success X** |
| 69264              | rollback                                                     | Success       |

**Bug Description**

We disable autocomit and set both the above two transactions as serializable isolation level, as shown in the second to third lines.

As definition, InnoDB implicitly converts all plain SELECT statements to SELECT ... FOR SHARE if autocommit is disabled. Therefore, the seventh line should acquire a range-level shared lock to protect all records whose pkAttr0 is 225.

However, the fifth line has acquired a record-level exclusive lock on a record whose pkAttr0 is 225. The exclusive lock acquired by the fifth line is imcompatible with the shared lock acquired by the seventh line, which indicates a bug hidden in range-level lock.

#### Bug#41 Read uncommitted transaction reads the result of a failed write operation

**Test Case**

| **Transaction ID**      | **Operation Detail**                                         | **State**                          |
| ----------------------- | ------------------------------------------------------------ | ---------------------------------- |
| Schema Creation         | set global innodb\_deadlock\_detect=off;create table t(a int primary key, b int); | Success                            |
| Database Initialization | insert into t values(1,2);insert into t values(2,4);         | Success                            |
| 1                       | begin;                                                       | Success                            |
| 2                       | begin;                                                       | Success                            |
| 2                       | set session transaction isolation level read uncommitted;    | Success                            |
| 3                       | begin;                                                       | Success                            |
| 3                       | set session transaction isolation level read uncommitted;    | Success                            |
| 2                       | delete from t where a=1;                                     | Success                            |
| 3                       | update t set b=321 where a=2;                                | Success                            |
| 2                       | update t set b=1421 where a=2;                               | Success                            |
| 3                       | insert into t value(1,1231);                                 | Rollback                           |
| 1                       | select \* from t where a=1;                                  | Returns (1,2) instead of empty set |

**Bug Description**

Transaction 2 writes new versions on records 1 and 2 successively, while Transaction 3 writes new versions on records 2 and 1 successively. So there is a deadlock situation between transaction 2 and 3. Before the deadlock between transaction 2 and 3 timeouts, another read uncommitted transaction 1 launch a query to read the record 1 that has been modified by transaction 2 and 3 successively. Since the second write operation of transaction 3 are failed due to deadlock, we should not see its write results. Therefore, as expected, the query result of transaction 1 should be the write result of transaction 2. However, the query result of transaction 1 is the write result before transaction 2, which is weird. We think there may be a subtle bug hidden in the current version of MySQL.

### Other types of Bugs

#### Bug#42 Update BLOB data error

**Test Case**

| **Operation ID**        | **Operation Detail**                                         | **State**                          |
| ----------------------- | ------------------------------------------------------------ | ---------------------------------- |
| Schema Creation         | create table t(pk0 int, pk1 int, pk2 int, attr3 blob);       | Success                            |
| Database Initialization | insert into t values(1,1,1,null);                            |                                    |
| 1                       | Update t set attr3=FILE("./data_case/obj/12obj_file.obj") where pk0=1 and pk1=1 and pk2= 1 | Success                            |
| 2                       | Update t set attr3=FILE("./data_case/obj/12obj_file.obj") and other column where pk0 = 1 and pk1 = 1 and pk2 = 1 | Success                            |
| **3**                   | **Select attr3 from t where pk0 = 1 and pk1 = 1 and pk2 = 1 for update** | **Success  Return attr3 = NULL X** |

**Bug Description**

For BLOB data type, when the new value and the old value written by the update operation are for the same binary file, the value actually written is null and success is returned, which indicates a BLOB-related bug hidden in MySQL.

## PostgreSQL

### Isolation-related bugs

#### Bug#43 Write skew in SSI

**Test Case**

| **Transaction ID** | **Operation Detail**                                         | **State**   |
| ------------------ | ------------------------------------------------------------ | ----------- |
| 206                | Select attribute1 from table\_7\_1 where primarykey= 832     | Success     |
| 204                | Select attribute1 from table\_7\_4 where primarykey= 1460    | Success     |
| 206                | Update table\_7\_4 set attribute where primarykey=1460       | Success     |
| 204                | Update table\_7\_1 set attribute1 = -635092 where primarykey= 832 | Success     |
| 204                | Commit                                                       | Success     |
| **206**            | **Commit**                                                   | **Success** |

**Bug Description**

Transaction 206 reads a record 832 in table\_ 7\_ 1，then transaction 204 writes a new record to cover it, so transactions 206 to 204 have a RW dependency. Similarly, transaction 204 reads the record 1460 in table\_ 7\_ 4, then transaction 206 writes a new record to cover it, so transactions 204 to 206 have a RW dependency. Finally, transactions 204 to 206 generate a circular dependency, that is, write skew anomalies that should be avoided in Snapshot Isolation Level of PostgreSQL.

#### Bug#44 Two different versions of the same row of records are returned in one query

See  [PostgreSQL: BUG #17017: Two versions of the same row of records are returned in one query](https://www.postgresql.org/message-id/17017-c37dbbadb77cfde9%40postgresql.org) 

**Test Case**

| **Transaction ID**      | **Operation Detail**                                 | **State**                               |
| ----------------------- | ---------------------------------------------------- | --------------------------------------- |
| Schema Creation         | Create Table t(a int primary key, b int);            | Success                                 |
| Database Initialization | Insert into t values(1,2);Insert into t values(2,3); | Success                                 |
| 1                       | begin;                                               | Success                                 |
| 1                       | set transaction isolation level repeatable read;     | Success                                 |
| 1                       | Select \* from t where a=1;                          | Success                                 |
| 2                       | Begin;                                               | Success                                 |
| 2                       | set transaction isolation level read committed;      | Success                                 |
| 2                       | Delete from t where a=2;                             | Success                                 |
| 2                       | Commit;                                              | Success                                 |
| 1                       | Insert into t values(2,4);                           | Success                                 |
| **1**                   | **Select \* from t where a=2;**                      | **Returns (2,4)(2,3) instead of (2,4)** |

**Bug Description**

According to the definition of snapshot isolation, a query in a transaction should always see a consistent view of the database. That is, it should see a consistent version for each record in the database. However, bug#20 shows that two different versions of the same record are returned to a query under the snapshot isolation of PostgreSQL. This certainly violates the definition of snapshot isolation. Thus, we report it to the PostgreSQL community.

## OpenGauss

### Isolation-related Bugs

#### Bug#45 Violating First-Updater-Wins

**Test Case**

| Transaction ID | Session1                                                     | Session2                                                     | State                     |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------- |
| 2              |                                                              | Begin;                                                       | Success                   |
| 2              |                                                              | set session transaction isolation level repeatable read;     | Success                   |
| 2              |                                                              | update "table0" set "coAttr31_0" = 1048.0 where ( "pkAttr0" = 280 ) and ( "pkAttr1"  = 241 ) and ( "pkAttr2" = ‘vc204’ ) and ( "pkAttr3" =  ‘vc361’ ) and ( "pkAttr4" = 363 );--row count=1 | Success                   |
| 1              | Begin;                                                       |                                                              | Success                   |
| 1              | set session transaction isolation level repeatable read;     |                                                              | Success                   |
| 1              | select "pkAttr0", "pkAttr1", "pkAttr2", "pkAttr3", "pkAttr4", "pkAttr5", "pkAttr6", "pkAttr7", "fkAttr0\_0", "fkAttr0\_1", "fkAttr0\_2", "fkAttr0\_3", "fkAttr0\_4" from "view0" where ( "fkAttr0\_0" = 94 ) and ( "fkAttr0\_1" = 239 ) or ( "fkAttr0\_2" \< 'vc119' ) and ( "fkAttr0\_3" \> 'vc81u' ) and ( "fkAttr0\_4" = 278 ) ; |                                                              | Success                   |
| 2              |                                                              | commit                                                       | Success                   |
| **1**          | **delete from "table0" where ( "pkAttr0" = 280 ) and ( "pkAttr1" = 241 ) and ( "pkAttr2" = 'vc204' ) and ( "pkAttr3" = 'vc361' ) and ( "pkAttr4" = 363 );** |                                                              | **Success --row count=1** |

**Bug Description**

Transaction 1 starts before transaction 2 commit, and both transaction 1 and 2 write a new version on a record (280, 241,'vc204' , 'vc361' ,363 ). Therefore, transaction 1 and 2 are a pair of concurrent transaction, which should be avoided by first updater wins mechanism in OpenGauss.

#### Bug#46 Violating Read-Consistency

**Test Case**

Create table table2 (primarykey int primary key, coAttr25\_0 int);

Insert into table2 values(6,0);

Insert into table2 values(7,0);

| **Transaction ID** | **Session1**                                                 | **Session2**                                                 | **State**                                                |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------------------------------- |
| 1                  | Begin;                                                       |                                                              | Success                                                  |
| 1                  | set session transaction isolation level repeatable read;     |                                                              | Success                                                  |
| 1                  | update "table2" set "coAttr25\_0" = 78354, where "primaryKey" = 7; |                                                              | Success                                                  |
| 2                  |                                                              | Begin;                                                       | Success                                                  |
| 2                  |                                                              | set session transaction isolation level repeatable read;     | Success                                                  |
| 2                  |                                                              | "update "table2" set " coAttr25\_0" = 14 where "primaryKey" = 6; | Success                                                  |
| 2                  |                                                              | Commit                                                       | Success                                                  |
| **1**              | **Select "primaryKey", "fkAttr0\_0", "coAttr25\_0" from "table2";** |                                                              | **Success returns"primaryKey":"6", "coAttr25\_0": "14"** |

**Bug Description**

Transaction 1 launch a update operation while fetches a consistent snapshot. According to the rule of repeatable read isolation level, any operation in transaction 1 should sees a same snapshot., so transaction 1 should not see the write result created by transaction 2. However, transaction 1 sees the write result created by transaction 2, which indicates a consistency read violation.

## Oceanbase

### Isolation-related bugs

#### Bug#47 Read inconsistency

**Test Case**

| Transaction ID | Session1                                                     | Session2                                                     | State                                                        |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1              | set session transaction isolation level repeatable read;     |                                                              | Success                                                      |
| 2              |                                                              | set session transaction isolation level repeatable read;     | Success                                                      |
| 1              | START TRANSACTION READ ONLY,WITH CONSISTENT SNAPSHOT;        |                                                              | Success                                                      |
| 2              |                                                              | START TRANSACTION;                                           | Success                                                      |
| 2              |                                                              | update table0 set coAttr17 = 19635, coAttr18 = 1244, coAttr19 = 92947 where ( pkAttr0 = 'vc239' ) and ( pkAttr1 = 'vc234' ) and ( pkAttr2 = 'vc233' ); | Success, affects 1 row                                       |
| 2              |                                                              | COMMIT                                                       | Success                                                      |
| **1**          | **select pkAttr0, pkAttr1, pkAttr2, coAttr17, coAttr18, coAttr19 from table0 order by pkAttr0 ;** |                                                              | **Success returns (vc239, vc234, vc233, 19635, 1244, 92947)** |

**Bug Description**

After confirmation with developer, the repeatable read isolation level of Oceanbase is consistent with snapshot isolation as defined in the paper "A Critique of ANSI SQL Isolation Levels". Specifically, snapshot isolation is define as:

1. A read operation sees a snapshot as of the start of the transaction.
2. No transaction modify the record that has been modified by another concurrent transaction.

However, Oceanbase violates the first rule of snapshot isolation. Specifically, after transaction 1 obtains the consistency snapshot, another parallel transaction 2 issues a write operation. Transaction 1 should not see the write result created by transaction 2. In practice, transaction 1 sees it.



## DB-X

### Isolation-related bugs

#### Bug#48 Deadlock between DML transaction and Partition DDL operations

**Test Case**

| **Transaction ID** | **Operation Detail**                                         | **State**                                                    |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Schema Creation    | drop table if exists table0;<br>create table table0 (pkId integer, pkAttr0 integer, commonAttr0_0 integer, commonAttr1_0 varchar(10), commonAttr2_0 double(10, 2), commonAttr3_0 varchar(10), commonAttr4_0 varchar(10), primary key(pkAttr0)) ROW_FORMAT = DYNAMIC  PARTITION BY KEY(pkAttr0)  PARTITIONS 4 ; | Success                                                      |
| Session 1          | BEGIN;                                                       | Success                                                      |
| Session 1          | select `pkId`, `commonAttr4_0`, `pkAttr0` from `table0` where ( `pkAttr0`  =  77  )  ; | Success                                                      |
| Session 2          | **alter table table0 coalesce partition 1;**                 | **Blocked (Waiting for metadata lock held by S1)**           |
| **Session 1**      | **update `table0` set `commonAttr1_0` = "CLf"... where (`pkAttr0`  =  132  );** | **ERROR 1213 (40001): Deadlock found when trying to get lock X** |

**Bug Description**

This bug manifests as a deadlock when a DML transaction runs concurrently with specific partition-related DDL operations on the same table.

1.  **Session 1** starts a transaction and performs a read operation (`SELECT`), acquiring a shared metadata lock (MDL) on the table.
2.  **Session 2** attempts a partition DDL (e.g., `COALESCE PARTITION`, `REORGANIZE PARTITION`, or `ADD PARTITION`). This requires an exclusive lock and is blocked by Session 1.
3.  **Session 1** proceeds to perform an `UPDATE` on the same table.

Instead of simply being blocked or successfully executing (depending on the exact lock compatibility), Session 1 immediately fails with a Deadlock error. This suggests a conflict in the wait graph between the DML locks and the pending Partition DDL locks. This behavior is also reproducible in MySQL (Bug #117735), but in DB-X, the deadlock cycle is difficult to trace in the transaction logs.

