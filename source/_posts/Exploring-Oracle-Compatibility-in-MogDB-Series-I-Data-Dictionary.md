---
title: Exploring Oracle Compatibility in MogDB (Series I) - Data Dictionary
date: 2024-06-06 12:45:38
tags: [MogDB, Oracle-Compatibility]
---

## Install PTK

PTK is the optimal tool for installing the MogDB database (an enterprise-grade commercial distribution based on openGauss) or the openGauss database, offering a seamless and smooth installation experience.

To install PTK, simply run the following command:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://cdn-mogdb.enmotech.com/ptk/install.sh | sh
```

This command will automatically install PTK in the user's home directory under `$HOME/.ptk`. This directory will serve as the working directory for PTK, storing cache files, metadata information, cluster configuration information, backup information, and other related files. Additionally, the installation command will add the path `$HOME/.ptk/bin` to the PATH environment variable in the corresponding shell profile file, enabling the user to use the `ptk` command directly after logging into the server. In this tutorial, we installed PTK using the root user, although this is not mandatory.

For more detailed PTK installation instructions, please refer to: [PTK Installation Guide](https://docs.mogdb.io/en/ptk/v1.1/install).

The environment used for this series of articles is CentOS 7.6 for x86-64. MogDB must currently run on a Linux operating system. If you wish to install MogDB on macOS or Windows, you can do so using container deployment. For more information, refer to: [Container-based MogDB Installation](https://docs.mogdb.io/en/mogdb/v5.0/container-based-installation#container-based-mogdb-installation)

To check your CentOS version, you can use the following command:

```bash
# cat /etc/centos-release
CentOS Linux release 7.6.1810 (Core)
```

MogDB can also run on ARM architecture CPUs. You can list all supported CPU brands using the `ptk candidate cpu` command:

```bash
# ptk candidate cpu
    CPU Model
------------------
  Cortex-ARM64
  Kunpeng-ARM64
  Phythium-ARM64
  Hygon-x64
  Intel-x64
  AMD-x64
  zhaoxin-x64
```

If PTK notifies you of a new version while running any command, you can upgrade it directly using the `ptk self upgrade` command.

```bash
# ptk ls
  cluster_name | id | addr | user | data_dir | db_version | create_time | comment
---------------+----+------+------+----------+------------+-------------+----------
Warning: New version '1.5.0' is available, you are using ptk version '1.1.3'.
You can upgrade ptk via command: 'ptk self upgrade'
You can also set environment by 'export PTK_NO_CHECK_VERSION=true' to disable this warning
# ptk self upgrade
INFO[2024-06-06T11:59:01.105] current version: 1.1.3 release, target version: latest
INFO[2024-06-06T11:59:01.105] download package from http://cdn-mogdb.enmotech.com/ptk/latest/ptk_linux_x86_64.tar.gz
INFO[2024-06-06T11:59:01.105] downloading ptk_linux_x86_64.tar.gz ...
> ptk_linux_x86_64.tar.gz: 17.78 MiB / 20.08 MiB [----------------------------------------------------------------------->_________] 88.52% 27.93 MiB p/s ETA 0s
> ptk_linux_x86_64.tar.gz: 20.08 MiB / 20.08 MiB [---------------------------------------------------------------------------------] 100.00% 29.07 MiB p/s 900ms
INFO[2024-06-06T11:59:02.956] upgrade ptk successfully
[root@mogdb-kernel-0004 ~]# ptk version
PTK Version:    v1.5.0 release
Go Version:     go1.19.10
Build Date:     2024-06-04T17:16:07
Git Hash:       edd5dbb0
OS/Arch:        linux/amd64
```



## Install MogDB

With PTK, you can easily create multiple MogDB instances on a single server. Each MogDB instance can be assigned to a different operating system user, allowing you to manage multiple database instances with a single PTK installation.

To quickly create a test database without any configuration, you can use the `ptk demo` command:

```bash
# ptk demo
Cluster Name: "demo_kVP"
+--------------+--------------+------------------------------+-------+---------+---------------------------------+----------+
| az(priority) |      ip      |         user(group)          | port  |  role   |            data dir             | upstream |
+--------------+--------------+------------------------------+-------+---------+---------------------------------+----------+
| AZ1(1)       | 172.16.0.164 | demo_user_kVP(demo_user_kVP) | 26000 | primary | /home/demo_user_kVP/demodb/data | -        |
+--------------+--------------+------------------------------+-------+---------+---------------------------------+----------+
✔ Is cluster config correct (default=n) [y/n]: y
INFO[2024-06-06T12:12:05.948] start check hostname ...
INFO[2024-06-06T12:12:05.951] check hostname success
INFO[2024-06-06T12:12:05.951] skip check os
INFO[2024-06-06T12:12:05.951] start check distro ...
INFO[2024-06-06T12:12:05.972] check distro success
[....]
Demo Summary:

Deploy Status:

  cluste_name |     host     |     user      | port  |    status     | message
--------------+--------------+---------------+-------+---------------+----------
  demo_kVP    | 172.16.0.164 | demo_user_kVP | 26000 | start_success | success

Database Detail:

      item      |              value
----------------+----------------------------------
  user_password | Demo&kVP
  db_password   | Demo&kVP
  base_dir      | /home/demo_user_kVP/demodb
  app_dir       | /home/demo_user_kVP/demodb/app
  data_dir      | /home/demo_user_kVP/demodb/data
  tool_dir      | /home/demo_user_kVP/demodb/tool
  tmp_dir       | /home/demo_user_kVP/demodb/tmp

Mode Compatibility:

  mode |        database        |        plugins        |               other
-------+------------------------+-----------------------+-------------------------------------
  PG   | postgres_compatibility | none                  |
  A    | oracle_compatibility   | whale[success]        |
       |                        | compat_tools[success] |
       |                        | mogila[success]       | db user: mogdb, password: Demo&kVP

Plugin Install Details:
  mode |    plugin    | status  |                 reference                 | error
-------+--------------+---------+-------------------------------------------+--------
  A    | whale        | success | https://docs.mogdb.io/zh/mogdb/v3.0/whale |
       | compat_tools | success | https://gitee.com/enmotech/compat-tools   |
       | mogila       | success | https://gitee.com/enmotech/mogila         |
```

The `ptk demo` command accomplishes the following tasks:

1. Creates an operating system user named `demo_user_kVP`.
2. Creates a single-instance MogDB database of the latest version (currently 5.0.7).
3. Sets the database to listen on port 26000. Before running the `ptk demo` command, ensure that port 26000 is not occupied by other applications.
4. Create an initial database user with the same name as the OS user: `demo_user_kVP`. Also, create a monitor user named `mogdb`. Both users will have their initial passwords specified in the "Database Detail" section.
5. Creates two test databases: `oracle_compatibility` in Oracle-compatible mode and `postgres_compatibility` in PostgreSQL-compatible mode.
6. For the automatically created `oracle_compatibility` database, the following enhancements are also performed:
   - Installs the mogila dataset. For more information about this dataset, refer to: [Using Sample Dataset Mogila](https://docs.mogdb.io/en/mogdb/v5.0/mogila)
   - Installs the `whale` plugin, which provides extensive Oracle compatibility features. For more information on the Whale plugin, refer to: [whale](https://docs.mogdb.io/en/mogdb/v5.0/whale)
   - Installs the `compat-tools` utility, which supplements the `whale` plugin with additional Oracle compatibility features that are not yet implemented, such as Oracle-compatible data dictionary views. For more information on `compat-tools`, refer to: [enmotech/compat-tools](https://gitee.com/enmotech/compat-tools)

     **Note:** `compat-tools` only work within a single database. This means that if you install compat-tools in the `oracle_compatibility` database (as is done automatically by the `ptk demo` command), you will only be able to query the corresponding Oracle-compatible data dictionary views when logged into that database. If you want to use these views in another database, you must install compat-tools in that database as well. For example, if you want to query Oracle-compatible data dictionary views in the `postgres` database, you need to download `compat-tools` separately from [enmotech/compat-tools](https://gitee.com/enmotech/compat-tools) then run the following command:

     ```bash
     gsql -h 127.0.0.1 -p 26000 -d postgres -f runMe.sql
     ```

     

## Exploring Oracle-Compatible Data Dictionary in MogDB

Switch to the operating system user associated with the demo database. You can log in to the database using the `gsql` command.

**Note:** If you want to connect to the database remotely, you should use the `mogdb` user. The `ptk demo` command creates two users in the database. One starts with `demo_user_`, which is the initial user. Due to MogDB's enhanced security features, the initial user is not allowed to connect remotely. The other user is `mogdb`, which can be used for remote connections. The initial password for the users is displayed at the end of the `ptk demo` command output. Additionally, you should modify the `pg_hba.conf` file to allow remote connections. For more information, refer to: [Connecting to a Database Remotely](https://docs.mogdb.io/en/mogdb/v5.0/gsql#connecting-to-a-database-remotely).

```bash
# ptk ls
  cluster_name |  id  |        addr        |     user      |            data_dir             |          db_version          |     create_time     | comment
---------------+------+--------------------+---------------+---------------------------------+------------------------------+---------------------+----------
  demo_kVP     | 6001 | 172.16.0.164:26000 | demo_user_kVP | /home/demo_user_kVP/demodb/data | MogDB 5.0.7 (build c4707384) | 2024-06-06T12:12:24 |
# su - demo_user_kVP
$ gsql -d oracle_compatibility -r
gsql ((MogDB 5.0.7 build c4707384) compiled at 2024-05-24 10:51:53 commit 0 last mr 1804 )
Non-SSL connection (SSL connection is recommended when requiring high-security)
Type "help" for help.

oracle_compatibility=# \dt
                             List of relations
 Schema |     Name      | Type  | Owner |             Storage
--------+---------------+-------+-------+----------------------------------
 public | actor         | table | mogdb | {orientation=row,compression=no}
 public | address       | table | mogdb | {orientation=row,compression=no}
 public | category      | table | mogdb | {orientation=row,compression=no}
 public | city          | table | mogdb | {orientation=row,compression=no}
 public | country       | table | mogdb | {orientation=row,compression=no}
 public | customer      | table | mogdb | {orientation=row,compression=no}
 public | film          | table | mogdb | {orientation=row,compression=no}
 public | film_actor    | table | mogdb | {orientation=row,compression=no}
 public | film_category | table | mogdb | {orientation=row,compression=no}
 public | inventory     | table | mogdb | {orientation=row,compression=no}
 public | language      | table | mogdb | {orientation=row,compression=no}
 public | payment       | table | mogdb | {orientation=row,compression=no}
 public | rental        | table | mogdb | {orientation=row,compression=no}
 public | staff         | table | mogdb | {orientation=row,compression=no}
 public | store         | table | mogdb | {orientation=row,compression=no}
(15 rows)
```

All these tables come from the Mogila test dataset.

Compat-tools brings a large number of Oracle-compatible data dictionary views (refer to: [Oracle-Compatible Views](https://gitee.com/enmotech/compat-tools#oracle-compatibility-views)). Here are some simple examples.

```sql
-- sysdate function and dual table is supported
oracle_compatibility=# select sysdate from dual;
       sysdate
---------------------
 2024-06-06 12:47:57
(1 row)

-- V$ views are supported
oracle_compatibility=# select sid,username,status,program from v$session;
      sid       |   username    | status |        program
----------------+---------------+--------+------------------------
 47724487509760 | demo_user_kVP | active | gsql
 47724085905152 | demo_user_kVP | idle   | WLMArbiter
 47724057593600 | demo_user_kVP | idle   | WorkloadMonitor
 47724032427776 | demo_user_kVP | active | workload
 47723769759488 | demo_user_kVP | active | Asp
 47723818518272 | demo_user_kVP | idle   | statement flush thread
 47723696359168 | demo_user_kVP | idle   | CfsShrinker
 47723730437888 | demo_user_kVP | idle   | WDRSnapshot
 47723597268736 | demo_user_kVP | idle   | ApplyLauncher
 47723658610432 | demo_user_kVP | idle   | TxnSnapCapturer
 47723563190016 | demo_user_kVP | active | JobScheduler
(11 rows)

-- DBA_ views are supported
oracle_compatibility=# select * from dba_users;
   username    | user_id | password | account_status | lock_date | expiry_date |   profile
---------------+---------+----------+----------------+-----------+-------------+--------------
 demo_user_kVP |      10 | ******** | NORMAL         |           |             | DEFAULT_POOL
 MOGDB         |   18720 | ******** | NORMAL         |           |             | DEFAULT_POOL
(2 rows)

-- USER_ views are supported
oracle_compatibility=# select table_name,num_rows,last_analyzed from user_tables;
  table_name   | num_rows |         last_analyzed
---------------+----------+-------------------------------
 COUNTRY       |        0 |
 CITY          |      600 | 2024-06-06 12:13:24.504009+08
 ACTOR         |        0 |
 FILM_ACTOR    |     5462 | 2024-06-06 12:13:24.518718+08
 CATEGORY      |        0 |
 FILM_CATEGORY |     1000 | 2024-06-06 12:13:24.531421+08
 LANGUAGE      |        0 |
 FILM          |     1000 | 2024-06-06 12:13:24.56158+08
 PAYMENT       |        0 |
 CUSTOMER      |        0 |
 INVENTORY     |        0 |
 RENTAL        |        0 |
 STAFF         |        0 |
 ADDRESS       |        0 |
 STORE         |        0 |
(15 rows)

oracle_compatibility=# select count(*) from CITY;
 count
-------
   600
(1 row)

-- some short name views are supported, like IND, OBJ, COLS 
oracle_compatibility=# select index_name,table_owner,uniqueness,distinct_keys,partitioned from ind where table_name='CITY';
    index_name     | table_owner | uniqueness | distinct_keys | partitioned
-------------------+-------------+------------+---------------+-------------
 CITY_PKEY         | PUBLIC      | UNIQUE     |           600 | NO
 IDX_FK_COUNTRY_ID | PUBLIC      | NONUNIQUE  |           109 | NO
(2 rows)
```

**Note:** Due to significant differences in the underlying structure between MogDB and Oracle, these Oracle-compatible views may not include all the fields present in the corresponding Oracle database views.



**To be continued:**

\- Exploring Oracle Compatibility in MogDB (Series II) - System Functions

\- Exploring Oracle Compatibility in MogDB (Series III) - DBMS Packages

