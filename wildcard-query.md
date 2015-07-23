## Wild card query

Wild card query is kind of query where search string is in form of e.g. `*ild*` or `?ild*`. Lucene document tells us quoted

    this query can be slow, as it needs to iterate over many terms. In order to prevent extremely slow
    WildcardQueries, a Wildcard term should not start with the wildcard

As part of my study how to fix a bug in filename search in Confluence, I take time to look at Lucene source code to understand how wild card query is evaluated from performance and resource consumption perspective.

**Query execution**

A complex query is rewrite i.e. decomposed into a tree of sub queries, each is evaluated lazily bottom up, their future results are combined in form of document iterator satisfied the given query. 

Query execution is implemented in these tightly coupled classes `Query`, `Filter`, `Weight`, `Scorer`, so we may see excution logic juggling between them. `Filter` encapsulates `Query`, which creates `Weight`, which in turn create `Scorer`. `Scorer` is an iterator over documents satisfied a given query. 

Elementary query at leafs of a query tree use inverted index which basically maps a term to a set of documents containing specified term. This literally means for an elemetary query to work a term is needed.

**Wild card query execution and performance impact**

To evaluate wild card query, Lucene (using segment reader from `LeafReaderContext`) first iterates over a dictionary of all terms of the field specified in the query, filter out all terms that aren't matched the wild cards. For each matched term, it runs an elementary seach to retrieve set of relevant documents and merge them together.

**Memory consumption**

The main execution loop is over each matched term, so the memory consumption is roughly the same as of any other queries. To be precise, we need bitmap capable of holding all documents of the index segment.

**number of IO**

The need to read entired dictionary of all terms may raise a flag but it shouldn't be as Lucene uses burst trie data structure to compress data so the dictionary file is tiny compare to other files in the index.

**CPU cycle**






