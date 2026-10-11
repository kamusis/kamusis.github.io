---
title: Flashback Features in MogDB
tags: MogDB
date: 2024-06-15 18:32:07
---


## What is Database Flashback

The flashback capability of a database allows users to query the contents of a table at a specific point in the past or quickly restore a table or even the entire database to a time before an erroneous operation. This feature can be extremely useful, and sometimes even a lifesaver.

Database flashback capabilities are generally divided into three levels:

- **Row-level Flashback:** Also known as flashback query, this typically involves using a SELECT statement to retrieve data from a table as it was at a specific point in time, such as right before a DELETE command was issued.
- **Table-level Flashback:** Also known as flashback table, this usually involves using specialized DDL statements to recreate a table from the recycle bin. This is often used to recover from a TRUNCATE TABLE or DROP TABLE operation.
- **Database-level Flashback:** Also known as flashback database, this involves using specialized DDL statements to restore the entire database to a previous point in time. Unlike PITR (Point-In-Time Recovery) which involves restoring from backups, flashback database is faster as it does not require restoring the entire database from backup sets.

In MogDB version 5.0, only flashback query and flashback table have been implemented so far. Flashback database has not yet been implemented (Oracle has supported flashback database since version 10g).



## Scenario 

Imagine a regrettable scenario (**Note:** do not run the DELETE SQL command in a production environment).

You have a table that records account names and account balances.

```sql
MogDB=# select * from accounts;
 name | amount
------+--------
 A    |    100
 B    |    100
 C    |    100
 D    |     99
(4 rows)
```

You intended to execute an SQL command to delete account records with a balance of 99 units. Normally, this should delete one record. To demonstrate, let's use SELECT instead of DELETE.

```sql
MogDB=# select * from accounts where amount=99;
 name | amount
------+--------
 D    |     99
(1 row)
```

However, the minus sign "-" and the equals sign "=" are adjacent on the keyboard, and you have fat fingers, so you accidentally pressed the minus sign. As a result, the command you sent to the database looks like this:

```sql
delete from accounts where amount-99;
```

For demonstration purposes, we'll still use SELECT instead of DELETE.

```sql
MogDB=# select * from accounts where amount-99;
 name | amount
------+--------
 A    |    100
 B    |    100
 C    |    100
(3 rows)
```

A terrifying thing happens: all data except the record with a balance exactly equal to 99 is returned. This means that if this were the DELETE command mentioned above, you would have deleted all accounts with a balance not equal to 99.

The good news is that starting from MogDB version 3.0, validation for such dangerous syntax involving the minus sign has been added. Now, executing the same SQL will result in an error.

```sql
MogDB=# delete from accounts where amount-99;
ERROR:  argument of WHERE must be type boolean, not type integer
LINE 1: delete from accounts where amount-99;
                                   ^
```

However, in the community editions of openGauss, MySQL, or MariaDB, such dangerous syntax can still be executed normally.

```sql
gsql ((openGauss 6.0.0-RC1 build ed7f8e37) compiled at 2024-03-31 12:41:30 commit 0 last mr  )
Non-SSL connection (SSL connection is recommended when requiring high-security)
Type "help" for help.

openGauss=# select version();
                                                                          version
------------------------------------------------------------------------------------------------------------------------------------------------------------
 (openGauss 6.0.0-RC1 build ed7f8e37) compiled at 2024-03-31 12:41:30 commit 0 last mr   on aarch64-unknown-linux-gnu, compiled by g++ (GCC) 10.3.1, 64-bit
(1 row)

openGauss=# create table accounts (name varchar2, amount int);
CREATE TABLE
openGauss=# insert into accounts values ('A',100),('B',100),('C',100),('D',99);
INSERT 0 4

openGauss=# delete from accounts where amount-99;
DELETE 3
openGauss=# select * from accounts;
 name | amount
------+--------
 D    |     99
(1 row)
```

No matter what kind of erroneous operation occurs, suppose a data deletion error really happens. In MogDB, you still have the flashback feature available for recovery.



## Flashback Feature in MogDB

The flashback feature and its related implementations have undergone some changes since MogDB version 3.0.

1. **Applicable Only to Ustore Storage Engine:**

   The flashback feature only works for tables using the ustore storage engine. The default astore no longer supports flashback queries. Therefore, you need to set `enable_ustore=on`. This parameter is off by default, and changing it requires a database restart to take effect.

   ```sql
   MogDB=# alter system set enable_ustore=on;
   NOTICE:  please restart the database for the POSTMASTER level parameter to take effect.
   ALTER SYSTEM SET
   ```

2. **Setting `undo_retention_time`:**

   This parameter specifies the retention time for old version data in the rollback segment, equivalent to the allowable time span for flashback queries. The default value is 0, meaning any flashback query will encounter a "restore point not found" error. Changing this parameter does not require a database restart.

   ```sql
   MogDB=# alter system set undo_retention_time=86400; -- 86400 seconds = 24 hours
   ALTER SYSTEM SET
   ```

3. **Enabling Database Recycle Bin for Truncate or Drop Operations:**

   To flashback a table from a truncate or drop operation, you need to enable the database recycle bin by setting `enable_recyclebin=on`. This parameter is off by default, and changing it does not require a database restart.

   ```sql
   MogDB=# alter system set enable_recyclebin=on;
   ALTER SYSTEM SET
   ```

### Creating and Populating the Ustore Table

Create a ustore-based `accounts` table and insert some test data.

```sql
MogDB=# create table accounts (name varchar2, amount int) with (storage_type=ustore);
CREATE TABLE

MogDB=# insert into accounts values ('A',100),('B',100),('C',100),('D',99);
INSERT 0 4
```

### Simulating an Erroneous Deletion

Now, due to some erroneous operation, you delete all account records with amounts not equal to 99.

```sql
MogDB=# delete from accounts where amount<>99;
DELETE 3

MogDB=# select * from accounts;
 name | amount
------+--------
 D    |     99
(1 row)
```

### Flashback Query

When you realize the mistake, it might be 1 minute or 1 hour later. As long as it is within 24 hours (due to the `undo_retention_time` setting), you can recover the data.

Check the current timestamp and estimate the timestamp at the time of the erroneous operation. For simplicity, let's assume you noted the system's timestamp before issuing the delete command.

```sql
MogDB=# select sysdate;
       sysdate
---------------------
 2024-06-13 18:40:18
(1 row)
```

Issue a flashback query to retrieve the data as it was at the specified timestamp.

```sql
MogDB=# select * from accounts timecapsule TIMESTAMP to_timestamp('2024-06-13 18:40:18','YYYY-MM-DD HH24:MI:SS');
 name | amount
------+--------
 A    |    100
 B    |    100
 C    |    100
 D    |     99
(4 rows)
```

### Recovering Data

You can recover the data by creating a temporary table with the flashback query results and then inserting the data back into the original table.

```sql
MogDB=# create table tmp_accounts as select * from accounts timecapsule TIMESTAMP to_timestamp('2024-06-13 18:40:18','YYYY-MM-DD HH24:MI:SS') where amount<>99;
INSERT 0 3

MogDB=# select * from tmp_accounts;
 name | amount
------+--------
 A    |    100
 B    |    100
 C    |    100
(3 rows)

MogDB=# insert into accounts select * from tmp_accounts;
INSERT 0 3

MogDB=# select * from accounts;
 name | amount
------+--------
 D    |     99
 A    |    100
 B    |    100
 C    |    100
(4 rows)
```

Alternatively, if no new data has been added to the table after the erroneous operation, you can use the `timecapsule table` command to flashback the entire table to the specified timestamp.

```sql
MogDB=# timecapsule table accounts to TIMESTAMP to_timestamp('2024-06-13 18:40:18','YYYY-MM-DD HH24:MI:SS');
TimeCapsule Table

MogDB=# select * from accounts;
 name | amount
------+--------
 D    |     99
 A    |    100
 B    |    100
 C    |    100
(4 rows)
```

### Recovering from Truncate or Drop Table

If you accidentally issue a `TRUNCATE` or `DROP` command, In this situation, the before commands don't help because the data has been truncated.

```sql
MogDB=# truncate table accounts;
TRUNCATE TABLE

MogDB=# select * from accounts timecapsule TIMESTAMP to_timestamp('2024-06-14 02:10:28','YYYY-MM-DD HH24:MI:SS');
ERROR:  Snapshot too old, ScanRelation, the info: snapxmax is 45865, snapxmin is 45865, csn is 30047, relfrozenxid64 is 45877, globalRecycleXid is 17356.

MogDB=# timecapsule table accounts to TIMESTAMP to_timestamp('2024-06-14 02:10:28','YYYY-MM-DD HH24:MI:SS');
TimeCapsule Table

MogDB=# select * from accounts;
 name | amount
------+--------
(0 rows)
```

For `TRUNCATE` or `DROP TABLE` operations, use the `to before` keyword to recover the table from the recycle bin.

```sql
MogDB=# timecapsule table accounts to before truncate;
TimeCapsule Table

MogDB=# select * from accounts;
 name | amount
------+--------
 A    |    100
 B    |    100
 C    |    100
 D    |     99
(4 rows)
```

Similarly, if the table was dropped, you can recover it from the recycle bin using the same `to before` keyword.

```sql
MogDB=# drop table accounts;
DROP TABLE
MogDB=# select * from accounts;
ERROR:  relation "accounts" does not exist on dn_6001
LINE 1: select * from accounts;
                      ^
MogDB=# timecapsule table accounts to before drop;
TimeCapsule Table
MogDB=# select * from accounts;
 name | amount
------+--------
 A    |    100
 B    |    100
 C    |    100
 D    |     99
(4 rows)
```



## Conclusion

MogDB's flashback feature is an essential tool for recovering from accidental data deletions, truncations, or drops. By enabling the ustore storage engine, setting an appropriate `undo_retention_time`, and activating the recycle bin, you can leverage flashback queries and the `timecapsule` command to restore your data efficiently. These features ensure that you can maintain data integrity and quickly recover from human errors, providing robust data protection and operational resilience.
