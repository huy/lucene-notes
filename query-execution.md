## How query is executed

**Rewrite**

A complex query is rewrite i.e. decomposed with respect to query combinator (see [query combinator](query-filter.md)) into a tree of sub queries.

**Filter**

Each elementary query is evaluated (lazily if it is possible) bottom up, their results are combined in form of document iterator `DocEnum` satisfying the given query.

Internally main classes involved in filtering are `Query`, `Weight`, `Scorer`. `Query` creates `Weight`, which is represents runtime aspect of `Query`, `Weight` creates `Scorer`, which is iterator of document matching the query.

Elementary query at leaf of a query tree uses inverted index to retrieve set of documents containing specified term. This literally means for an elemetary query to work a term is needed.

In a traditional query, a complete terms are presented itself in the query expression. In other type of query like wild card, fuzzy, regex, Lucene has to figure out set of terms from the query expression. It does so by iterating over a dictionary of all terms and filter out un matched terms.

**Collector**

`Collector` is used to collect/transform hits into desire output, scoring or sorting is done in this step. The most common collector is `TopDocsCollector`, which collects top N documents with highest relevant score. `TopDocsCollector` uses modified version of priority queue with limited size to maintain top N documents by score. E.g.

    private static class InOrderTopScoreDocCollector extends TopScoreDocCollector {
       ...
       public void collect(int doc) throws IOException {
            float score = this.scorer.score();

            assert score != -1.0F / 0.0;

            assert !Float.isNaN(score);

            ++this.totalHits;
            if(score > this.pqTop.score) {
                this.pqTop.doc = doc + this.docBase;
                this.pqTop.score = score;
                this.pqTop = (ScoreDoc)this.pq.updateTop();
            }
        }

## Wild card and Fuzzy query and performance impact

To evaluate this kind of queries, Lucene first iterates over a dictionary of all terms of the field specified in the query, filter out all terms that don't match the wild cards. For each matching term, it retrieves set of relevant documents containing the term and merges them together.

**Memory consumption**

The main execution loop is over each matched term, so the memory consumption is roughly the same as of any other queries. To be precise, what we need is a bit array capable of holding ID of all documents of an index segment.

**CPU cycles**

The evaluation is done in this part ofclass `org.apache.lucene.search.MultiTermQueryWrapperFilter`

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

The operation will become expensive if the wild card expands to too many terms and each term corresponds to just few documents in the inverted index.
