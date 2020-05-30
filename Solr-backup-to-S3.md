This article discusses how to backup SolrCloud collections to S3 storage. It was inspired by a Solr [jira ticket thread](https://issues.apache.org/jira/browse/SOLR-9952), also refers to [official Solr doc](https://lucene.apache.org/solr/guide/8_5/collection-management.html#backup).

Since version 7 Solr already supports to backup collection to HDFS location. However official doc does not explicitly mention how to backup to S3 storage, neither can you use S3 as a backup repository using official Solr release out-of-the-box. Luckily, because HDFS has integrated S3 file systems, it doesn't require further dev work to make S3 applicable. We just need to tweak a little bit to add missing dependencies. 

## Setup

Take the most recent Solr release, version 8.5.0 (up to May, 2020), as example, here are the steps to create a S3 enabled Solr release package:

1. Download and untarnished `solr-8.5.0.tgz`

2. Add the following snippet to `server/solr/solr.xml`:

   ```xml
   <backup>
       <repository name="hdfs" class="org.apache.solr.core.backup.repository.HdfsBackupRepository" default="false">
         <str name="location">${solr.hdfs.default.backup.path}</str>
         <str name="solr.hdfs.home">${solr.hdfs.home:}</str>
         <str name="solr.hdfs.confdir">${solr.hdfs.confdir:}</str>
         <int name="solr.hdfs.buffer.size">${solr.hdfs.buffer.size:262144}</int>
      </repository>
   </backup> 
   ```

3. Find and put the following dependency jars to `solr-webapp/webapp/WEB-INF/lib/`.

   - *hadoop-aws-3.2.0.jar*

   - *aws-java-sdk-bundle-1.11.375.jar*

   Clearly *aws-java-sdk-bundle-1.11.375.jar* is not a miminum dependency of S3 file system. You are welcome to refine it, if have time.

4. Turn the directory back to a gzipped tar ball `solr-8.5.0.tgz`.

5. Install Solr using this tar ball.

Alternatively you can install Solr in normal way and add `solr.xml` and the two jar files to specific location on each Solr node seperately.

To configure Hadoop HDFS, put this `core-site.xml` file to any location on disk, preferably `<solr_dir>/data`. This location will be used in calling Solr backup API .

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<configuration>
    <property><name>file.blocksize</name><value>67108864</value></property>
    <property><name>file.stream-buffer-size</name><value>4096</value></property>
    <property><name>fs.AbstractFileSystem.file.impl</name><value>org.apache.hadoop.fs.local.LocalFs</value></property>
    <property><name>fs.AbstractFileSystem.hdfs.impl</name><value>org.apache.hadoop.fs.Hdfs</value></property>
    <property><name>fs.AbstractFileSystem.s3a.impl</name><value>org.apache.hadoop.fs.s3a.S3A</value></property>
    <property><name>fs.automatic.close</name><value>true</value></property>
    <property><name>fs.client.resolve.remote.symlinks</name><value>true</value></property>
    <property><name>fs.defaultFS</name><value>s3a://${YOUR_S3_BUCKET}</value></property>
    <property><name>fs.df.interval</name><value>60000</value></property>
    <property><name>fs.file.impl</name><value>org.apache.hadoop.fs.LocalFileSystem</value></property>
    <property><name>fs.s3.block.size</name><value>67108864</value></property>
    <property><name>fs.s3.buffer.dir</name><value>${hadoop.tmp.dir}/s3</value></property>
    <property><name>fs.s3.maxRetries</name><value>4</value></property>
    <property><name>fs.s3.sleepTimeSeconds</name><value>10</value></property>
    <property><name>fs.s3a.access.key</name><value>${YOUR_S3_ACCESS_KEY}</value></property>
    <property><name>fs.s3a.attempts.maximum</name><value>20</value></property>
    <property><name>fs.s3a.block.size</name><value>32M</value></property>
    <property><name>fs.s3a.buffer.dir</name><value>${hadoop.tmp.dir}/s3a</value></property>
    <property><name>fs.s3a.connection.establish.timeout</name><value>5000</value></property>
    <property><name>fs.s3a.connection.maximum</name><value>15</value></property>
    <property><name>fs.s3a.connection.ssl.enabled</name><value>false</value></property>
    <property><name>fs.s3a.connection.timeout</name><value>200000</value></property>
    <property><name>fs.s3a.fast.upload.active.blocks</name><value>4</value></property>
    <property><name>fs.s3a.fast.upload.buffer</name><value>disk</value></property>
    <property><name>fs.s3a.fast.upload</name><value>true</value></property>
    <property><name>fs.s3a.impl</name><value>org.apache.hadoop.fs.s3a.S3AFileSystem</value></property>
    <property><name>fs.s3a.max.total.tasks</name><value>50</value></property>
    <property><name>fs.s3a.multiobjectdelete.enable</name><value>true</value></property>
    <property><name>fs.s3a.multipart.purge.age</name><value>86400</value></property>
    <property><name>fs.s3a.multipart.purge</name><value>false</value></property>
    <property><name>fs.s3a.multipart.size</name><value>100M</value></property>
    <property><name>fs.s3a.multipart.threshold</name><value>2147483647</value></property>
    <property><name>fs.s3a.paging.maximum</name><value>5000</value></property>
    <property><name>fs.s3a.path.style.access</name><value>false</value></property>
    <property><name>fs.s3a.readahead.range</name><value>64K</value></property>
    <property><name>fs.s3a.secret.key</name><value>${YOUR_S3_SECRET_KEY}</value></property>
    <property><name>fs.s3a.socket.recv.buffer</name><value>8192</value></property>
    <property><name>fs.s3a.socket.send.buffer</name><value>8192</value></property>
    <property><name>fs.s3a.threads.keepalivetime</name><value>60</value></property>
    <property><name>fs.s3a.threads.max</name><value>100</value></property>
    <property><name>fs.s3n.block.size</name><value>67108864</value></property>
    <property><name>fs.s3n.multipart.uploads.block.size</name><value>67108864</value></property>
    <property><name>fs.s3n.multipart.uploads.enabled</name><value>false</value></property>
    <property><name>ha.failover-controller.new-active.rpc-timeout.ms</name><value>60000</value></property>
    <property><name>hadoop.tmp.dir</name><value>/tmp/hadoop-hdfs</value></property>
    <property><name>io.file.buffer.size</name><value>131072</value></property>
    <property><name>io.native.lib.available</name><value>true</value></property>
</configuration>
```

Please materialize `YOUR_S3_BUCKET`,  `YOUR_S3_ACCESS_KEY` and `YOUR_S3_SECRET_KEY` with your own settings. 

Now start Solr with the following command line arguments

```
-Dsolr.hdfs.default.backup.path=<my_path> -Dsolr.hdfs.home=s3a://<my_s3_bucket>/ -Dsolr.hdfs.confdir=<solr_dir>/data
```

where `solr.hdfs.confdir` shall point to the local directory where you put the file `core-site.xml`.

Note that the file system URI scheme `s3a` is crucial here . Compared with its predecessor `s3n`,  `s3a` supports larger file (>5GB), has higher performance and uses multi-part upload to speed up data transportation.  Supposedly `s3a` is superior to `s3n` in any aspect but I haven't benchmarked their performance though.

## Backup

Before backup, make sure s3 directory `s3://<my_s3_bucket>/<my_path>` does exist, otherwise Solr will complain.

You can invoke collection backup API synchronously:

```
curl 'localhost:8983/solr/admin/collections?action=BACKUP&name=my_backup&collection=my_collection&location=/my_path&repository=hdfs'
```

or asynchronously:

```
curl 'localhost:8983/solr/admin/collections?action=BACKUP&name=my_backup&collection=my_collection&location=/my_path&repository=hdfs&async=my-request-id'
```

and backup status by:

```
curl 'localhost:8983/solr/admin/collections?action=REQUESTSTATUS&requestid=my-request-id'
```

Either way it will backup Solr collection data and metadata to `s3://my_s3_bucket/my_path/my_backup`. You can open the file `backup.properties` to read the summary of this backup.

## Restore

Invoke the restoration:

```
curl 'localhost:8983/solr/admin/collections?action=RESTORE&name=my_backup&location=/my_path&collection=my_another_collection'
```

Note that `my_another_collection` cannot be existing before restoring.

It will restore and duplicate the Solr collection to newly created `my_another_collection`. Happy searching!