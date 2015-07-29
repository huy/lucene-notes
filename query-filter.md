## Query, Filter

**query vs filter**

Query is input for search, filter can be used in addition to limit scope of search. The fundamental difference is 
filter is not involved in scoring hits.

**combinator**

There are elementary queries (e.g. TermQuery), which can be combined using combinators (e.g. BooleanQuery). 
The result can be also combined again creating complex nested query tree.

The universal combinator is BooleanQuery, which can be used across all types of queries. 

There is however a specific combinator that is applied only for a specific type of queries e.g. SpanNearQuery can only 
combine two or more SpanQuery.
