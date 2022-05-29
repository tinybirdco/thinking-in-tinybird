# thinking-in-tinybird

Data project to illustrate how to iterate when building new use cases in tinybird.

first-draft branch: here's the state of the data project in the first iteration, still with no optimizations.

## Refactoring data sources

We are assuming you have the CLI installed and active. If that's not the case check our docs.

Authenticate and download the data project:

```bash
tb auth -i
```

choose your region and paste your token. Then create the folder structure and downlad the project from your workspace.

```bash
tb init
tb pull --auto
```

## Data sources

### Data types

Using the right type for each column is key for improving performance. Things as not using 16 bytes for a boolean value, avoiding nulls, or using LowCardinality capabilities can have a huge impact.

- companies:

It is very unlikely to have negative numbers as ids, so just by changing `Int16` to `UInt16` we have gained some extra room for our number of companies without sacrificing performance. Same for size, better `UInt32` than `Int32`.

We are not planning on adding millions of plans, so replacing String by LowCardinality(String) will offer us more speed when our companies table grow.

- events:

Same reasoning for company_id, so `UInt16`.
LowCardinality(String) for event, OS, and browser.

note: we didn't have any nullable column here, but they are also good candidates to avoid, for example setting '' or 0 as the alternative for null.

### Sorting keys

Sorting keys have two functions: they define how data is stored in the data source —remember rule 3 of the [best practices guide](https://www.tinybird.co/guide/best-practices-faster-sql-queries)— and they also add markers to find the data faster.

- companies:

Probably just with company_id would be enough.  
If you are planning to do queries like: get latest events for all the companies of a certain plan or size, then note that you shoud use the types in increasing order of cardinality, e.g., `ENGINE_SORTING_KEY "plan, size, company_id"`.

Let's leave just company_id then.


- events:

We will be filtering by company id, so it muust be our first index. It is also worth noting company_id and datetime order, since it is better to have the columns you are going to filter by exact matches first and the ones you will filter by range (datetime between start and end) later.


### Partition

Partitions are the way data is stored in the database file system.

- companies:

By default, tinybird created this partition key: `ENGINE_PARTITION_KEY "substring(toString(company_id), 1, 1)"` but looking at the [docs](https://docs.tinybird.co/tips/data-source-tips.html#choosing-the-engine-partition-key), and knowing it is a dimensions table, let's remove it.

- events: 

here we do have the time column mentioned in the docs, we could go for the default `ENGINE_PARTITION_KEY "toYear(datetime)"` or `ENGINE_PARTITION_KEY "toYYYYMM(datetime)"`.
As we expect lots of data in our datasource, let's go for a monthly based one. See note below for some rules.

Note: some things to take into account when selecting a partition:
- sometimes it's better not to have one. When in doubt, leave it blank, unless your datasource is expected to be >300GB.
- pay special attention if you will create MVs from the DS and are planning to de replaces. More info [here](https://www.tinybird.co/guide/replacing-and-deleting-data#replace-data-selectively).
- when ingesting new data you should ideally write to just one partition, and never more than 10.

### Refactored schemas

This is our result then:

- companies_refactor.datasource

```diff
 DESCRIPTION >
     __dimensions__ datasource, typically smaller and used to enrich events from facts datsource
 
 SCHEMA >
     `company_id` Int16,
     `name` String,
     `name` LowCardinality(String),
     `size` Int32,
-    `plan` String
+    `plan` LowCardinality(String)

 ENGINE "MergeTree"
-ENGINE_PARTITION_KEY "substring(toString(company_id), 1, 1)"
-ENGINE_SORTING_KEY "company_id, name, size, plan"
+ENGINE_SORTING_KEY "company_id"
```

- events.datasource

```diff
 DESCRIPTION >
     __facts__ datasource, where we will store all user generated events

 SCHEMA >
-    `company_id` Int16 `json:$.company_id`,
+    `company_id` UInt16 `json:$.company_id`,
     `datetime` DateTime64(3) `json:$.datetime`,
-    `device_OS` String `json:$.device.OS`,
+    `device_OS` LowCardinality(String) `json:$.device.OS`,
-    `device_browser` String `json:$.device.browser`,
+    `device_browser` LowCardinality(String) `json:$.device.browser`,
-    `event` String `json:$.event`,
+    `event` LowCardinality(String) `json:$.event`,
     `payload_author` String `json:$.payload.author`,
     `payload_entity_id` String `json:$.payload.entity_id`

 ENGINE "MergeTree"
-ENGINE_PARTITION_KEY "toYear(datetime)"
+ENGINE_PARTITION_KEY "toYYYYMM(datetime)"
-ENGINE_SORTING_KEY "datetime, event, payload_author, payload_entity_id"
+ENGINE_SORTING_KEY "company_id, datetime"
```

## Push the changes and fill the new data sources

```bash
tb push datasources/*_refactor
echo "NODE mat \nSQL >\n\n\tSELECT toUInt16(company_id) company_id, datetime, toLowCardinality(device_OS) device_OS, toLowCardinality(device_browser) device_browser, toLowCardinality(event) event, payload_author, payload_entity_id  FROM events\n\nTYPE materialized\nDATASOURCE events_refactor" > fill_events.pipe
tb push fill_events.pipe --populate --wait 
tb pipe rm fill_events --yes
echo "NODE mat \nSQL >\n\n\tSELECT toUInt16(company_id) company_id, name, size, toLowCardinality(plan) plan FROM companies\n\nTYPE materialized\nDATASOURCE companies_refactor" > fill_companies.pipe
tb push fill_companies.pipe --populate --wait 
tb pipe rm fill_companies --yes
```

>note this is not the main use of MVs.

## Pipes

Let's push a new endpoint to test if our changes had any impact. Just changing _events_ by _events_refactor_ and companies by _companies_refactor_, and call it to check differences in processed data:

```bash
tb push pipes/events_per_hour_refactor.pipe
```
After some calls and looking at our dashboard this is the difference when calling the original and refactor datasources:
![comparing_usage](https://user-images.githubusercontent.com/29075486/170879312-9fb1da0b-d14b-4ef5-841b-c30b076251fa.png)


Seems like we are happy with the difference, so let's edit events_per_hour to query our new datasources:

```diff
 NODE endpoint
 SQL >
 
     %
     SELECT 
       datetime,
       name,
       payload_author,
       event,
       payload_entity_id
-     FROM events
+     FROM events_refactor
-     JOIN companies
+     JOIN companies_refactor
     USING company_id
     WHERE company_id = {{Int16(company, 1)}}
     AND datetime 
       BETWEEN toDateTime64({{String(start_datetime, '2022-05-23 00:00:00', description="initial datetime", required=True)}},3) 
       AND toDateTime64({{String(end_datetime, '2022-05-25 23:59:59', description="final datetime", required=True)}},3) 
     ORDER BY datetime DESC
```

And use the CLI tests to double check results. If you `tb push --force` an endpoint, a battery of regression tests will run. Here is a sample of the output:

```bash
tb push pipes/events_per_hour.pipe --force
** Processing pipes/events_per_hour.pipe
** Building dependencies
** Running events_per_hour 
current https://api.us-east.tinybird.co/v0/pipes/events_per_hour.json?company=3&start_datetime=2022-05-23+00%3A00%3A00&end_datetime=2022-05-25+23%3A59%3A59&pipe_checker=true
    new https://api.us-east.tinybird.co/v0/pipes/events_per_hour__checker.json?company=3&start_datetime=2022-05-23+00%3A00%3A00&end_datetime=2022-05-25+23%3A59%3A59&pipe_checker=true ... ok
current https://api.us-east.tinybird.co/v0/pipes/events_per_hour.json?company=3&start_datetime=2022-05-23+00%3A00%3A00&end_datetime=2022-05-25+23%3A59%3A59&pipe_checker=true
    new https://api.us-east.tinybird.co/v0/pipes/events_per_hour__checker.json?company=3&start_datetime=2022-05-23+00%3A00%3A00&end_datetime=2022-05-25+23%3A59%3A59&pipe_checker=true ... ok
current https://api.us-east.tinybird.co/v0/pipes/events_per_hour.json?company=3&start_datetime=2022-05-23+00%3A00%3A00&end_datetime=2022-05-25+23%3A59%3A59&pipe_checker=true
    new https://api.us-east.tinybird.co/v0/pipes/events_per_hour__checker.json?company=3&start_datetime=2022-05-23+00%3A00%3A00&end_datetime=2022-05-25+23%3A59%3A59&pipe_checker=true ... ok
current https://api.us-east.tinybird.co/v0/pipes/events_per_hour.json?company=3&start_datetime=2022-05-23+00%3A00%3A00&end_datetime=2022-05-25+23%3A59%3A59&pipe_checker=true
    new https://api.us-east.tinybird.co/v0/pipes/events_per_hour__checker.json?company=3&start_datetime=2022-05-23+00%3A00%3A00&end_datetime=2022-05-25+23%3A59%3A59&pipe_checker=true ... ok
current https://api.us-east.tinybird.co/v0/pipes/events_per_hour.json?company=3&start_datetime=2022-05-23+00%3A00%3A00&end_datetime=2022-05-25+23%3A59%3A59&pipe_checker=true
    new https://api.us-east.tinybird.co/v0/pipes/events_per_hour__checker.json?company=3&start_datetime=2022-05-23+00%3A00%3A00&end_datetime=2022-05-25+23%3A59%3A59&pipe_checker=true ... ok
current https://api.us-east.tinybird.co/v0/pipes/events_per_hour.json?company=3&start_datetime=2022-05-23+00%3A00%3A00&end_datetime=2022-05-25+23%3A59%3A59&pipe_checker=true
    new https://api.us-east.tinybird.co/v0/pipes/events_per_hour__checker.json?company=3&start_datetime=2022-05-23+00%3A00%3A00&end_datetime=2022-05-25+23%3A59%3A59&pipe_checker=true ... ok

==== Test Metrics ====

------------------------------------------------------------------------
| Test Run | Test Passed | Test Failed | % Test Passed | % Test Failed |
------------------------------------------------------------------------
|        6 |           6 |           0 |         100.0 |           0.0 |
------------------------------------------------------------------------

==== Response Time Metrics ====

-------------------------------------------
| Timing Metric (s) |  Current |      New |
-------------------------------------------
| min               | 0.859984 | 0.676805 |
| max               | 1.124045 | 0.811588 |
| mean              | 1.032945 | 0.723708 |
| median            | 1.065268 | 0.712358 |
| p90               | 1.124045 | 0.811588 |
-------------------------------------------
** Token testing_token found, adding permissions
** => Test endpoint with:
** $ curl https://api.us-east.tinybird.co/v0/pipes/events_per_hour.json?token=$TESTING_TOKEN
** 'events_per_hour' created
** Not pushing fixtures
```

## Final note:

Trust but verify. These are some improvement proposals that we believe will make this use case faster, but when building your own, do test several approaches, combinations os keys, queries... and, as always, when in doubt, just ask us!
