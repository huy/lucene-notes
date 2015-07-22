## Wild card query demystify

Lucene document tells us that quoted 

    this query can be slow, as it needs to iterate over many terms. In order to prevent extremely slow
    WildcardQueries, a Wildcard term should not start with the wildcard

As part of my study how to fix a bug in filename search in Confluence, I take time to look at Lucene source code to understand how wild card query is evaluated from performance and resource consumption perspective.

**Lucene Query Evaluation from 1000 miles**

Lucene provides `org.apache.lucene.search.Filter` class in low level API for searching. Query is just a wrapper around `Filter`. As such we will focus `Filter`. One method of this class is

    abstract DocIdSet getDocIdSet(LeafReaderContext context, Bits acceptDocs)

It takes 2 arguments 

* `LeafReaderContext` is used to read data from index segment file aka sub index.
* `Bits` is a bitmap representing live documents i.e. all except deleted documents, note that is null instead of all 1 bit map in case we haven't deleted any documents from the index segment
