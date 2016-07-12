# RTFAP - Real-time Fraud Analysis Platform

A large bank wants to monitor its customer creditcard transactions to detect and deter fraud attempts. They want the ability to search and group transactions by merchant, credit card provider, amounts values. This is subject to change.

The client wants a REST API to return:  

- Identify all transactions tagged as fraudulent in the last minute/day/month/year.
- Identify all transactions tagged as fraudulent for a specific card.
- Report of transactions for a merchant on a specific day.
- Roll-up report of transactions by card and year.
- Search capability to search the entire transaction database by merchant, cc_no, amounts.
- The ratio of transaction success based on the first 6 digits of their credit card no.     
- The ratio of confirmed transactions against fraudulent transactions in the last minute.
- A moving ratio of approved transactions per minute, per hour.
- A count of approved transactions per minute, per hour.

Performance SLAs:
- The client wants assurance that his data model can handle 1,000 transactions a sec with stable latencies. The client currently handles accounts for over 15000 merchants and hoping to grow to 50,000 in a year.

![alt text] (https://raw.githubusercontent.com/carolinerg1/FraudPrevention/master/img.png)

##Setup
DataStax Enterprise supplies built-in enterprise search functionality on Cassandra data that scales and performs in a way that meets the search requirements of modern Internet Enterprise applications. Using this search functionality will allow the volume of transactions to grow without a loss in performance. DSE Search also allows for live indexing for improved index throughput and reduced reader latency. More details about live indexing can be found here -  http://docs.datastax.com/en/datastax_enterprise/4.8/datastax_enterprise/srch/srchConfIncrIndexThruPut.html

We will need to start DSE in Search mode to allow us to use the search functionalities that we need on top on Cassandra. To do this see the following
https://docs.datastax.com/en/datastax_enterprise/4.8/datastax_enterprise/srch/srchInstall.html
and the Spark (Analytics) functionality using:
http://docs.datastax.com/en/datastax_enterprise/4.8/datastax_enterprise/spark/sparkTOC.html

For the packathon - the cluster info is below:

- public ip's of 3 node cluster=> 104.42.107.223, 104.42.105.51, 104.42.109.110
- SSH to any one of the 3 nodes => e.g. ssh datastax@104.42.109.110; pwd=<<on internal hipchat room, scroll a bit>>
(private ip= 10.0.0.4 and other 2 DSE nodes will have 10.0.0.5 and 10.0.0.7; 10.0.0.6 is the opscenter node)
- Opscenter=> http://104.42.108.173:8888/opscenter/index.html
- Spark Master (currently running on) => http://104.42.105.51:7080/
- Jupyter notebook with RTFAP Test queries=> http://104.42.109.110:8084/notebooks/RTFAP%20Test%20Queries.ipynb#
- Visual Dashboard => http://104.42.109.110:8983/banana/#/dashboard

##DataModel

We will need multiple tables to fulfill the above query patterns and workloads (de-normalization is a good thing with NoSQL databases!).

We will use a single DC for testing purposes. For production deployment, we recommend an Active-Active HA setup across geographical regions with RF=3.
```
create keyspace if not exists rtfap WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '1' };
```

Table for: Transactions by txn_time buckets; We will create a Solr index on this tables to fulfill a bunch of search needs as well.
```
create table if not exists rtfap.transactions(
	cc_no text,
	cc_provider text,
	year int,
	month int,
	day int,
	hour int,
	min int,
	txn_time timestamp,
 	txn_id text,
 	user_id text,
	location text,
	items map<text, double>,
	merchant text,
	amount double,
	status text,
	notes text,
	tags set<text>,
	PRIMARY KEY ((cc_no, year, month,day), txn_time)
) WITH CLUSTERING ORDER BY (txn_time desc);
```

Table for: Roll-up of Daily transactions by merchant
```
create table if not exists rtfap.dailytxns_bymerchant(
	cc_no text,
	cc_provider text,
	day int,
	txn_time timestamp,
 	txn_id text,
 	user_id text,
	location text,
	items map<text, double>,
	merchant text,
	amount double,
	status text,
	notes text,
	tags set<text>,
	total_amount double STATIC,
	max_amount double STATIC,
	min_amount double STATIC,
	total_count bigint STATIC,
	PRIMARY KEY ((merchant, day), txn_time, txn_id)
) WITH CLUSTERING ORDER BY (txn_time desc);
```

Table for: hourlyaggregates_bycc
```
create table if not exists rtfap.hourlyaggregates_bycc(
    cc_no text,
    hour int,
    total_amount double,
    max_amount double,
    min_amount double,
    total_count bigint,
    PRIMARY KEY ((cc_no, hour))
);
```

Table for: dailyaggregates_bycc
```
create table if not exists rtfap.dailyaggregates_bycc(
    cc_no text,
    day int,
    total_amount double,
    max_amount double,
    min_amount double,
    total_count bigint,
    PRIMARY KEY ((cc_no, day))
);
```

Table for: monthlyaggregates_bycc
```
create table if not exists rtfap.monthlyaggregates_bycc(
    cc_no text,
    month int,
    total_amount double,
    max_amount double,
    min_amount double,
    total_count bigint,
    PRIMARY KEY ((cc_no, month))
);
```

Table for: yearlyaggregates_bycc
```
create table if not exists rtfap.yearlyaggregates_bycc(
    cc_no text,
    year int,
    total_amount double,
    max_amount double,
    min_amount double,
    total_count bigint,
    PRIMARY KEY ((cc_no, year))
);
```

##Sample inserts

```
insert into rtfap.transactions (year, month, day, hour, min, txn_time, cc_no, amount, cc_provider, items, location, merchant, notes, status, txn_id, user_id, tags) VALUES ( 2016, 03, 09, 11, 04, '2016-03-09 11:04:19', '1234123412341234', 200.0, 'VISA', {'tshirt':25, 'dressshirt':50, 'trousers':125}, 'San Francisco', 'Nordstrom', 'pretty good clothing', 'Approved', '098765', 'kunalak', {'Suspicious'});
insert into rtfap.transactions (year, month, day, hour, min, txn_time, cc_no, amount, cc_provider, items, location, merchant, notes, status, txn_id, user_id, tags) VALUES ( 2016, 03, 09, 11, 04, '2016-03-09 11:04:24', '1234123412341235', 400.0, 'VISA', {'cap':25, 'lamps':275, 'trousers':100}, 'San Diego', 'Macy', 'cool stuff-good customer', 'Rejected', '876354', 'simonanbridge', {'Fraudulent'});
insert into rtfap.transactions (year, month, day, hour, min, txn_time, cc_no, amount, cc_provider, items, location, merchant, notes, status, txn_id, user_id, tags) VALUES ( 2016, 03, 09, 11, 04, '2016-03-09 11:04:53', '1234123412341235', 800.0, 'VISA', {'chocolates':300, 'electronics':500}, 'London', 'Harrods', 'customer likes electronics', 'Approved', '982538', 'simonanbridge', {'HighValue'});
insert into rtfap.transactions (year, month, day, hour, min, txn_time, cc_no, amount, cc_provider, items, location, merchant, notes, status, txn_id, user_id, tags) VALUES ( 2016, 03, 09, 11, 04, '2016-03-09 11:04:59', '1234123412341236', 750.0, 'MASTERCARD', {'shoes':300, 'belts':150, 'clothes':300}, 'San Jose', 'GAP', 'customer likes electronics', 'Approved', '092753', 'cary', {'HighValue'});
insert into rtfap.transactions (year, month, day, hour, min, txn_time, cc_no, amount, cc_provider, items, location, merchant, notes, status, txn_id, user_id, tags) VALUES ( 2016, 03, 09, 12, 30, '2016-03-09 12:30:00', '1234123412341237', 1500.0, 'AMEX', {'clothes':1500}, 'New York', 'Ann Taylor', 'frequent customer', 'Approved', '876302', 'caroline', {'HighValue'});


```

##Sample queries

Queries to look up all transactions for given cc_no. (Transactions table is primarily writes-oriented and for searches)
```
SELECT * FROM rtfap.transactions WHERE cc_no='1234123412341234' limit 5
```
Queries to roll-up transactions for each merchant by day (dailytxns_bymerchant table itself will be populated using scheduled Spark Jobs)
```
SELECT * FROM rtfap.dailytxns_bymerchant where merchant='Nordstrom' and day=20160317
```

##Searching Data in DSE

The above queries allow us to query on the partition key and some or all of the clustering columns in the table definition. To query more generically on the other columns we will use DSE Search to index and search our data. To do this we use the dsetool to create a Solr core. We will also use the dsetool to create the core based on our table for testing purposes. In a production environment we would only index the columns that we would want to query on.

```
dsetool create_core rtfap.transactions generateResources=true reindex=true
```

To check that DSE Search is up and running sucessfully go to http://{servername}:8983/solr/

Now we can query our data in a number of ways. One is through cql using the solr_query column. The other is through a third party library like SolrJ which will interact with the search tool through ReST.

Below are the CQL Solr queries addressing some of the client requirements (&more) for searching the data in DSE:

Get counts (&records) of transactions faceted by merchant or cc_provider.
```
SELECT * FROM rtfap.transactions where solr_query='{"q":"*:*", "facet":{"field":"merchant"}}'
SELECT * FROM rtfap.transactions where solr_query='{"q":"*:*", "facet":{"field":"cc_provider"}}'
```

Get transactions by first 6 digits of cc_no (and perhaps filter query it further by the status!).
```
SELECT * FROM rtfap.transactions where solr_query='{"q":"cc_no: 123412*",  "fq":"status: Rejected"}';
```

Get all the transactions tagged as Fraudulent in the last day and last minute.
```
SELECT * FROM rtfap.transactions where solr_query = '{"q":"*:*", "fq":["txn_time:[NOW-1DAY TO *]", "tags:Fraudulent"]}'
SELECT * FROM rtfap.transactions where solr_query = '{"q":"*:*", "fq":["txn_time:[NOW-1MINUTE TO *]", "tags:Fraudulent"]}'
```
(just like above samples , full ad-hoc search on any transaction fields is possible including amounts, merchants etc.)

## Querying Data Using A ReST Web Interface

A ReSTful web interface provides an API for calling programs to query the data in Cassandra.
To use the web service, use the following url’s. These will return a json representation of the data using the ReST service.

The sample queries are served by a web service written in Java. The code for this web service is provided in the repo.
The web service adopts a framework that separates the web, service and data access layers into individual, easily maintainable components. 

To start the web service use the command:
```
$ mvn jetty:run
```
To bind to a specific interface other than localhost use:
```
$ mvn jetty:run -DcontactPoints=<server IP address>
```
For example - to run on a server with an IP of 10.0.0.4 and persist the web service after logging out use:
```
$ nohup mvn jetty:run -DcontactPoints=10.0.0.4 &
```

The queries demonstrate the use of both straightforward CQL and CQL-Solr. This can be seen in TransactionsDao.java:
- CQL queries bind the individual parameters passed from the web interface
- CQL-SOLR queries must bind the complete "where" clause as a single bind variable

### - List all the card transactions across all cards and vendors
List all the card transactions across all cards and vendors in the TRANSACTIONS table:

http://104.42.109.110:8080/datastax-banking-iot/rest/getalltransactions 
```
SELECT * FROM transactions;
```
### - List all transactions for a merchant on a specified day
Retrieve data for all transactions for the specified day in the DAILYTXNS_BYMERCHANT rollup table where the merchant = "GAP" 

http://104.42.109.110:8080/datastax-banking-iot/rest/getdailytransactionsbymerchant/GAP/20160309 
```
SELECT * FROM dailytxns_bymerchant where merchant='GAP' and day= 20160309;
```
### - Aggregated purchase history for a specific card and year
Retrieve data for all transactions for the specified year in the YEARLYAGGREGATES_BYCC rollup table where the card number = "1234123412341235"

http://104.42.109.110:8080/datastax-banking-iot/rest/getyearlytransactionsbyccno/1234123412341235/2016
```
SELECT * FROM yearlyaggregates_bycc where cc_no='1234123412341235' and year=2016;
```
### - Rolling transaction success ratio and count, by minute and hour
Retrieve a rolling ratio of successful transactions and transaction count over the previous minute and hour

http://104.42.109.110:8080/datastax-banking-iot/rest/getTransactionsApprovalByDate/201603270521
```
select approved_rate_hr, approved_txn_hr, approved_rate_min, approved_txn_min from txn_count_min where year=2016 and month=3 and day=27 and hour=5 and minute=22;
```

### - List all transactions over a specified amount
Retrieve data for all transactions in the TRANSACTIONS table where the amount is greater than a specified value

http://104.42.109.110:8080/datastax-banking-iot/rest/getalltransactionsbyamount/1000
```
SELECT * FROM rtfap.transactions where solr_query = '{"q":"*:*",  "fq":"amount:[1000 TO *]"}}'
```
### - List all rejected transactions
Retrieve all transactions in the TRANSACTIONS table where status="Rejected"

http://104.42.109.110:8080/datastax-banking-iot/rest/getallrejectedtransactions 
```
SELECT * FROM transactions where solr_query='{"q":"status: Rejected"}';
```
### - List all transactions faceted by merchant
Retrieve all transactions in the TRANSACTIONS table, faceted by merchant

http://104.42.109.110:8080/datastax-banking-iot/rest/getfacetedtransactionsbymerchant 
```
SELECT * FROM transactions where solr_query='{"q":"*:*", "facet":{"field":"merchant"}}';
```
### - List all transaction success ratio (faceted by status) in the last period e.g. MINUTE
Retrieve all transactions in the TRANSACTIONS table, faceted by status, over the last year/month/minute

http://104.42.109.110:8080/datastax-banking-iot/rest/getfacetedtransactionsbystatusinlastperiod/MINUTE
```
SELECT * FROM rtfap.transactions where solr_query = '{"q":"*:*",  "fq":"txn_time:[NOW-1" + lastPeriod + " TO *]","facet":{"field":"status"}}';
```
### - List all transaction success ratio (faceted by status) for a specified card in the last period e.g. MINUTE
Retrieve all transactions in the TRANSACTIONS table, faceted by status, for the specified card number and period

http://104.42.109.110:8080/datastax-banking-iot/rest/getfacetedtransactionsbyccnoandstatusinlastperiod/123412*/YEAR
```
SELECT * FROM rtfap.transactions where solr_query = '{"q":"cc_no:123412*",  "fq":"txn_time:[NOW-1MINUTE TO *]","facet":{"field":"status"}}';
```
### - List all transactions for a specific card
Retrieve all transactions in the TRANSACTIONS table for a specified card number (optional wild card)

http://104.42.109.110:8080/datastax-banking-iot/rest/getalltransactionsbyccno/123412*
```
SELECT * FROM transactions where solr_query='{"q":"cc_no:123412*"}';
```
### - List all fraudulent transactions for a specific card
Retrieve all transactions in the TRANSACTIONS table tagged as "Fraudulent" for a specified card number

http://104.42.109.110:8080/datastax-banking-iot/rest/getallfraudulenttransactionsbyccno/123412*
```
SELECT * FROM transactions where solr_query='{"q":"cc_no:123412*", "fq":["tags:Fraudulent"]}';
```
### - List all fraudulent transactions for a specified period-to-date
Retrieve data for all transactions in the TRANSACTIONS table tagged as "Fraudulent" over the last year

http://104.42.109.110:8080/datastax-banking-iot/rest/getallfraudulenttransactionsinlastperiod/YEAR
```
SELECT * FROM transactions where solr_query = '{"q":"*:*", "fq":["txn_time:[NOW-1YEAR TO *]", "tags:Fraudulent"]}';
```
Retrieve data for all transactions in the TRANSACTIONS table tagged as "Fraudulent" over the last month

http://104.42.109.110:8080/datastax-banking-iot/rest/getallfraudulenttransactionsinlastperiod/MONTH
```
SELECT * FROM transactions where solr_query = '{"q":"*:*", "fq":["txn_time:[NOW-1MONTH TO *]", "tags:Fraudulent"]}';
```
Retrieve data for all transactions in the TRANSACTIONS table tagged as "Fraudulent" over the last day

http://104.42.109.110:8080/datastax-banking-iot/rest/getallfraudulenttransactionsinlastperiod/DAY
```
SELECT * FROM transactions where solr_query = '{"q":"*:*", "fq":["txn_time:[NOW-1DAY TO *]", "tags:Fraudulent"]}';
```

## Analyzing data using DSE Spark Analytics

DSE provides integration with Spark out of the box. This allows for ETL'ing and analyses of data in-place on the same cluster where the data is ingested with workload isolation. The data ingested in a Cassandra only (oltp) ring is automatically replicated to the logical ring of nodes hosting Spark Workers as well.

This provides huge value in terms of significantly reduced ETL complexity (no data movement to different clusters) and thus increasing time to insight from your data through a "cohesive lambda architecture" sans its complexities.

###Batch Analytics

A Spark batch job that runs daily rolling up all the transactions in the last day by merchant and calculating the total_amount, avg_amount and total_count.
(Daily Roll-Up Reports of last-Week and last-Day transactions for each merchant.)

```
sqlContext.sql("""CREATE TEMPORARY TABLE temp_transactions
     USING org.apache.spark.sql.cassandra
     OPTIONS (
       table "transactions",
       keyspace "rtfap",
       cluster "Test Cluster",
       pushdown "true"
     )""")

val rollup1= sqlContext.sql("select txn_time, cc_no, amount, cc_provider, items, location, merchant, notes, status, txn_id, user_id, tags, int(translate(string(date(txn_time)),'-','')) as day from temp_transactions")    

rollup1 show
rollup1 printSchema

import org.apache.spark.sql.SaveMode

rollup1.write.format("org.apache.spark.sql.cassandra")
.mode(SaveMode.Overwrite)
.options(Map("keyspace" -> "rtfap", "table" -> "dailytxns_bymerchant"))
.save()
```

Jupyter notebook: http://104.42.109.110:8084/notebooks/RTFAP%20Test%20Queries.ipynb

###Streaming Analytics
Cary to update here



##Stress yaml

Running a cassandra-stress test with the appropriate YAML profile for the table helps show how DSE will perform in terms of latency and throughput for writes and reads to/from the system.

The stress YAML files are uploaded to this [directory](https://github.com/carolinerg1/FraudPrevention/tree/master/stress%20yamls).

The YAML tries to mirror real data, for example: month is a value between 1 and 12, year is between 2010 and 2016, credit card number is 16 characters in length, etc

You can read more about stress testing a data model here
http://www.datastax.com/dev/blog/improved-cassandra-2-1-stress-tool-benchmark-any-schema
http://docs.datastax.com/en/cassandra/2.1/cassandra/tools/toolsCStress_t.html

An example of running the stress tool is below using [txn_by_cc_stress.yaml](https://github.com/carolinerg1/FraudPrevention/blob/master/stress%20yamls/txn_by_cc_stress.yaml):

For inserts
```
cassandra-stress user profile=./txn_by_cc_stress.yaml ops\(insert=1\) cl=LOCAL_ONE n=100000 -rate auto -node 10.0.0.5

Results:
op rate                   : 1846 [insert:1846]
partition rate            : 1846 [insert:1846]
row rate                  : 1846 [insert:1846]
latency mean              : 2.1 [insert:2.1]
latency median            : 1.4 [insert:1.4]
latency 95th percentile   : 3.8 [insert:3.8]
latency 99th percentile   : 11.0 [insert:11.0]
latency 99.9th percentile : 28.0 [insert:28.0]
latency max               : 1753.2 [insert:1753.2]
Total partitions          : 100000 [insert:100000]
Total errors              : 0 [insert:0]
total gc count            : 0
total gc mb               : 0
total gc time (s)         : 0
avg gc time(ms)           : NaN
stdev gc time(ms)         : 0
Total operation time      : 00:00:54
```
    
Increasing the number of threads increases the op rate as expected:

```
cassandra-stress user profile=./txn_by_cc_stress.yaml ops\(insert=1\) cl=LOCAL_ONE n=100000 -rate threads=8 -node 10.0.0.4,10.0.0.5,10.0.0.7

Results:
op rate                   : 2639 [insert:2639]
partition rate            : 2639 [insert:2639]
row rate                  : 2639 [insert:2639]
latency mean              : 3.0 [insert:3.0]
latency median            : 1.7 [insert:1.7]
latency 95th percentile   : 7.0 [insert:7.0]
latency 99th percentile   : 11.3 [insert:11.3]
latency 99.9th percentile : 26.6 [insert:26.6]
latency max               : 722.7 [insert:722.7]
Total partitions          : 100000 [insert:100000]
Total errors              : 0 [insert:0]
total gc count            : 0
total gc mb               : 0
total gc time (s)         : 0
avg gc time(ms)           : NaN
stdev gc time(ms)         : 0
Total operation time      : 00:00:37
```


For reads
```
cassandra-stress user profile=./txn_by_cc_stress.yaml ops\(singletrans=1\) -node 10.0.0.4

cassandra-stress user profile=./txn_by_cc_stress.yaml ops\(dailytrans=1\) -node 10.0.0.4

```

##Visual Dashboard

![alt dashboard](https://github.com/carolinerg1/FraudPrevention/blob/master/banana/TransactionDashboard.png)

[Dashboard](http://104.42.109.110:8983/banana/#/dashboard) was done using Banana. Follow this [guide](https://medium.com/@carolinerg/visualizing-cassandra-solr-data-with-banana-b54bf9dd24c#.nqzr0may3) to set it up.

The default dashboard is available in this repo under [Banana](https://github.com/carolinerg1/FraudPrevention/tree/master/banana). You will need to replace default.json under "/usr/share/dse/banana/src/app/dashboards"


##Code Sample

A full code example with inserts and queries can be found here - https://github.com/carolinerg1/FraudPrevention

Please follow the instructions to download and populate your cluster with example data. This example also shows how to provide access to the data through a JSON rest web service.


