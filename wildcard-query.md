## Wild card query

Wild card query is kind of query where search string is in form of e.g. `*ild*` or `?ild*`. Lucene document tells us quoted

    this query can be slow, as it needs to iterate over many terms. In order to prevent extremely slow
    WildcardQueries, a Wildcard term should not start with the wildcard

As part of my study how to fix a bug in filename search in Confluence, I take time to look at Lucene source code to understand how wild card query is evaluated from performance and resource consumption perspective.

### How query is executed

A complex query is rewrite i.e. decomposed into a tree of sub queries, each is evaluated lazily bottom up, their lazy results are combined in form of document iterator `DocEnum` satisfying the given query. 

Elementary query at leaf of a query tree uses inverted index to retrieve set of documents containing specified term. This literally means for an elemetary query to work a term is needed.

In a traditional query, a complete terms are presented itself in the query expression. In other type of query like wild card, fuzzy, regex, Lucene has to figure out set of terms from the query expression. It does so by iterating over a dictionary of all terms and filter out un matched terms.

### Wild card query and performance impact

To evaluate wild card query, Lucene first iterates over a dictionary of all terms of the field specified in the query, filter out all terms that don't match the wild cards. For each matching term, it retrieves set of relevant documents containing the term and merges them together.

**Memory consumption**

The main execution loop is over each matched term, so the memory consumption is roughly the same as of any other queries. To be precise, what we need is a bit array capable of holding ID of all documents of an index segment.

**IO and CPU**

The need to read entired dictionary of all terms may raise a flag but we shouldn't worry as Lucene uses burst trie data structure to compress data so for any non random set of terms, a dictionary file is tiny compare to other files in the index.

The worry part of wild card query evaluation is this part of code in class `org.apache.lucene.search.MultiTermQueryWrapperFilter`

    TermsEnum termsEnum = this.query.getTermsEnum(terms); // return an iterator over all matched terms
    ...
    
    FixedBitSet bitSet = new FixedBitSet(context.reader().maxDoc());
    DocsEnum docsEnum = null;
    do {
        docsEnum = termsEnum.docs(acceptDocs, docsEnum, 0); // return an iterator over documents containing current term

        int docid;
        while((docid = docsEnum.nextDoc()) != 2147483647) {
            bitSet.set(docid);
        }
    } while(termsEnum.next() != null);
    return bitSet;

The operation will become expensive if the wild card expands to too many terms and each term corresponds to just few documents in the inverted index. It is quite possible e.g. when term is filename.

Notes

* This source code is taken from Lucene 4.4.0
* In current Lucene trunk, the behavoir is more less the same if the wild card expands to number of terms exceeding certain threshold (currently set to 16).

**What we do to mitigate it**

If a wild card expands to too many terms, it also mean that the wild card is not restrictive enough and the search will likely return many hits. 

Assuming that

* we know maximum number of hits
* we relax the requirement about returning top hit documents by score
* we don't care about total hits

We can retrieve just enough hit documents so many wild card queries will not bring down the server.

