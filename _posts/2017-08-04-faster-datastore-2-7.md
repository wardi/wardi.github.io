---
layout: post
title: Faster datastore_search in CKAN
categories: [Open Data]
redirect_from: /article/2017/08/faster-datastore-ckan-2-7/
excerpt_separator: <!--more-->
---

CKAN's `datastore_search` now comes with format options and is up to **17x** faster.

This article covers:
- new total row calculation
- new result generation
- new record formats

![datastore_search Performance Improvements](/images/datastore_search_performance.png)

**TL;DR:**

1. Use CKAN 2.7 or later and resource views and other code that uses `datastore_search` is faster with no other changes required
2. Update your `datastore_search` client code to use one of the new `records_format=csv` and/or `include_total=false` options to make it much, much faster

<!--more-->

*update* This post now appears on the [ckan.org blog](https://ckan.org/2017/08/10/faster-datastore-in-ckan-2-7/)

## Total Row Calculation

The first opportunity to make DataStore search faster came when users observed search slowing down as their datasets grew. The same slow-down was not happening when using SQL search, so this problem was specific to our `datastore_search` implementation.

Profiling `datastore_search` showed that a window function used to calculate the total number of matching rows was not being cached by PostgreSQL. Instead is was recalculated for every row returned, meaning *we walked the whole table once for every row we returned*. Pulling the total calculation out of the original query more than doubled the speed for larger queries, with a slight cost for small queries (we now make two round trips to the database instead of one)

If you store tens or hundreds of millions of rows in your DataStore tables you might be interested in improving the total calculation even further by caching the total counts in Redis at the API layer or in PostgreSQL with a trigger. However CKAN 2.7 fixes the worst issue with total calculation for most large tables.

## Result Generation

Profiling `datastore_search` then showed most of our time spent handling data in Python: SQLAlchemy marshalling result rows into Python objects, DataStore walking over those objects and converting them into Python dictionaries and simplejson encoding those dictionaries to JSON to send as an API response.

The best optimisation is to do less work; can we skip all this data handling somehow?

It happens that PostgreSQL 9.3 has a rich set of JSON functions and is fully capable of building the exact response records we need. Embedding those records in our JSON API response is now supported by simplejson 3.10's new RawJSON object. This change makes our PostgreSQL query more complicated but it cuts out almost all data handling in Python, doubling our speed again for large queries.

But, can we do even less work?

There's no need to calculate the total records for cases like dumping DataStore tables to a file or updating a page of data in a resource view after the first page. With the new result generation optimisation making that part of our test query so much faster, the time simply generating a total record count now accounts for almost half the total. Let's use the new `include_total=False` parameter to skip that too. We're now producing results seven times as fast as we could on CKAN 2.6

## New Record Formats

The reason we started this optimisation work was to make dumping DataStore tables to JSON (as lists), CSV, TSV, XML formats faster. Now `datastore_search` is faster, but after it generates its results we still need to convert those results into the requested output format.

Why not generate these formats directly from PostgreSQL too? And why not let all users of the DataStore search API benefit? That's what we've done with the new `datastore_search` `records_format` parameter.

  - `records_format=objects` (the default)

    the records field will contain a list with one JSON object per record, e.g:

    ```json
    "records": [
      {"id": 1, "name": "a record", "marked": true},
      {"id": 2, "name": "another", "marked": false}
    ]
    ```

  - `records_format=lists`

    the records field will contain a list with one list per record, a more compact JSON format used when dumping tables, e.g:

    ```json
    "records": [
      [1, "a record", true],
      [2, "another", false]
    ]
    ```

  - `records_format=csv`

    the records field will contain a string containing CSV data with no header, e.g:

    ```json
    "records": "1,a record,true\r2,another,false\r"
    ```

  - `records_format=tsv`

    the records field will contain a string containing TSV (tab separated) data with no header, e.g:

    ```json
    "records": "1\ta record\ttrue\r2\tanother\tfalse\r"
    ```

XML format is not directly supported by `datastore_search` yet so it's only about 1.5x faster using `records_format=objects`. Dumping JSON is more than 2x faster using `records_format=lists`. If you need to dump large DataStore tables to XML frequently consider working on an XML `records_format` to bring its performance up to the level of JSON.

Dumping to CSV and TSV formats is 12x faster using `records_format=csv` and `records_format=tsv`, with the `datastore_search` calls themselves 17x faster.

## Acknowledgements

This work was sponsored by **OpenGov** and incorporated into CKAN 2.7 with the help of CKAN core developers and contributors.

## References

  - [CKAN](https://ckan.org/)
  - [Sponsor](https://opengov.com/)
  - [PostgreSQL JSON functions](https://www.postgresql.org/docs/9.3/static/functions-json.html)
  - [simplejson RawJSON feature](https://github.com/simplejson/simplejson/pull/143)
  - [New total calculation changes](https://github.com/ckan/ckan/pull/3467)
  - [Search performance changes](https://github.com/ckan/ckan/pull/3523)
