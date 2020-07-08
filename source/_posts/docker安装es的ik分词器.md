---
title: “docker安装es的ik分词器”
date: 2020-06-22 13:55:53
tags: ElasticSearch
categories: ElasticSearch
---
### 背景说明
跟着极客时间上的光头老师操作，使用docker-compose的方式启动ES集群，第二章安装IK和HanLP分词器。  
可他没讲docker的安装方式，而且我是基于windows安装的docker，所以有了这个文章做记录。  
docker-compose.yaml文件是完全拷贝的： 
```
version: '2.2'
services:
  cerebro:
    image: lmenezes/cerebro:0.8.3
    container_name: cerebro
    ports:
      - "9000:9000"
    command:
      - -Dhosts.0.host=http://elasticsearch:9200
    networks:
      - es7net
  kibana:
    image: docker.elastic.co/kibana/kibana:7.1.0
    container_name: kibana7
    environment:
      - I18N_LOCALE=zh-CN
      - XPACK_GRAPH_ENABLED=true
      - TIMELION_ENABLED=true
      - XPACK_MONITORING_COLLECTION_ENABLED="true"
    ports:
      - "5601:5601"
    networks:
      - es7net
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.1.0
    container_name: es7_01
    environment:
      - cluster.name=geektime
      - node.name=es7_01
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.seed_hosts=es7_01,es7_02
      - cluster.initial_master_nodes=es7_01,es7_02
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es7data1:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - es7net
  elasticsearch2:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.1.0
    container_name: es7_02
    environment:
      - cluster.name=geektime
      - node.name=es7_02
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.seed_hosts=es7_01,es7_02
      - cluster.initial_master_nodes=es7_01,es7_02
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es7data2:/usr/share/elasticsearch/data
    networks:
      - es7net


volumes:
  es7data1:
    driver: local
  es7data2:
    driver: local

networks:
  es7net:
    driver: bridge
```

### 安装过程
#### 本地ES安装插件脚本
这是常规的ES安装脚本，看完docker-compose.yaml文件，里面没有配置本地docker的地址，所以可以确定这个命令只是给本地的ES安装插件，对docker容器是无效的。  
```
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.1.0/elasticsearch-analysis-ik-7.1.0.zip
```

#### docker容器安装ES插件
这段是网上找的，照着操作，可是网络不给力，试了一晚上都没成功。  
```
docker-compose exec es01 elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.2.0/elasticsearch-analysis-ik-7.2.0.zip
docker-compose exec es02 elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.2.0/elasticsearch-analysis-ik-7.2.0.zip
//然后要重启es容器
docker-compose restart es01
docker-compose restart es02
```
等了几小时，出现这种错误，内心是难以平复的。  
```
D:\docker\7.x-docker-2-es-instances>docker-compose exec elasticsearch elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.1.0/elasticsearch-analysis-ik-7.1.0.zip
-> Downloading https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.1.0/elasticsearch-analysis-ik-7.1.0.zip
[=================================================] 100%??
Exception in thread "main" java.io.EOFException: Unexpected end of ZLIB input stream
        at java.base/java.util.zip.InflaterInputStream.fill(InflaterInputStream.java:245)
        at java.base/java.util.zip.InflaterInputStream.read(InflaterInputStream.java:159)
        at java.base/java.util.zip.ZipInputStream.read(ZipInputStream.java:195)
        at java.base/java.io.FilterInputStream.read(FilterInputStream.java:107)
        at org.elasticsearch.plugins.InstallPluginCommand.unzip(InstallPluginCommand.java:652)
        at org.elasticsearch.plugins.InstallPluginCommand.execute(InstallPluginCommand.java:230)
        at org.elasticsearch.plugins.InstallPluginCommand.execute(InstallPluginCommand.java:216)
        at org.elasticsearch.cli.EnvironmentAwareCommand.execute(EnvironmentAwareCommand.java:86)
        at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:124)
        at org.elasticsearch.cli.MultiCommand.execute(MultiCommand.java:77)
        at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:124)
        at org.elasticsearch.cli.Command.main(Command.java:90)
        at org.elasticsearch.plugins.PluginCli.main(PluginCli.java:47)
```

#### 通过file协议安装
无意间通过浏览器直接把elasticsearch-analysis-ik-7.1.0.zip下载了，尝试通过file协议安。  
结果是找不到文件，又是内心难以平复。  
```
D:\docker\7.x-docker-2-es-instances>docker-compose exec elasticsearch elasticsearch-plugin install file:///D:/docker/7.x-docker-2-es-instances/elasticsearch-analysis-ik-7.1.0.zip
-> Downloading file:///D:/docker/7.x-docker-2-es-instances/elasticsearch-analysis-ik-7.1.0.zip
Exception in thread "main" java.io.FileNotFoundException: /D:/docker/7.x-docker-2-es-instances/elasticsearch-analysis-ik-7.1.0.zip (No such file or directory)
        at java.base/java.io.FileInputStream.open0(Native Method)
        at java.base/java.io.FileInputStream.open(FileInputStream.java:213)
        at java.base/java.io.FileInputStream.<init>(FileInputStream.java:155)
        at java.base/java.io.FileInputStream.<init>(FileInputStream.java:110)
        at java.base/sun.net.www.protocol.file.FileURLConnection.connect(FileURLConnection.java:86)
        at java.base/sun.net.www.protocol.file.FileURLConnection.getInputStream(FileURLConnection.java:184)
        at org.elasticsearch.plugins.InstallPluginCommand.downloadZip(InstallPluginCommand.java:380)
        at org.elasticsearch.plugins.InstallPluginCommand.download(InstallPluginCommand.java:278)
        at org.elasticsearch.plugins.InstallPluginCommand.execute(InstallPluginCommand.java:229)
        at org.elasticsearch.plugins.InstallPluginCommand.execute(InstallPluginCommand.java:216)
        at org.elasticsearch.cli.EnvironmentAwareCommand.execute(EnvironmentAwareCommand.java:86)
        at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:124)
        at org.elasticsearch.cli.MultiCommand.execute(MultiCommand.java:77)
        at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:124)
        at org.elasticsearch.cli.Command.main(Command.java:90)
        at org.elasticsearch.plugins.PluginCli.main(PluginCli.java:47)
```

#### 进入docker容器再通过file协议安装
先从本地拷贝到docker容器：  
```
D:\docker\7.x-docker-2-es-instances>docker cp D:/elasticsearch-analysis-ik-7.1.0.zip es7_02:/opt/elasticsearch-analysis-ik-7.1.0.zip
```

再进入docker:  
```
docker exec -it es7_01 /bin/bash
```

最后执行安装：  
```
[root@1105e84f27f9 elasticsearch]# cd plugins/
[root@1105e84f27f9 plugins]#  elasticsearch-plugin install file///opt/elasticsearch-analysis-ik-7.1.0.zip
^C[root@1105e84f27f9 plugins]#  elasticsearch-plugin install file:///opt/elasticsearch-analysis-ik-7.1.0.zip
-> Downloading file:///opt/elasticsearch-analysis-ik-7.1.0.zip
[=================================================] 100%??
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@     WARNING: plugin requires additional permissions     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
* java.net.SocketPermission * connect,resolve
See http://docs.oracle.com/javase/8/docs/technotes/guides/security/permissions.html
for descriptions of what these permissions allow and the associated risks.

Continue with installation? [y/N]y
-> Installed analysis-ik
[root@1105e84f27f9 plugins]#
[root@1105e84f27f9 plugins]# exit
```

### 验证IK安装成功
```
POST _analyze
{
  "analyzer": "ik_max_word",
  "text": ["剑桥分析公司多位高管对卧底记者说，他们确保了唐纳德·特朗普在总统大选中获胜"]
}
```

![Alt text](uploads/20200623001.png)