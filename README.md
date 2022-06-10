# thinking-in-tinybird
Data project to illustrate how to iterate when building new use cases in tinybird.

## Work backwards

Our way of approaching new use cases in tinybird is as follows: 

Think of the output and adapt the pipes and schema accordingly. 

Rather than applying directly the data model and building the logic based on that,
 ```

*data |-------|      |-------|   api
  ---->| model |----->| logic |---->
       |-------|      |-------|
 ```
 redesign your logic and model based on expected output.
 ```
data  |-------|      |-------|   api*
  ---->| model |<-----| logic |<----
       |-------|      |-------|
 ```



In practice, this means:

## Work with some existing data sources or upload samples of new ones

If the use case you’re trying to build is a completely new one, you may need to create new data sources. The easiest way to do so is usually via the UI appending a sample

## Define the expected output

Once you have a clear understanding of what you need to return and what parameters you're going to define you're ready to move to the next step.

## Based on your datasources, start building your pipes. 

Don't think of MVs yet, just your datasources and the rules you learned [here](https://www.tinybird.co/guide/best-practices-faster-sql-queries)

> you can see a result of the first three steps in the `fist-draft` [branch](https://github.com/tinybirdco/thinking-in-tinybird/tree/first-draft) of this repo.

## Refactor your data sources

Based on the queries you will do and focused on reducing the amount of scanned data, edit your data source column types and indexes.

> you can check more details about all things to take into account in the `refactor-datasources` [branch](https://github.com/tinybirdco/thinking-in-tinybird/tree/refactor-datasources) of this repo.

[Sneak peek](https://ui.us-east.tinybird.co/snapshot/b59d6076f9a747f5898a7916c3ce1767) of the improvements

## Extra tips

The join of our pipe is OK because the companies data source only has 5 rows, but for JOINs with larger data sources, using subqueries reduce a lot the memory footprint. See this [example](https://ui.tinybird.co/snapshot/10e0ab7dcb8e4925bc63a3d46a0eebc2) taken from the Typeform [blogpost](https://www.tinybird.co/blog-posts/typeform-utm-realtime-analytics).

Sometimes your pipes constantly run an aggregtion that scans many rows, or you need several steps to transform the data model into something useful. If that's the case, check [materialized views](https://www.tinybird.co/guide/materialized-views), a feature that let's you transform data at ingestion time and incrementally refresh the view.
