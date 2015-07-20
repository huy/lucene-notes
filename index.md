## Lucene index

Lucene persists its data in an index. If the media is file system then index is a directory containing a set of data files.

**Codec**

There are different data types, the way how each data type is serialized (aka data format) is dictated by codec. 

As Lucene code evolves so the codec. Codec version is not neccesary the same as Lucene release version. 
E.g. the default codec version of Lucene 4.4.x is 4.2 while the default codec of Lucene 4.9.x is 4.9. 

We can find codec version by looking at class `org.apache.lucene.codecs.Codec`  e.g.

    ...
    public abstract class Codec implements NamedSPI {
        ...
        private static Codec defaultCodec = forName("Lucene49");




