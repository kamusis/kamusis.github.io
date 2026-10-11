---
title: 'MTK: The Ultimate Tool for Seamlessly Migrating Oracle Databases to MogDB'
date: 2024-06-03 12:57:19
tags: [MogDB, Oracle, Database Migration]
---

## Get the latest version MTK.

```
wget https://cdn-mogdb.enmotech.com/mtk/v2.6.3/mtk_2.6.3_linux_amd64.tar.gz
tar -xvf mtk_2.6.3_linux_amd64.tar.gz
```



## Generate MTK trial license online

The trial license is valid for 1 month, and each email address can generate only one license. However, email addresses with the "enmotech.com" domain can generate licenses repeatedly. If clients wish to extend their trial of MTK beyond the initial month, they should contact Enmotech's sales or pre-sales team to request an additional 1-month license.

```
[kamus@altlinux10 mtk_2.6.3_linux_amd64]$ ./mtk license gen
License File Not Found (default license.json)
The License code is invalid, start applying
✔ Email: kamus@enmotech.com█
Start applying for email kamus@enmotech.com authorization.
Start parsing the interface to return data.
Successful application for authorization. Please check the mail and save it as license.json.
```

Upon receiving the email, upload the attached license.json file to the MTK directory. Then, use the command mtk -v to verify the license validation.

```
[kamus@altlinux10 mtk_2.6.3_linux_amd64]$ ./mtk -v
Using license file: /home/kamus/mogdb-tools/mtk_2.6.3_linux_amd64/license.json
Name: kamus@enmotech.com
Expiry: 2022-10-24 12:08:58.751194162 +0800 +0800
License key verified!
License checks OK!

MMMMMMMM               MMMMMMMMTTTTTTTTTTTTTTTTTTTTTTTKKKKKKKKK    KKKKKKK
M:::::::M             M:::::::MT:::::::::::::::::::::TK:::::::K    K:::::K
M::::::::M           M::::::::MT:::::::::::::::::::::TK:::::::K    K:::::K
M:::::::::M         M:::::::::MT:::::TT:::::::TT:::::TK:::::::K   K::::::K
M::::::::::M       M::::::::::MTTTTTT  T:::::T  TTTTTTKK::::::K  K:::::KKK
M:::::::::::M     M:::::::::::M        T:::::T          K:::::K K:::::K
M:::::::M::::M   M::::M:::::::M        T:::::T          K::::::K:::::K
M::::::M M::::M M::::M M::::::M        T:::::T          K:::::::::::K
M::::::M  M::::M::::M  M::::::M        T:::::T          K:::::::::::K
M::::::M   M:::::::M   M::::::M        T:::::T          K::::::K:::::K
M::::::M    M:::::M    M::::::M        T:::::T          K:::::K K:::::K
M::::::M     MMMMM     M::::::M        T:::::T        KK::::::K  K:::::KKK
M::::::M               M::::::M      TT:::::::TT      K:::::::K   K::::::K
M::::::M               M::::::M      T:::::::::T      K:::::::K    K:::::K
M::::::M               M::::::M      T:::::::::T      K:::::::K    K:::::K
MMMMMMMM               MMMMMMMM      TTTTTTTTTTT      KKKKKKKKK    KKKKKKK

Release version: v2.6.3
Git Commit hash: da0ed8ee
Git Commit Date: 2022-09-22T01:17:49Z
Git Tag        : v2.6.3
Build timestamp: 20220922011907
```



## Install Oracle instant client

MTK requires the [Oracle Instant Client](https://www.oracle.com/database/technologies/instant-client/downloads.html) to migrate Oracle objects to MogDB. In this tutorial, we will download the [Oracle Instant Client for Linux x86-64 Basic Package](https://download.oracle.com/otn_software/linux/instantclient/217000/instantclient-basic-linux.x64-21.7.0.0.0dbru.zip). After downloading, unzip the file and set the `LD_LIBRARY_PATH` parameter as follows:

```bash
export LD_LIBRARY_PATH=/home/kamus/instantclient_21_7:$LD_LIBRARY_PATH
```

In this tutorial, we will migrate a sample schema "customer_orders" to MogDB. The scripts for Oracle can be downloaded from the [db-sample-schemas](https://github.com/oracle-samples/db-sample-schemas) repository on GitHub.



![image-20220924140451621](https://s2.loli.net/2022/09/24/mKnfuY28shRNI9X.png)



## Initialize migration project

```
./mtk init-project -s oracle -t mogdb -n ora2mogdb
```



## Modify MTK configuration file

Modify the example MTK configuration file stored in the `project_name_dir/config` directory. Refer to the [MTK documentation](https://docs.mogdb.io/en/mtk/v2.0/mtk-parameter) for detailed information on each parameter. The essential configuration sections for MTK are `source`, `target`, and `object`.

- **`source` section**: This defines the connection to the source database. MTK needs to query the Oracle database dictionary to retrieve DDL. Therefore, it is typically recommended to use a DBA user, with the default system user being sufficient.

- **`target` section**: This defines the connection to the target database.

- **`object` section**: To migrate all objects within a schema, simply specify the schema name in the `schemas` section.

The `mtk.json` configuration file should look like this:

```json
{
    "source": {
        "type": "oracle",
        "connect": {
            "version": "",
            "host": "119.3.182.31",
            "user": "system",
            "port": 15221,
            "password": "oracle",
            "dbName": "ORACLE21C",
            "charset": "",
            "clientCharset": ""
        }
    },
    "target": {
        "type": "mogdb",
        "connect": {
            "version": "",
            "host": "127.0.0.1",
            "user": "co",
            "port": 26000,
            "password": "Enmo@123",
            "dbName": "postgres",
            "charset": "",
            "clientCharset": ""
        }
    },
    "object": {
        "tables": [],
        "schemas": ["co"],
        "excludeTable": {},
        "tableSplit": {}
    },
    "dataOnly": false,
    "schemaOnly": false
}
```

For this tutorial, we plan to migrate all objects in the "CO" schema from the Oracle database to the same user in MogDB. For testing purposes, we will not create a new database in MogDB. Instead, we will create a new user "co" in the default database, `postgres`.

Run the following command to create the user:

```bash
[omm@altlinux10 ~]$ gsql -d postgres -p 26000 -r
gsql ((MogDB 3.0.2 build 9bc79be5) compiled at 2022-09-18 00:37:49 commit 0 last mr  )
Non-SSL connection (SSL connection is recommended when requiring high-security)
Type "help" for help.

MogDB=# create user co identified by "Enmo@123";
CREATE ROLE
```



## Start migration

Now, we can start migration.

```
./mtk -c ora2mogdb/config/mtk.json
```



## Check migration report 

The migration result report will be generated in the project report directory, available in both plain text and HTML formats. For simplicity, I have included the text format result in this tutorial.

```
-----------------------
ObjectName Type Summary
-----------------------

+------------------+-------------------+-------------------+--------+-----------+-------------+-------------+-------------+--------------------|-------------+
|       Type       |     StartTime     |      EndTime      | Status | Total Num | Success Num | Warring Num | Failed  Num |Failed(Invalid) Num | Time        |
+------------------+-------------------+-------------------+--------+-----------+-------------+-------------+-------------+--------------------|-------------+
|Schema            |2022-09-24 15:12:36|2022-09-24 15:12:36|finish  |1          |1            |0            |0            |0                   |282 ms       ｜
|Sequence          |2022-09-24 15:12:36|2022-09-24 15:12:36|finish  |0          |0            |0            |0            |0                   |210 ms       ｜
|ObjectType        |2022-09-24 15:12:36|2022-09-24 15:12:36|finish  |0          |0            |0            |0            |0                   |356 ms       ｜
|Queue             |2022-09-24 15:12:36|2022-09-24 15:12:37|finish  |0          |0            |0            |0            |0                   |177 ms       ｜
|Table             |2022-09-24 15:12:37|2022-09-24 15:12:47|finish  |7          |7            |0            |0            |0                   |9 s 952 ms   ｜
|TableData         |2022-09-24 15:12:47|2022-09-24 15:12:53|finish  |7          |7            |0            |0            |0                   |6 s 743 ms   ｜
|Index             |2022-09-24 15:12:53|2022-09-24 15:12:53|finish  |7          |7            |0            |0            |0                   |1 ms         ｜
|Constraint        |2022-09-24 15:12:53|2022-09-24 15:12:53|finish  |24         |23           |0            |1            |0                   |51 ms        ｜
|DBLink            |2022-09-24 15:12:53|2022-09-24 15:12:53|finish  |0          |0            |0            |0            |0                   |67 ms        ｜
|View              |2022-09-24 15:12:53|2022-09-24 15:12:54|finish  |4          |2            |0            |2            |0                   |723 ms       ｜
|MaterializedView  |2022-09-24 15:12:54|2022-09-24 15:12:54|finish  |0          |0            |0            |0            |0                   |138 ms       ｜
|Function          |2022-09-24 15:12:54|2022-09-24 15:12:54|finish  |0          |0            |0            |0            |0                   |113 ms       ｜
|Procedure         |2022-09-24 15:12:54|2022-09-24 15:12:55|finish  |0          |0            |0            |0            |0                   |109 ms       ｜
|Package           |2022-09-24 15:12:55|2022-09-24 15:12:55|finish  |0          |0            |0            |0            |0                   |77 ms        ｜
|Trigger           |2022-09-24 15:12:55|2022-09-24 15:12:55|finish  |0          |0            |0            |0            |0                   |404 ms       ｜
|Synonym           |2022-09-24 15:12:55|2022-09-24 15:12:55|finish  |0          |0            |0            |0            |0                   |74 ms        ｜
|TableDataCom      |2022-09-24 15:12:55|2022-09-24 15:12:56|finish  |7          |7            |0            |0            |0                   |810 ms       ｜
|AlterSequence     |2022-09-24 15:12:56|2022-09-24 15:12:56|finish  |0          |0            |0            |0            |0                   |71 ms        ｜
|CollStatistics    |2022-09-24 15:12:56|2022-09-24 15:12:56|finish  |7          |7            |0            |0            |0                   |29 ms        ｜
+------------------+-------------------+-------------------+--------+-----------+-------------+-------------+-------------+--------------------|-------------+
```



We can see that all tables and table data have been successfully migrated to MogDB without any errors. However, there is one failed constraint and two failed views.

The failed constraint is a JSON check constraint, which is not supported by MogDB.

![image-20220924153609138](https://s2.loli.net/2022/09/24/RDKmsNFvroWTPgZ.png)

The failed views are due to the `grouping_id` function and the `json_table` function, which are not yet implemented in MogDB ().

| SQL                                                          | ERROR                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image-20220924154621375](https://s2.loli.net/2022/09/24/4BMfnQ6alrqhvd8.png) | pq: function grouping_id(character varying, character varying) does not exist |
| ![image-20220924154529308](https://s2.loli.net/2022/09/24/D6kHROy3vFULnQA.png) | pq: syntax error at or near "columns"                        |



## Check migration result 

Run the sample queries to ensure that all the data has been migrated to MogDB without any errors.

```
/* 5 products with the highest revenue
   With their corresponding order rank */
select p.product_name, 
       count(*) number_of_orders,
       sum ( oi.quantity * oi.unit_price ) total_value,
       rank () over ( 
         order by count(*) desc 
       ) order_count_rank
from   products p
join   order_items oi
on     p.product_id = oi.product_id
group  by p.product_name
order  by sum ( oi.quantity * oi.unit_price ) desc
fetch  first 5 rows only;
```



```
MogDB=> select p.product_name,
MogDB->        count(*) number_of_orders,
MogDB->        sum ( oi.quantity * oi.unit_price ) total_value,
MogDB->        rank () over (
MogDB(>          order by sum ( oi.quantity * oi.unit_price ) desc
MogDB(>        ) revenue_rank
MogDB-> from   products p
MogDB-> join   order_items oi
MogDB-> on     p.product_id = oi.product_id
MogDB-> group  by p.product_name
MogDB-> order  by count(*) desc
MogDB-> fetch  first 5 rows only;
     product_name      | number_of_orders | total_value | revenue_rank
-----------------------+------------------+-------------+--------------
 Girl's Trousers (Red) |              148 |    15794.76 |            1
 Boy's Hoodie (Grey)   |              100 |     3754.08 |           35
 Men's Pyjamas (Blue)  |              100 |     3274.61 |           36
 Men's Coat (Red)      |               98 |     4230.30 |           31
 Boy's Socks (White)   |               98 |     3081.12 |           38
(5 rows)
```



## Conclusion

Migrating tables, table data, and indexes from Oracle to MogDB typically proceeds without issues. However, for views, procedures, functions, and packages, some modifications to the source code may be necessary. With the ongoing development of Oracle compatibility in MogDB, we believe that the need for such modifications will decrease over time, making the migration process from Oracle to MogDB even smoother.
