---
title: Multiregion Overview
summary: Learn how to use CockroachDB's improved multiregion user experience.
toc: true
---

# Overview

CockroachDB v21.1+ has improved multi-region capabilities that make it easier to run global applications on CockroachDB.  To take advantage of these improved capabilities, you will need to understand the following concepts:

- _Database Regions_, which provide an abstraction for cloud provider regions and zones.
- _Survivability Goals_, which dictate the blast radius of failure(s) that a database can survive.
- _Table Localities_, which determine in which region(s) a table's underlying data is stored.

At a high level, the simplest process for running a multi-region cluster is:

1. Set up node regions at startup using CLI locality flags.
2. Add a primary region to a database, making it a "multi-region" database.
3. (*Optional*) Change the database's survivability goals (zone or region).  This step is optional because by default multi-region databases will be configured to survive zone failures.
4. (*Optional*) Change table localities (global, regional by table, regional by row).  This step is optional because by default the tables in a database will be homed in the database's primary region (as set during step 1 above).

The steps above describe the simplest case, where you accept all of the default settings.  These settings will provide the best experience for most users.

If your application has performance or availability needs that are different than what the default settings provide, there are many more options to explore.

For more information about CockroachDB's multi-region capabilities and the tuning options that are available, see below.

(<strong><font style="size:xx-large" color="red">XXX</font></strong>: INTEGRATE THE INFO BELOW, SOMEHOW)

I do think it’s interesting to map out what would lead a user to change zero, one, or both of these knobs once a database’s regions are added. Doing so does indicate that many people, if they are bothering to set up a multi-region database, will decide to change one of these. But not everyone. I think the chart would look something like:

- change nothing: zone survivability, single-region writes, multi-region stale reads (otherwise, why make the database multi-region?). This isn’t a crazy setup - it’s essentially what Aurora global database gives you. It would even be pretty reasonable if/once we can recover from insufficient quorum situations (with data loss).

- change only survivability: region survivability, single-region writes, (optionally) multi-region stale reads. Useful for single-region apps that want higher levels of survivability.

- change only table locality: zone survivability, multi-region reads and writes. Useful for multi-region apps that want low-latency writes and are not concerned with surviving a region failure.

- change both survivability and table locality: region survivability, multi-region reads and writes. Useful for multi-region apps that want a high level of survivability.

## Database regions

 _Database regions_ are a high-level abstraction for a geographic region.  Each region is broken into multiple zones.  These terms are meant to correspond directly to the region and zone terminology used by cloud providers.

Regions and zones are defined at the cluster level using the following [node startup locality options|https://www.cockroachlabs.com/docs/v21.1/cockroach-start.html]:

- Regions are specified using the {{region}} key, and
- Zones through the use of the {{zone}} key

The regions added during node startup become _Database Regions_ when they are added to a database using the following statement:

{code}
ALTER DATABASE <db> SET PRIMARY REGION <region>
{code}

Now that the database has a primary region assigned to it, it is a "multi-region database."  All data in that database is stored within its assigned regions, with the majority of its data in the primary region.

{info}
NOTE: If you are happy to accept the default _Survivability Goals_ and _Table Localities_ (both described below), there is nothing else you need to do once you have set a primary database region.  The defaults should work well for most users.
{info}

## Survivability goals

All tables within the same database operate with the same survivability goal.  Each database is allowed to have its own survivability goal setting.

The following survivability goals are available:

- Zone failure
- Region failure

Surviving zone failures is the default.  You can upgrade a database to survive region failures at the cost of slower write performance (due to network hops) using the following statement:

{code}
ALTER DATABASE <db> SURVIVE REGION FAILURE
{code}

For more information about these options, see below.

##* Surviving zone failures

The zone level survivability goal ensures that:

1. If a single zone is unavailable, the database will remain fully available for reads and writes.
2. If multiple zones fail in the same region, it's possible that the database will become unavailable.

{info}
NOTE: Surviving zone failures is the default setting for multi-region databases.  You do not need to change any settings to survive zone failures.  This setting should work well for most users.
{info}

##* Surviving region failures

The region level survivability goal ensures that:

1. The database will remain available when an entire region (including multiple availability zones) is unavailable,
2. ... at the cost that write latency will be increased by at least as much as the round-trip time to the nearest region. (In other words, you are adding network hops and making things slower in exchange for robustness.)
3. Read performance will be unaffected.

As mentioned above, you can upgrade a database to survive region failures using the following statement:

{code}
ALTER DATABASE <db> SURVIVE REGION FAILURE
{code}

## Table locality

Every table in a multi-region database has a "table locality setting" applied to it.  The table locality setting determines where the majority of the table's underlying data is located.

By default, all tables in a multi-region database are _regional_ tables -- that is, the majority of the table's data will be homed in the database's primary region.

{info}
NOTE: Table locality settings are used for optimizing latency under different read/write patterns.  If you are happy with the performance tradeoffs of the default setting (regional tables), there is nothing else you need to do once you set your database's primary region as described above.  The default setting should work well for most users.
{info}

##* Regional tables

For _regional_ tables, access to the table will be fast in the table's "home region" and slower in other regions.  In other words, regional tables are optimized for access from a single region.  By default, a regional table's home region is the database's primary region (described above).

Regional tables work well when your application requires low-latency reads and writes for an entire table in a single region.

{info}
NOTE: By default, all tables in a multi-region database are _regional_ tables that use the database's primary region.  Unless you know your application needs different performance characteristics than regional tables provide, there is no need to change this setting.
{info}

##* Global tables

 _Global_ tables are optimized for low-latency reads from every region in the database.  The tradeoff is that writes will incur higher latencies from any given region, since writes have to be replicated across every region to make the global low-latency reads possible.

Use global tables when your application has a "read-mostly" table of reference data (such as postal codes) that is rarely updated, and needs to be available to all regions.

For an example of a table that can benefit from the _global_ table locality setting in a multi-region deployment, see the {{promo_codes}} table from the [MovR application|https://www.cockroachlabs.com/docs/v21.1/movr].

##* Regional by row tables

In _regional by row_ tables, individual rows are optimized for access from different regions.  This setting divides a table and all of its indexes into [partitions|https://www.cockroachlabs.com/docs/v21.1/partitioning.html], with each partition optimized for access from a different region.

Use regional by row tables when your application requires low-latency reads and writes where rows are accessed from different regions.  For example, a users table in a global application may need to keep some users' data in specific regions due to regulations (such as GDPR), for better performance, or both.

For an example of a table that can benefit from the _regional by row_ setting in a multi-region deployment, see the {{users}} table from the [MovR application|https://www.cockroachlabs.com/docs/v21.1/movr].

When accessing regional by row tables from your application, the optimal region for a given row can be specified in one of the following ways:

- Explicitly by the application as part of an {{INSERT ... WHERE crdb_region = <region>}} statement
- Automatically, depending on the region of the gateway node from which the row has been inserted.  To make this determination, the builtin function {{gateway_region()}} is invoked to determine the gateway node's region

Once set, a row's home region will not be changed unless you modify it explicitly with an {{UPDATE}} statement.  When you update a row's region, CockroachDB moves it to the specified region.

{note}
NOTE: The _regional by row_ table locality setting is the most complex setting to understand, configure, and use correctly.  You should only use this setting if you know your application needs the specific performance characteristics it can provide.
{note}

## See also

- [Multiregion overview](multiregion-overview.md)
- [Multiregion `REGIONAL` tables](multiregion-regional-tables.md)
- [Multiregion `GLOBAL` tables](multiregion-global-tables.md)
- [Multiregion `REGIONAL BY ROW` tables](multiregion-regional-by-row-tables.md)

<!-- EOF -->
