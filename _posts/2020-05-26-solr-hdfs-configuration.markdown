---
layout:     post
title:      "Configure Solr with HDFS"
subtitle:   "Hadoop"
date:       2020-05-26 12:00:00
author:     "zhaoyim"
header-img: ""
catalog: true
tags:
    - Hadoop
---

1.install solr

```
$ tar -zxvf solr-8.5.1.tgz
$ mv solr-8.5.1 /opt
```
2.create core folder
```

$ mkdir /opt/solr-8.5.1/server/solr/hdfs_core
```

3.copy the default config files to hdfs core

```
$ cp -r /opt/solr-8.5.1/server/solr/configsets/_default/conf/ /opt/solr-8.5.1/server/solr/hdfs_core/
```

4. copy the dependencies jars

```
$ cp -r /opt/solr-8.5.1/dist/* /opt/solr-8.5.1/server/solr/hdfs_core/conf/lib
$ cp -r /opt/solr-8.5.1/contrib/extraction/lib /opt/solr-8.5.1/server/solr/hdfs_core/conf/lib
```

5.edit the config

```
$ vim solrconfig.xml

# lockType to hdfs
<lockType>${solr.lock.type:hdfs}</lockType>

# conf the dependency lib
<lib dir="/opt/solr-8.5.1/server/solr/hdfs_core/conf/lib" regex=".*\.jar"/>

# directoryFactory
  <directoryFactory name="DirectoryFactory" class="solr.HdfsDirectoryFactory">
    <str name="solr.hdfs.home">hdfs://10.1.236.196:8020/solr</str>
    <bool name="solr.hdfs.blockcache.enabled">true</bool>
    <int name="solr.hdfs.blockcache.slab.count">1</int>
    <bool name="solr.hdfs.blockcache.direct.memory.allocation">true</bool>
    <int name="solr.hdfs.blockcache.blocksperbank">16384</int>
    <bool name="solr.hdfs.blockcache.read.enabled">true</bool>
    <bool name="solr.hdfs.nrtcachingdirectory.enable">true</bool>
    <int name="solr.hdfs.nrtcachingdirectory.maxmergesizemb">16</int>
    <int name="solr.hdfs.nrtcachingdirectory.maxcachedmb">192</int>
  </directoryFactory>

# requestHandler for import data
  <requestHandler name="/dataimport" class="org.apache.solr.handler.dataimport.DataImportHandler">
    <lst name="defaults">
      <str name="config">data-config.xml</str>
    </lst>
  </requestHandler>

```

6.create data-config.xml

```
$ vim data-config.xml

<?xml version="1.0" encoding="UTF-8" ?>
<dataConfig>
  <dataSource name="fileDataSource" type="FileDataSource"/>
  <dataSource name="binFileDataSource" type="BinFileDataSource"/>
  <document>
    <entity name="office-files" dataSource="fileDataSource" rootEntity="false" processor="FileListEntityProcessor" baseDir="/data/zhaoyim/docs" fileName=".*\.(doc)|(docx)|(pdf)|(xls)|(xlsx)|(ppt)|(pptx)" recursive="true">
      <entity name="tika-office-files" processor="TikaEntityProcessor" url="${office-files.fileAbsolutePath}" format="text" dataSource="binFileDataSource" onError="skip">
        <field column="Author" name="author" meta="true"/>
        <field column="title" name="title" meta="true"/>
        <field column="text" name="text"/>
      </entity>
    </entity>
  </document>
</dataConfig>
```

7.edit managed-schema

```
    <field name="text" type="string" indexed="true" stored="true" required="false" multiValued="false"/>
    <field name="title" type="string" indexed="true" stored="true" required="false" multiValued="false"/>
    <field name="author" type="string" indexed="true" stored="true" required="false" multiValued="false"/>
```

8.start the server
```
# start
/opt/solr-8.5.1/bin/solr start # if use root user should add -force

# stop
/opt/solr-8.5.1/bin/solr/stop -all
```

9.add core in the solr UI
__note:__ 
set name to hdfs_core
set instanceDir to hdfs_core
![img](/img/in-post/20200526/solr-add-core.png)

after create the core successfully, check the hdfs path
![img](/img/in-post/20200526/solr-check-hdfs-path.png)

10. then can import pdf doc from folder __/data/zhaoyim/docs__