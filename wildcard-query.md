## Wild card query

Wild card query is kind of query where search string is in form of e.g. `*ild*` or `?ild*`. Lucene document tells us quoted

    this query can be slow, as it needs to iterate over many terms. In order to prevent extremely slow
    WildcardQueries, a Wildcard term should not start with the wildcard

As part of my study how to fix a bug in filename search in Confluence, I take time to look at Lucene source code to understand how wild card query is evaluated from performance and resource consumption perspective.

**Query execution**

Query execution is implemented in these tightly coupled classes `Query`, `Filter`, `Weight`, `Scorer`, so we may see excution logic juggling between them. `Query` creates `Weight`, which in turn create `Scorer`. `Score` is an iterator over documents satisfied a given query. 

Lucene provides `org.apache.lucene.search.Filter` abstract class in low level search API. Query is wrapped inside  `Filter`. One method of `Filter` class is

    abstract DocIdSet getDocIdSet(LeafReaderContext context, Bits acceptDocs)

It takes 2 arguments 

1. `LeafReaderContext` is used to read data from index segment file (aka sub index).
2. `Bits` is a bitmap representing live documents i.e. all except deleted documents, note that `null` instead of all 1 bit map is used to represent a situation when we haven't deleted any documents from the index segment. 

It returns a set of documents that satisfy the wrapped query. Important things about Lucene low level search operation are

* elementary search operation use inverted index which basically maps a term (aka word) to a set of documents containing specified term. This literally means for an elemetary seach a term is needed.
* result of elementary search is a bit map representing a set of documents
* complex query is decomposed into serie of elemetary queries, each is evaluated, then their result are union and/or intersect to form a final list of hit documents

**Combine result from individual segment search**


**Wild card query execution and performance impact**

To evaluate wild card query, Lucene (using segment reader from `LeafReaderContext`) first iterates over a dictionary of all terms of the field specified in the query, filter out all terms that aren't matched the wild cards. For each matched term, it runs an elementary seach to retrieve set of relevant documents and merge them together.

**Memory consumption**

The main execution loop is over each matched term, so the memory consumption is roughly the same as of any other queries. To be precise, we need bitmap capable of holding all documents of the index segment.

**number of IO**

The need to read entired dictionary of all terms may raise a flag but it shouldn't be as Lucene uses burst trie data structure to compress data so the dictionary file is tiny compare to other files in the index.

**CPU cycle**






