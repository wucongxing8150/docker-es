## 安装

### Github

git@github.com:wucongxing8150/docker-es.git

### es

```Shell
# 设置max_map_count的值
cat /proc/sys/vm/max_map_count sysctl -w vm.max_map_count=262144

docker run -d --name local-es -p 9200:9200 -p 9300:9300 -e ES_JAVA_OPTS="-Xms512m -Xmx512m" -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.3.2
```

### es-head

```Shell
docker pull mobz/elasticsearch-head:5
docker run --name local-es-head -p 9100:9100 mobz/elasticsearch-head:5

# 解决操作时406问题
docker cp local-es-head:/usr/src/app/_site/vendor.js ./

# 6886行
// contentType: "application/x-www-form-urlencoded",
contentType: "application/json;charset=UTF-8",

# 7574行
// var inspectData = s.contentType === "application/x-www-form-urlencoded" &&
var inspectData = s.contentType === "application/json;charset=UTF-8" &&        

docker cp vendor.js  local-es-head:/usr/src/app/_site
```

### kibana

```Shell
docker pull docker.elastic.co/kibana/kibana:7.6.2

docker run  --name kibana --link local-es-01 -p 5601:5601 docker.elastic.co/kibana/kibana:7.6.2
```

### docker-compose

```YAML
version: '2.2'
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    container_name: local-es-01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./es/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./es/elastic-certificates.p12:/usr/share/elasticsearch/config/elastic-certificates.p12
    ports:
      - 9200:9200
    networks:
      - elastic

  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    container_name: local-es-02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./es/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - ./es/elastic-certificates.p12:/usr/share/elasticsearch/config/elastic-certificates.p12
    networks:
      - elastic

  kibana:
    depends_on: 
      - es01
    image: docker.elastic.co/kibana/kibana:7.6.2
    container_name: local-kibana
    ports:
      - 5601:5601
    environment:
      ELASTICSEARCH_URL: http://es01:9200
      ELASTICSEARCH_HOSTS: http://es01:9200
    volumes:
      - ./kibana/kibana.yml:/usr/share/kibana/config/kibana.yml
    networks:
      - elastic
  es-head:
    image: mobz/elasticsearch-head:5
    container_name: local-es-head
    ports:
      - 9100:9100
    networks:
      - elastic
volumes:
  data01:
    driver: local
  data02:
    driver: local
networks:
  elastic:
    driver: bridge
```

## 注意事项

- 安装es-head，需调整es跨域问题

```Shell
# 进入es容器内部
cd /usr/share/elasticsearch/config
# 或
cd /config
http.cors.enabled: true
http.cors.allow-origin: "*"
```

- es秘钥问题

```Shell
docker exec -it es01 /bin/bash

[root@bedf706ff508 elasticsearch]# ./bin/elasticsearch-setup-passwords -h
Sets the passwords for reserved users

Commands
--------
auto - Uses randomly generated passwords
interactive - Uses passwords entered by a user

Non-option arguments:
command

Option             Description
------             -----------
-E <KeyValuePair>  Configure a setting
-h, --help         Show help
-s, --silent       Show minimal output
-v, --verbose      Show verbose output
[root@bedf706ff508 elasticsearch]# ./bin/elasticsearch-setup-passwords auto
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,logstash_system,beats_system,remote_monitoring_user.
The passwords will be randomly generated and printed to the console.
Please confirm that you would like to continue [y/N]y


Changed password for user apm_system
PASSWORD apm_system = HL8opbKgWpi2iEAyR2Ev

Changed password for user kibana
PASSWORD kibana = 9nmgZZfZv66XEF7y7jNh

Changed password for user logstash_system
PASSWORD logstash_system = Gf05fZG0i9C6ZkAfxxkB

Changed password for user beats_system
PASSWORD beats_system = o8FQXa8DrTwPXgH1nCZM

Changed password for user remote_monitoring_user
PASSWORD remote_monitoring_user = sODKhrNgAp0E7wESr5qA

Changed password for user elastic
PASSWORD elastic = jox3OT92zSJ1WLL9rW1q

# 重启
```

- es-head访问问题

```Shell
http://127.0.0.1:9100/?auth_user=elastic&auth_password=jox3OT92zSJ1WLL9rW1q
```

- Kibana访问问题

```Shell
# 修改kibana.yml
elasticsearch.username: elastic
elasticsearch.password: jox3OT92zSJ1WLL9rW1q
```
