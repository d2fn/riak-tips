Riak Tips
=========

A collection of Riak tips follows. This is based on my own experience using 1.2.1 with leveldb and bitcask backends. Yours, obviously, may differ.

#1
A riak cluster is not typically IO or CPU bound. Rather, coordination overhead between nodes is typically a pinch point. Bear in mind that, in almost every case, each read and write requires some coordination among replicas. You can use this to your advantage by treating Riak almost as a block device. This will have the effect of amortizing coordination overhead over a smaller keyspace. 

_Corollary: reduce keyspace size by grouping data into blocks._
