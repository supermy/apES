
[toc]

#ELK 进行全文检索与条件查询

应用场景：
自动同步业务数据
1.全文检索；
2.使用 es-dsl 进行通用组合查询；
3.提供过滤 api,进行动态下拉检索；


###部署 es

####下载并且解压

```aidl
    
    wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.6.1.zip
    tar -xzvf elasticsearch-5.6.1.zip
    
    允许插件访问
    vim elasticsearch.yml
    http.cors.enabled: true
    http.cors.allow-origin: "*"
    
 
    
```

####启动 es

#####前台运行
```$xslt
    ./bin/elasticsearch
```

#####后台运行
```$xslt
    ./bin/elasticsearch -d
```



####安装es插件

#####安装 x-Pack 插件

###### X-Pack是一个Elastic Stack的扩展，将安全，警报，监视，报告和图形功能包含在一个易于安装的软件包中。

```$xslt

    命令安装，MAC 因为.DS_Store报错
    bin/elasticsearch-plugin install x-pack -v
    
    手动安装-科学打开方式
    wget https://artifacts.elastic.co/downloads/elasticsearch-plugins/x-pack/x-pack-5.6.1.zip
    
    解压
    unzip x-pack-5.6.1.zip
    
    目录移动到插件目录
    mv elasticsearch plugins
    
    默认的用户名：elastic，密码：changeme。可以通过 curl 修改默认密码：
    curl -XPUT -u elastic 'localhost:9200/_xpack/security/user/elastic/_password' -d '{
      "password" : "123456"
    }    
    
    x-pack 插件测试
    http://127.0.0.1:9200/_xpack/
    
```
###### head 插件安装 ,数据浏览与查询

```$xslt
    
    git clone git://github.com/mobz/elasticsearch-head.git
    cd elasticsearch-head
    npm install
    npm install grunt --save
    
    修改elasticsearch-head下Gruntfile.js文件，默认监听在127.0.0.1下9200端口， 
    
    执行
    cd elasticsearch-head 
    grunt server
    
    浏览器访问 
    http://172.16.31.220:9100/
```



###### elasticsearch-analysis-ik 插件安装，全文检索与条件组合查询，提供下拉检索 api

* 安装 注意版本号与 es 要一一对应

```aidl
    
    命令行安装
    ./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.6.1/elasticsearch-analysis-ik-5.6.1.zip
    
    下载源码
    wget  https://github.com/medcl/elasticsearch-analysis-ik/archive/v5.6.1.zip
    
    编译
    mvn package
    
    将releases目录 zip 解压到 es/plugin

```
* 自定义字典配置

```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
    <properties>
        <comment>IK Analyzer 扩展配置</comment>
        <!--用户可以在这里配置自己的扩展字典 -->
        <entry key="ext_dict">custom/mydict.dic;custom/single_word_low_freq.dic</entry>
         <!--用户可以在这里配置自己的扩展停止词字典-->
        <entry key="ext_stopwords">custom/ext_stopword.dic</entry>
        <!--用户可以在这里配置远程扩展字典 -->
        <entry key="remote_ext_dict">location</entry>
        <!--用户可以在这里配置远程扩展停止词字典-->
        <entry key="remote_ext_stopwords">http://xxx.com/xxx.dic</entry>
    </properties>
```

* 热更新 IK 分词使用方法

目前该插件支持热更新 IK 分词，通过上文在 IK 配置文件中提到的如下配置

```xml
 	<!--用户可以在这里配置远程扩展字典 -->
	<entry key="remote_ext_dict">location</entry>
 	<!--用户可以在这里配置远程扩展停止词字典-->
	<entry key="remote_ext_stopwords">location</entry>


其中 `location` 是指一个 url，比如 `http://yoursite.com/getCustomDict`，该请求只需满足以下两点即可完成分词热更新。

1. 该 http 请求需要返回两个头部(header)，一个是 `Last-Modified`，一个是 `ETag`，这两者都是字符串类型，只要有一个发生变化，该插件就会去抓取新的分词进而更新词库。

2. 该 http 请求返回的内容格式是一行一个分词，换行符用 `\n` 即可。

满足上面两点要求就可以实现热更新分词了，不需要重启 ES 实例。

可以将需自动更新的热词放在一个 UTF-8 编码的 .txt 文件里，放在 nginx 或其他简易 http server 下，当 .txt 文件修改时，http server 会在客户端请求该文件时自动返回相应的 Last-Modified 和 ETag。可以另外做一个工具来从业务系统提取相关词汇，并更新这个 .txt 文件。
```

* 测试 elasticsearch-analysis-ik

1.create a index

```bash
curl -XPUT http://localhost:9200/index
```

2.create a mapping

```bash
ik_max_word 和 ik_smart 什么区别?
ik_max_word: 会将文本做最细粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,中华人民,中华,华人,人民共和国,人民,人,民,共和国,共和,和,国国,国歌”，会穷尽各种可能的组合；
ik_smart: 会做最粗粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,国歌”。


curl -XPOST http://localhost:9200/index/fulltext/_mapping -d'
{
        "properties": {
            "content": {
                "type": "text",
                "analyzer": "ik_max_word",
                "search_analyzer": "ik_max_word"
            }
        }
    
}'
```

3.index some docs

```bash
curl -XPOST http://localhost:9200/index/fulltext/1 -d'
{"content":"美国留给伊拉克的是个烂摊子吗"}
'
```

```bash
curl -XPOST http://localhost:9200/index/fulltext/2 -d'
{"content":"公安部：各地校车将享最高路权"}
'
```

```bash
curl -XPOST http://localhost:9200/index/fulltext/3 -d'
{"content":"中韩渔警冲突调查：韩警平均每天扣1艘中国渔船"}
'
```

```bash
curl -XPOST http://localhost:9200/index/fulltext/4 -d'
{"content":"中国驻洛杉矶领事馆遭亚裔男子枪击 嫌犯已自首"}
'
```

4.query with highlighting

```bash
curl -XPOST http://localhost:9200/index/fulltext/_search  -d'
{
    "query" : { "match" : { "content" : "中国" }},
    "highlight" : {
        "pre_tags" : ["<tag1>", "<tag2>"],
        "post_tags" : ["</tag1>", "</tag2>"],
        "fields" : {
            "content" : {}
        }
    }
}
'
```



###部署 kibana ,数据的可视化展现

```aidl
    
    wget https://www.elastic.co/downloads/kibana
    
    解压 
    kibana-5.6.0-SNAPSHOT-darwin-x86_64.tar
    
    运行（可先去掉 es 的 xpack）
    bin/kibana
    
    后台运行
    nohup bin/kibana &
    

    启动成功后浏览器打开 
    http://localhost:5601/
    
    这里默认的用户名：elastic，密码：changeme。可以通过 curl 修改默认密码：
    curl -XPUT -u elastic 'localhost:9200/_xpack/security/user/kibana/_password' -d '{
      "password" : "123456"
    }'
    
    Kibana 可视化管理
    在页面左侧选择【DevTools】，就可以进行相应 REST 操作，使用 DSL 语句查询的时候在这个开发者工具界面能提供很多有用的输入提示：

   
    
    

```

####部署 kibana插件 x-pack -正确打开方式

安装完成之后访问需要密码，默认的用户名：elastic，密码：changeme

```aidl
       
    安装插件
    bin/kibana-plugin install x-pack
    
```


###部署 logstash ,数据采集

```aidl

    下载
    logstash-5.6.1.tar.gz
    
    解压
    tar -xzvf logstash-5.6.1.tar.gz
    
    运行
    bin/logstash
    
    测试是否成功
    bin/logstash -e 'input { stdin { } } output { stdout {codec=>rubydebug} }'
    
    测试配置文件
    bin/logstash -t -f etc/
    
    后台运行
    nohup bin/logstash -f etc/ &
    
    停止
    ps -ef |grep logstash 
    kill -9  id

    

```

####部署 kibana插件 logstash-input-jdbc

```aidl

       
    安装插件 x-pack
    bin/logstash-plugin install x-pack
    
    修改密码
    curl -XPUT -u elastic 'localhost:9200/_xpack/security/user/kibana/_password' -d '{
      "password" : "123456"
    }'
    
```


```aidl

    更换为国内的 ruby 源
    gem sources --add https://gems.ruby-china.org/ --remove https://rubygems.org/
    gem sources -l

    gem install bundler
    

    安装插件 logstash-input-jdbc
    bin/logstash-plugin install logstash-input-jdbc
    

    jdbc 配置
    input {
      jdbc {
        jdbc_driver_library => "mysql-connector-java-5.1.38.jar"
        jdbc_driver_class => "com.mysql.jdbc.Driver"
        jdbc_connection_string => "jdbc:mysql://127.0.0.1:3306/test"
        jdbc_user => "root"
        jdbc_password => "123456"
      # or jdbc_password_filepath => "/path/to/my/password_file"
        statement => "select id,foo,bar from mysql101"
        jdbc_paging_enabled => "true"
        jdbc_page_size => "50000"
      }
    }
    
    filter {
     
    }
    
    output {
      stdout {
        codec => rubydebug
      }
      elasticsearch {
        hosts => "127.0.0.1"
        index => "tourist_toilet"
        document_type =>”fulltext"
      }
    }
    
    
    
 
    执行之前配置索引与全文检索
    建立索引
    
    curl -XPUT http://localhost:9200/mysql101
    
    建立全文检索
    curl -XPOST http://localhost:9200/mysql101/fulltext/_mapping -d'
    {
            "properties": {
                "content": {
                    "type": "text",
                    "analyzer": "ik_max_word",
                    "search_analyzer": "ik_max_word"
                }
            }
        
    }’
    
    运行
    logstash-5.6.1/bin/logstash -f jdbc.conf
    
    查看数据同步情况
    http://127.0.0.1:9200/mysql101/_search
    
    

```

### 轻量型数据采集器 Beats

```aidl

    安装常用指标采集器
    wget https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-5.6.1-darwin-x86_64.tar.gz
    tar -xzvf metricbeat-5.6.1-darwin-x86_64.tar.gz
    
    安装仪表盘模板
    ./scripts/import_dashboards -es http://127.0.0.1:9200 -user elastic -pass changeme
    
    
    运行
    ./metricbeat -e -c metricbeat.yml
    
```


### supervisor 安装、配置、常用命令

Supervisor 是一个用 Python 写的进程管理工具，可以很方便的用来在 UNIX-like 系统（不支持 Windows）下启动、重启（自动重启程序）、关闭进程（不仅仅是 Python 进程）。

```aidl

    需要一个工具，时刻监控 web 应用的运行情况，管理该进程。
    
    Supervisor 就是解决这种需求的工具，可以保证程序崩溃后，重新把程序启动起来等功能。
    
```

###云化一键安装

```aidl
    bash <(curl -fsSL https://download.elastic.co/cloud/elastic-cloud-enterprise.sh) install
```

####FAQ
```$xslt
    官方 demo 示例
    http://demo.elastic.co/app/kibana
    
    删除当前目录的.DS_store 
    find . -name '*.DS_Store' -type f -delete
    
    删除所有目录的.DS_store 
    sudo find / -name “.DS_Store” -depth -exec rm {} \;
    
    禁止.DS_store生成： 
    defaults write com.apple.desktopservices DSDontWriteNetworkStores -bool TRUE
    
    恢复.DS_store生成： 
    defaults delete com.apple.desktopservices DSDontWriteNetworkStores
    
    ELK中无法启动kibana，解决“Elasticsearch is still initializing the kibana index... ”
    需要删除.kibana的索引。
     curl -XDELETE http://localhost:9200/.kibana
     curl -XDELETE http://localhost:9200/*

    
    ES 启动问题处理   
    切换到root用户修改配置sysctl.conf
    vi /etc/sysctl.conf 
    添加下面配置：
    vm.max_map_count=655360
    并执行命令：
    sysctl -p
    
    Debian 系统先行安装
    apt-get install ruby-full
    
    禁用X-Pack 插件 security
    vim config/kibana.yml
    vim config/elasticsearch.yml
    添加以下内容
    xpack.security.enabled: false
    
    curl -XPUT -u elastic 'http://<host>:<port>/_xpack/license' -H "Content-Type: application/json" -d @license.json
    
```