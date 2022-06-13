# scribl
Exporting Splunk Data at Scale with Scribl.  This is a python script that can be run on each Splunk Indexer for the purpose of exporting historical bucket data (raw events + metadata) at scale by balancing the work across multiple CPUs then forwarding to Cribl.

# Background
Splunk to Cribl = scribl (#thanksKam)

Exporting large amounts of previously indexed data from Splunk is unrealistic via the Splunk-supported approaches detailed here:  https://docs.splunk.com/Documentation/Splunk/8.2.6/Search/Exportsearchresults.

The core Splunk binary in every install provides a switch (cmd exporttool) that allows you to export the data from the compressed buckets on the indexers back into their original raw events.  You can dump them to very large local csv files or stream them to stdout so a script can redirect over the network to a receiver such as Cribl Stream.  Very few people know about this switch.  

Assuming that Splunk is installed in /opt/splunk, the below commands can be applied to a particular bucket in an index called “bots” to export it. 

Exporting to stdout:
```
/opt/splunk/bin/splunk cmd exporttool /opt/splunk/var/lib/splunk/bots/db/db_1564739504_1564732800_2394 /dev/stdout -csv
```

Exporting to a local csv file:
```
/opt/splunk/bin/splunk cmd exporttool /opt/splunk/var/lib/splunk/bots/db/db_1564739504_1564732800_2394 /exports/bots/db_1564739504_1564732800_2394.csv -csv
```

There will be many buckets so some poor soul will need to build a script to export all or some of the buckets and some sort of parallelization should be used to speed the process up.  The exported data will be very large (uncompressed, 3-20x) compared to the size of the individual buckets that make up the index!

# Requirements

Splunk stores its collected data on the indexers within the “Indexing Tier” as detailed below.  The data is compressed and stored in a collection of time series buckets that reside on each indexer.  Each bucket contains a rawdata journal, along with associated tsidx, and metadata files. The search heads access these buckets and it’s very rare for someone to access them directly from the indexer CLI unless there is a need to export data to retrieve the original raw events.  We will use the indexer CLI to export the original raw events (per bucket and in parallel) as well as a few other pieces of important metadata as detailed below. 

