
# Clickhouse Database Imports data from S3! 

I have been trying out the Clickhouse Database for a few weeks now and really impressed. For those who don't know, ClickHouse is a high-performance, columnar-oriented database management system designed primarily for real-time analytics. It was developed by Yandex and is known for its ability to process very large volumes of data quickly while using minimal system resources. ClickHouse excels in handling analytical queries with high throughput and low latency, making it a popular choice for big data, log analysis, time-series data, and online analytical processing (OLAP) tasks. Companies such as [Altinity]('https://altinity.com'), [Clickhouse]('http://clickhouse.com'), (first two who offer SaaS around Clickhouse) Craigslist, Dashdive, Deutsche Bank, and many others. [The Github repo]('https://github.com/ClickHouse/ClickHouse') is a good place to get started. 

In using it, much of my background with MySQL and PostgreSQL has helped me, plus, it actually has interfaces for MySQL and PostgreSQL clients!

## S3 Functionality

One feature I find really interesting and useful is the ability to query data from AWS S3. 

### Setup

Create a bucket:

`aws s3 mb s3://clickhouse-patg`

Copy a gzipped TSV file with data (headers in this one as well)

`aws s3 cp data.tsv.gz s3://clickhouse-patg/`

Set the server preferences with a file `/etc/clickhouse-server/config.d/s3.xml`:

```
<clickhouse>
    <s3>
        <endpoint-name>
            <endpoint>https://clickhouse-patg.s3.amazonaws.com</endpoint>
            <access_key_id>AKredacted</access_key_id>
            <secret_access_key>xyzredacted</secret_access_key>
            <!-- <use_environment_credentials>false</use_environment_credentials> -->
            <!-- <header>Authorization: Bearer SOME-TOKEN</header> -->
        </endpoint-name>
    </s3>
</clickhouse>
```

Restart the server (Ubuntu 24.04)

`systemctl restart clickhouse-server.service`

Query the table!

```

clickhouse-client --password # uses 'default' user and password set on install

myhost.internal :) select * from s3('https://clickhouse-patg.s3.amazonaws.com/data.tsv.gz','TabSeparatedRaw') limit 10;

SELECT *
FROM s3('https://clickhouse-patg.s3.amazonaws.com/data.tsv.gz', 'TabSeparatedRaw')
LIMIT 10

Query id: 3b587b74-4803-48e1-9492-13042794de55

   ┌─id	stringval	digit────────────────────┐
1. │ 0	nvlpuJWZ78rM4Ma9	3.5153584288539355 │
2. │ 1	4NY8I0R1DAjVGGHO	26.154132477041742 │
3. │ 2	9rof1FJDFg3CvAb6	60.927590847441266 │
4. │ 3	Nx1jGSRf8VOh5TSS	84.07472488753888  │
5. │ 4	PWriUaORQc32mbcP	78.96101702751513  │
6. │ 5	DxJmb0ASVZiGIeef	17.171794187401467 │
7. │ 6	fy5peZZeJqwMpt5Q	38.114516479322496 │
8. │ 7	A0LeX5v7WghiFbva	30.961514875964678 │
9. │ 8	Bqi2IInBlisHSYeS	35.935753912623916 │
   └──────────────────────────────────────────────────┘
    ┌─id	stringval	digit────────────────────┐
10. │ 9	KNHLNmxFRMAPKRMr	52.692043323314586 │
    └──────────────────────────────────────────────────┘

10 rows in set. Elapsed: 1.111 sec. 

infra.cs342cloud.internal :) 
```

Now, insert the data from S3 into a table on the server with this data:

```

INSERT INTO testdata (id, stringval, digit) SELECT *
FROM s3('https://clickhouse-patg.s3.amazonaws.com/data.tsv.gz', 'TabSeparatedRaw')

Query id: 72031373-f4bd-4fab-9164-9f5df6b30036

Ok.

0 rows in set. Elapsed: 2.718 sec. Processed 2.00 million rows, 66.00 MB (735.87 thousand rows/s., 24.28 MB/s.)
Peak memory usage: 123.58 MiB.

myhost.internal :) select * from testdata limit 10;

SELECT *
FROM testdata
LIMIT 10

Query id: 74ca3a98-0f06-4e01-9bdc-812d9b8048dd

    ┌─id─┬─stringval────────┬─────digit─┐
 1. │  0 │ nvlpuJWZ78rM4Ma9 │ 3.5153584 │
 2. │  1 │ 4NY8I0R1DAjVGGHO │ 26.154133 │
 3. │  2 │ 9rof1FJDFg3CvAb6 │  60.92759 │
 4. │  3 │ Nx1jGSRf8VOh5TSS │  84.07472 │
 5. │  4 │ PWriUaORQc32mbcP │  78.96101 │
 6. │  5 │ DxJmb0ASVZiGIeef │ 17.171795 │
 7. │  6 │ fy5peZZeJqwMpt5Q │ 38.114517 │
 8. │  7 │ A0LeX5v7WghiFbva │ 30.961515 │
 9. │  8 │ Bqi2IInBlisHSYeS │ 35.935753 │
10. │  9 │ KNHLNmxFRMAPKRMr │ 52.692043 │
    └────┴──────────────────┴───────────┘

10 rows in set. Elapsed: 0.006 sec. 

```

Very cool indeed!

# Summary

Clickhouse is an amazing database thus far, and as shown above, one of the many things it can do is utilize data in S3, in this example, query or insert.

Stay tuned for more!
