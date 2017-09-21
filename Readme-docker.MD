
[toc]

#ELK 进行全文检索与条件查询-Docker部署

应用场景：
自动同步业务数据
1.全文检索；
2.使用 es-dsl 进行通用组合查询；
3.提供过滤 api,进行动态下拉检索；


###部署 es-docker

开发模式
```aidl
docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:5.6.1
```

产品模式
```aidl
docker-compose up

docker-compose.yml:

version: '2'
services:
  elasticsearch1:
    image: docker.elastic.co/elasticsearch/elasticsearch:5.6.1
    container_name: elasticsearch1
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    mem_limit: 1g
    volumes:
      - esdata1:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - esnet
  elasticsearch2:
    image: docker.elastic.co/elasticsearch/elasticsearch:5.6.1
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "discovery.zen.ping.unicast.hosts=elasticsearch1"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    mem_limit: 1g
    volumes:
      - esdata2:/usr/share/elasticsearch/data
    networks:
      - esnet

volumes:
  esdata1:
    driver: local
  esdata2:
    driver: local

networks:
  esnet:
```

测试
```aidl
    curl -u elastic http://127.0.0.1:9200/_cat/health
```

自定义镜像
```aidl
    docker build --tag=elasticsearch-custom .
    
```


```aidl
docker pull docker.elastic.co/beats/heartbeat:5.6.1

docker run \
  -v ~/heartbeat.yml:/usr/share/heartbeat/heartbeat.yml \
  docker.elastic.co/beats/heartbeat:5.6.1
```

文件监控
```aidl
docker pull docker.elastic.co/beats/filebeat:5.6.1

docker run \
  -v ~/filebeat.yml:/usr/share/filebeat/filebeat.yml \
  docker.elastic.co/beats/filebeat:5.6.1
```

监控宿主机-指标监控
```aidl
docker pull docker.elastic.co/beats/metricbeat:5.6.1

docker run \
  -v ~/metricbeat.yml:/usr/share/metricbeat/metricbeat.yml \
  docker.elastic.co/beats/metricbeat:5.6.1
  
docker run \
  --volume=/proc:/hostfs/proc:ro \ 
  --volume=/sys/fs/cgroup:/hostfs/sys/fs/cgroup:ro \ 
  --volume=/:/hostfs:ro \ 
  --net=host 
  docker.elastic.co/beats/metricbeat:5.6.1 -system.hostfs=/hostfs
    
```

监控另一个容器-指标监控
```aidl
docker run \
  --network=mysqlnet \ 
  -e MYSQL_PASSWORD=secret \ 
  docker.elastic.co/beats/metricbeat:5.6.1
```

监控网络数据包
```aidl
docker pull docker.elastic.co/beats/packetbeat:5.6.1

docker run \
  -v ~/packetbeat.yml:/usr/share/packetbeat/packetbeat.yml \
  docker.elastic.co/beats/packetbeat:5.6.1
```

数据采集
```aidl
docker pull docker.elastic.co/logstash/logstash:5.6.1

docker run --rm -it -v ~/pipeline/:/usr/share/logstash/pipeline/ docker.elastic.co/logstash/logstash:5.6.1

docker run --rm -it -v ~/settings/logstash.yml:/usr/share/logstash/config/logstash.yml docker.elastic.co/logstash/logstash:5.6.1

```

```aidl
docker pull docker.elastic.co/kibana/kibana:5.6.1

services:
  kibana:
    image: docker.elastic.co/kibana/kibana:5.6.1
    volumes:
      - ./kibana.yml:/usr/share/kibana/config/kibana.yml

services:
  kibana:
    image: docker.elastic.co/kibana/kibana:5.6.1
    environment:
      SERVER_NAME: kibana.example.org
      ELASTICSEARCH_URL: http://elasticsearch.example.org
      
```