---
title: >-
  Ensuring Data Integrity: How Major Databases Handle Partial Writes and Atomic
  Operations
date: 2024-05-30 19:22:37
tags: [MogDB, openGauss, Oracle, PostgreSQL, MySQL, Partial Writes]
---

In the world of databases, ensuring atomicity in write operations is crucial. This post explores the importance of atomic writes and how various databases handle potential issues like partial writes.

#### Understanding Partial Writes

In the I/O write chain, the size of atomic operations can vary. An operation at a higher level may be split into multiple operations at a lower level, leading to scenarios where some operations succeed while others fail. This phenomenon is known as a partial write.

- **Legacy Disk Drives**: Default sector size is 512 bytes.
- **Modern Disk Drives**: Advanced format sector size is 4k.
- **Filesystem Page Size**: For example, ext4 has a default page size of 4k.
- **Database Page Sizes**:
  - Oracle: 8K
  - MySQL: 16K
  - PostgreSQL: 8K
  - openGauss: 8K

### The Reality of Partial Writes

While theoretically, databases must handle partial writes, in practice, such events are quite rare. The critical aspect is not just preventing partial writes but ensuring robust mechanisms for data integrity checks (checksums) and recovery when issues do arise.

### How Oracle Handles Partial Writes

Oracle's approach to partial writes is pragmatic:

1. **Automatic Repair**: If a fractured block is detected, Oracle tries to repair it using the corresponding block from Active Data Guard (ADG) or the flashback log.
2. **Error Reporting**: If automatic repair fails, an ORA-01578 error is logged, prompting the DBA to perform a media recovery.
3. **Block-Level Recovery**: Oracle supports block-level recovery, making the process less cumbersome and time-consuming.

Oracle’s backup sets are designed to handle partial writes:

- **Using OS Commands**: When backing up data files using OS commands, Oracle requires the tablespace to be in backup mode. Any changes to data blocks are logged in the redo log, ensuring a clean block can be restored during recovery.
- **Using RMAN**: RMAN handles partial writes by ensuring that each backup block is clean, eliminating the need to worry about fractured blocks.
- **Block Checking**: Oracle provides mechanisms like DB_BLOCK_CHECKING for comprehensive checksums.

### How MySQL Handles Partial Writes

MySQL employs a **double write buffer** to safeguard against partial writes. 

1. **Double Write Buffer**: Despite its name, the double write buffer is not stored in memory; instead, it consists of persistent files on disk. The mechanism works as follows: before writing a data block to its final location on disk, MySQL first writes it to the double write buffer.
2. **Data Integrity**: This approach ensures that there is always a clean copy of the block available—either in the double write buffer or in the data files—allowing for safe redo operations.

However, this method is not without its challenges. One of the primary concerns is its efficiency, as it effectively doubles the I/O workload for redo logging. This has sparked ongoing discussions about the trade-offs and effectiveness of using a double write buffer to address partial writes.

### How PostgreSQL Handles Partial Writes

PostgreSQL employs **full page writes** to manage partial writes effectively:

1. **Full Page Writes**: When a data block is updated for the first time after a checkpoint, the entire block is written to the WAL (Write-Ahead Logging) log. This is akin to the database always being in "Backup status," similar to issuing the `alter tablespace x begin backup` command in an Oracle database.
2. **Data Integrity**: This mechanism ensures that even if a partial write occurs, the database remains consistent because the original page can be used for redo operations.

There has been community discussion about adopting a double write buffer mechanism, but it remains an area of interest rather than active development. For more details, you can check out these discussions:

- [Discussion on Double Write Buffer Mechanism (Post 1)](https://www.postgresql.org/message-id/1962493974.656458.1327703514780.JavaMail.root@zimbra-prod-mbox-4.vmware.com)
- [Discussion on Double Write Buffer Mechanism (Post 2)](https://www.postgresql.org/message-id/20180418062240.GJ18178%40paquier.xyz)

### How MogDB Handles Partial Writes

MogDB (the enterprise edition of openGauss) combines strategies from MySQL and PostgreSQL:

1. **Double Write Buffer**: Similar to MySQL, MogDB uses a double write buffer by setting `enable_double_write=on`.
2. **Full Page Writes**: MogDB retains PostgreSQL’s full page write mechanism by setting `full_page_writes=on`.

Depending on the configuration of incremental checkpoints (enable_incremental_checkpoint):

- **Incremental Checkpoints Off**:
  - With `full_page_writes=on`, full page writes are used.
  - With `full_page_writes=off`, partial writes are not a concern.
- **Incremental Checkpoints On**:
  - With `enable_double_write=on`, double writes are used.
  - With `enable_double_write=off`, the behavior depends on full_page_writes settings.

### Conclusion

Ensuring atomic writes in databases is essential for data integrity and consistency. While partial writes are rare, robust mechanisms for detection and recovery are vital. Oracle, MySQL, PostgreSQL, and MogDB/openGauss each have unique approaches to handling these scenarios, ensuring that data remains reliable and recoverable.
