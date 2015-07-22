## Wild card query demystify

Wild card query is kind of query where search string is in form of `*ild*` or `?ild*`. Lucene document tells us that quoted

    this query can be slow, as it needs to iterate over many terms. In order to prevent extremely slow
    WildcardQueries, a Wildcard term should not start with the wildcard

As part of my study how to fix a bug in filename search in Confluence, I take time to look at Lucene source code to understand how wild card query is evaluated from performance and resource consumption perspective.

**Lucene Query Evaluation from 1000 miles**

Lucene provides `org.apache.lucene.search.Filter` abstract class in low level search API. Query is wrapped inside  `Filter`. One method of `Filter` class is

    abstract DocIdSet getDocIdSet(LeafReaderContext context, Bits acceptDocs)

It takes 2 arguments 

* `LeafReaderContext` is used to read data from index segment file aka sub index.
* `Bits` is a bitmap representing live documents i.e. all except deleted documents, note that `null` instead of all 1 bit map is used to represent a situation when we haven't deleted any documents from the index segment. 

It returns a set of documents that satisfy the wrapped query. 
