---
layout: post
title: "Index Skip Scan Optimization in Kudu"
author: Anupama Gupta
---

This summer I got the opportunity to work as an intern with the Kudu Team at Cloudera.
My project was based on query optimization particularly in cases when the query predicate
contained the non-first columns of the composite (multi-column) primary key.

<!--more-->

Let me start by giving a brief overview of the current query flow in Kudu with an example.
Consider a table with the following schema:

Create Table metrics (
    host string,
    tstamp int,
    clusterid int,
    role string,
    PRIMARY KEY (host, tstamp, clusterid)
)

<!--more-->

In this case, by default, Kudu internally builds a primary key index (as B-Tree) for the table "metrics".
The data in the B-Tree leaf nodes is sorted lexicographically starting from the leftmost primary key column.
Therefore, when the user query contains the first key column ("host"), Kudu uses the primary key push down
operation to optimize the scan time.
However, what if the user query contains any of the non-first primary key columns ("tstamp") ? Currently, a full
table scan is done in this case (since the primary key index is primarily sorted on the basis of the first key column).
The question is, can we do better than a full table scan here ?

<!--more-->

The answer is yes ! And the idea is, if we observe the primary key index data, we find that the values for key
columns before "tstamp" column (can be generalized to any of the non-first primary key columns) form a prefix.
This allows us to seek to the next possible matching row that satisfies the predicate on the tstamp column.
For example, consider the following table rows (sorted by the key columns) and the query is :
select clusterid from metrics where tstamp = 100;

![png]({{ site.github.url }}/img/index-skip-scan/skip-scan-example-table.png){: .img-responsive}

<!--more-->

The query server can use the index to look up all rows for which host = "helium" and tstamp = 100. This method,
popularly known as index skip scan optimization can skip all the rows for which host = "helium" and tstamp != 100
(holds true for all distinct values of host such as "ubuntu", "westeros").

This optimization can speed up queries significantly (upto 13x), depending on the cardinality (no of distinct values) of the
prefix columns (in this case "host"). Lower the prefix column cardinality, better the index skip scan performance.
Based on heuristics on upto 10M rows per tablet, we decided to disable skip scan when the number of seeks
for distinct prefix keys exceeds sqrt(#total rows).

<!--more-->

The performance graph of this approach is shown below:

![png]({{ site.github.url }}/img/index-skip-scan/skip-scan-performance-graph.png){: .img-responsive}

All in all, working on a challenging project, receiving guidance from my team members and holding fruitful discussions
with them has been a memorable learning experience for me!


