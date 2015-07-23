## Wild card query

Wild card query is kind of query where search string is in form of e.g. `*ild*` or `?ild*`. Lucene document tells us quoted

    this query can be slow, as it needs to iterate over many terms. In order to prevent extremely slow
    WildcardQueries, a Wildcard term should not start with the wildcard

As part of my study how to fix a bug in filename search in Confluence, I take time to look at Lucene source code to understand how wild card query is evaluated from performance and resource consumption perspective.

**Query execution**

A complex query is rewrite i.e. decomposed into a tree of sub queries, each is evaluated lazily bottom up, their future results are combined in form of document iterator satisfied the given query. 

Elementary query at leaf of a query tree uses inverted index to retrieve set of documents containing specified term. This literally means for an elemetary query to work a term is needed.

In a traditional query, a complete terms are presented itself in the query expression. In other type of query like wild card, fuzzy, regex, Lucene has to figure out set of terms from the query expression. It does so by iterating over a dictionary of all terms to figure out matched terms.

**Wild card query execution and performance impact**

To evaluate wild card query, Lucene first iterates over a dictionary of all terms of the field specified in the query, filter out all terms that aren't matched the wild cards. For each matched term, it retrieves set of relevant documents containing the matched term and merges them together.

**Memory consumption**

The main execution loop is over each matched term, so the memory consumption is roughly the same as of any other queries. To be precise, what we need is a bit array capable of holding all documents of an index segment.

**number of IO**

The need to read entired dictionary of all terms may raise a flag but it shouldn't be as Lucene uses burst trie data structure to compress data so the dictionary file is tiny compare to other files in the index.

**CPU cycle**






