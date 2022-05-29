# thinking-in-tinybird
Data project to illustrate how to iterate when building new use cases in tinybird.

`main` [branch](https://github.com/tinybirdco/thinking-in-tinybird): explanation of the repo and the process we will be following.  
`refactor-datasources` [branch](https://github.com/tinybirdco/thinking-in-tinybird/tree/first-draft): here's the state of the data project in the first iteration, still with no optimizations.

## Build a quick working prototype

At the beginning, focus just on the use case. Add some samples or reuse existing data sources, think of the result you want to produce, and build your pipes accordingly.

The key part of this step is to understand the desired output, the dynamic parameters your endpoints will support and the kind of transformations you will be doing in your pipes.

As Tinybird is that fast, you get instant feedback and develop use cases really fast. Improving performance will ve covered on the `refactor-datasources` [branch](https://github.com/tinybirdco/thinking-in-tinybird/tree/first-draft).
