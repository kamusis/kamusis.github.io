---
title: >-
  Some Misconceptions About Database Performance - Application Concurrent
  Requests, Database Connections, and Database Connection Pools
date: 2024-08-21 17:44:47
tags: [PostgreSQL, Database, Performance]
---

Let's start with a chart from a database stress test result.

![1724216413686](https://s2.loli.net/2024/08/21/Ra2u1Q9cJNpOlUm.png)

In the chart above, the X-axis represents the number of concurrent connections, and the Y-axis represents the TPS (Transactions Per Second) provided by the database. The stress testing tool used is pgbench, and the database is a single instance of PostgreSQL 16. TPS_PGDirect represents the TPS when pgbench directly sends the load to PostgreSQL, while TPS_PGBouncer represents the TPS when pgbench sends the load to the PGBouncer database connection pool.

We can clearly see that when the number of connections exceeds 256, the TPS begins to gradually decline when the load is sent directly to the database. When the number of connections eventually rises to over 8000, the database is almost at a standstill. However, when using PGBouncer, the TPS remains stable and excellent, with the database consistently providing high TPS regardless of the number of connections.

Many people might conclude from this chart that if the concurrent connections increase beyond a certain threshold, PostgreSQL becomes inadequate, and PGBouncer must be used to improve performance.

Some users may attempt to conduct various hardware environment combination tests to derive a reference threshold number to guide at what pressure PostgreSQL can cope, and once it exceeds a certain pressure, PGBouncer should be enabled.

So, is this conclusion correct? And does testing out a threshold number have real guiding significance?

## Understanding Concurrent Performance

To determine how many concurrent business requests a database can handle, we should focus on the Y-axis from the test chart above, rather than the X-axis. The Y-axis TPS represents the number of transactions the database can process per second, while the X-axis connection count is merely the database connection number we configured ourselves (which can be simply understood as the max_connections parameter value). We can observe that at 256 connections, the direct database stress test with pgbench allows the database to provide over 50,000 TPS, which is actually almost the highest number in the entire stress test (WITHOUT PGBouncer). Subsequent tests using PGBouncer, this number was not significantly exceeded. Therefore, PGBouncer is not a weapon to improve database performance.

What is performance? Performance means: for a determined business transaction pressure (how much work a business transaction needs to handle, such as how much CPU, memory, and IO it consumes), the more transactions the database can execute, the higher the performance. In the chart above, this is represented by the Y-axis TPS (Transactions Per Second).

For example, suppose a user request (Application Request) requires executing 5 transactions from start to finish, and assume these transactions consume similar amounts of time (even if there are differences, we can always get the average transaction runtime in a system). In this case, 50,000 TPS means the database can handle 10,000 user requests per second, which can be simply understood as handling 10,000 concurrent users. If TPS rises to 100,000, the database can handle 20,000 concurrent users; if TPS drops to 10,000, it can only handle 2,000 concurrent users. This is a simple calculation logic, and in this description, we did not mention anything about database connections (User Connections).

So why does it seem like PGBouncer improves performance when the number of connections continues to increase? Because in this situation, PGBouncer acts as a limiter on database connections, reducing the extra overhead caused by the database/operating system handling too many connections, allowing the database and operating system to focus on processing the business requests sent from each connection, and enabling user requests exceeding the connection limit to share database connections. What PGBouncer does is: if your restaurant can only provide 10 tables, then I only allow 10 tables of guests to enter, and other guests have to wait outside.

However, why should we let 100 tables of guests rush into the restaurant all at once? Without PGBouncer, can't we control allowing only 10 tables of guests to enter? In modern applications, application-side connection pools are more commonly used, such as connection pools in various middleware software (Weblogic, WebSphere, Tomcat, etc.) or dedicated JDBC connection pools like HikariCP.

Exceeding the number of connections that can be handled is unnecessary. When the database is already at its peak processing capacity, having more connections will not improve overall processing capability; instead, it will slow down the entire database.

To summarize the meaning of the chart above more straightforwardly: when the total number of database connections is at 256, the database reaches its peak processing capacity, and direct database connections can provide 50,000 TPS without any connection pool. If 50,000 TPS cannot meet the application's concurrent performance requirements, increasing the number of connections will not help. But if connections are increased (wrongly, without any other control), a layer of connection pool like PGBouncer can be added as a connection pool buffer to keep performance stable at around 50,000 TPS.

So, do we have any reason to arbitrarily increase the number of connections? No. What we need to do is: allow the database to open only the number of connections within its processing capacity and enable these connections to be shared by more user requests. When the restaurant size is fixed, the higher the turnover rate, the higher the restaurant's daily revenue.

## How to Ask Questions About Performance

Instead of asking: how can we ensure the database provides sufficient TPS with 4000 connections, we should ask: how can we ensure the database provides 50,000 TPS.

Because the number of connections is not a performance metric, it is merely a parameter configuration option.

In the example above, a connection count of 4000 or 8000 is not a fixed value, nor is it an optimized value. We cannot try to do more additional database optimizations on an unoptimized, modifiable parameter setting, as this may lead to a reversal of priorities. If we can already provide maximum processing capacity with 256 fully loaded database connections, why optimize a database with more than 256 connections? We should ensure the database always runs at full capacity with a maximum of 256 connections.

Now, our question about performance can be more specific. Suppose we have multiple sets of database servers with different configurations; what should the fully loaded connection count be for each set to achieve optimal processing capacity?

This requires testing. However, we will ultimately find that this fully loaded connection count is usually related only to the database server's hardware resources, such as the number of CPU cores, CPU speed, memory size, storage speed, and the bandwidth and latency between storage and host.

## Is Database Application Performance Only Related to Hardware Resources?

Of course not. Concurrent performance (P) = Hardware processing capability (H) / Business transaction pressure (T).

As mentioned earlier, the performance metric TPS is for a "determined business transaction pressure," meaning a determined T value.

When the T value is determined, we can only improve the H value to further enhance the P value. Conversely, there is naturally another optimization approach: reducing the T value. There are various ways to reduce the T value, such as adding necessary indexes, rewriting SQL, using partitioned tables, etc. The optimization plan should be formulated based on where the T value is consumed most, which is another topic.

## Conclusion

To answer the two initial questions:

1. If concurrent connections increase beyond a certain threshold, PostgreSQL becomes inadequate, and PGBouncer must be used to improve performance. Is this conclusion correct?

If we must create more connections to the database than the hardware processing capacity for certain hardware resources, using PGBouncer can stabilize performance. However, we have no reason to create more connections than the hardware processing capacity. We have various means to control the maximum number of connections the database can open, and various middleware solutions can provide connection pools. From this perspective, PGBouncer does not improve database performance; it merely reduces concurrency conflicts, doing what any connection pool software can do.

2. Is it meaningful to attempt various hardware environment combination tests to derive a reference threshold number to guide at what pressure PostgreSQL can cope, and once it exceeds a certain pressure, PGBouncer should be enabled?

It has some significance, but simple stress tests like pgbench are not enough to guide real applications. Concurrent performance (P) = Hardware processing capability (H) / Business transaction pressure (T). Testing in a determined hardware environment means the H value is fixed, so we should focus on T during testing. The T value provided by pgbench is a very simple transaction pressure (only four tables, simple CRUD operations), and the P value tested under this pressure can only be used as a reference. To guide application deployment, typical transaction pressure of the application itself must be abstracted for testing to have guiding significance.
