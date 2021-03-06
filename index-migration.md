## Lucene index migration

This article tries to shed some light on Lucene index migration, which is neccesary part of library upgrade process.

### Understanding index

Lucene persists its data in an index. Assuming the media is file system then index is a directory containing a set of data files. 

**Codec**

There are different data types, the way how each data type is serialized (aka data format) is dictated by codec. As Lucene code evolves so the codec. Codec version is not neccesary the same as Lucene release version. 
E.g. the default codec version of Lucene 4.4.x is 4.2 while the default codec of Lucene 4.9.x is 4.9. We can find codec version by looking at class `org.apache.lucene.codecs.Codec`  e.g.

    ...
    public abstract class Codec implements NamedSPI {
        ...
        private static Codec defaultCodec = forName("Lucene49");

**Segment**

Lucene index has composite structure, which means an index consists of several sub indexes aka segment. Listing an index folder, we may see something like this

    -rw-r--r--  1 huyle  staff  268 19 Jul 07:55 _0.cfe
    -rw-r--r--  1 huyle  staff  647 19 Jul 07:55 _0.cfs
    -rw-r--r--  1 huyle  staff  291 19 Jul 07:55 _0.si
    -rw-r--r--  1 huyle  staff  284 19 Jul 08:46 _1.cfe
    -rw-r--r--  1 huyle  staff  823 19 Jul 08:46 _1.cfs
    -rw-r--r--  1 huyle  staff  303 19 Jul 08:46 _1.si
    -rw-r--r--  1 huyle  staff   36 19 Jul 09:15 segments.gen
    -rw-r--r--  1 huyle  staff  341 19 Jul 09:15 segments_1
    -rw-r--r--  1 huyle  staff    0 19 Jul 08:06 write.lock

The index contains two segments `0` and `1`. Each segment is represented by 3 files with ext `.si`, `.cfs`, `.cfe`. The `.si` file is metadata of the segment while `.cfe` and `.cfs` contains entries and data in compound form, which means different data types are packed into the same file instead of separated file per data type. The codec used to deserialized data file back to memory is written in `.si` file. e.g.

    $ xxd _0.si
    0000000: 3fd7 6c17 134c 7563 656e 6534 3053 6567  ?.l..Lucene40Seg
    0000010: 6d65 6e74 496e 666f 0000 0000 0334 2e34  mentInfo.....4.4
    ...

**Active segment**

Lucene index follows single writer multiple reader paradigm. The active segment is segment currently used by the writer. Everytime the writer issues a commit, active segment is writen to disk and a new segment is created using the most recent codec amd it becomes the new active segment.

When migrating to a new version of Lucene, we may end up with a index having some segments encoded by old codec and others by new codec.

The reason behind such design is to support incremental indexing with concurrent access. Its clever design means when searching is still in progress we can safely add/remove document without expensive synchronization because the search always sees the index at a point time it was opened.

**Merging segments**

Having too many segments may reach OS limit as well as consume at lot of OS file handles and hurt performance. Lucene at its own discretion (depending on specified merge policy) merge many non active segments into single one reducing number of files in the index directory.

It is done by creating a new segment and copy data from old segments into it. A new segment is created using the most recent codec, which means segment migration happens gradually without downtime and/or performance impact. 

**Index upgrade**

As described above, index uprade can happens gradually during merging (aka silent upgrade). Lucene however allows explicit once time index upgrade through `org.apache.lucene.index.IndexUprader` class. Behind the scene, it merges all old format segments into new created segment.

### Things to consider

Lucene makes a best effort to easy the uprade process involving huge index. It includes handful of old codec(s) to make the library work with segments on many old codec. Never the less, there are still things to be verified and considered when doing upgrade.

**Available codec(s)**

Check the available codecs against one specified in `.si` files. Each codec implemented in form of a package e.g. the present of `org.apache.lucene.codecs.lucene42` mean that the library can read segment in 4.2 format.

**Performance impact** 

Decide whether to perform once time upgrade or let Lucene to do silent upgrade. The questions to ask is if there is any peformance penalty of searching with old format segments. 

We hope that deserializing old format by the new code will not be worse than doing the same by old code unless there is significal mismatch between in memory data structure(s).


