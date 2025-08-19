# Elasticsearch 服务搭建

> Elasticcsearch 是一种分布式的 JSON 格式数据交互的数据检索引擎
>
> Elasticsearch 的四大金刚：
>
> ![image-20250807095813632](Elasticsearch 服务搭建.assets/image-20250807095813632.png)

## 使用 Docker Compose 安装

创建 `docker-compose.yml` 文件:

~~~ yaml
version: '3'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.2
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - xpack.security.enabled=false # 禁用安全功能(开发环境)
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - es_network

volumes:
  es_data:
    driver: local

networks:
  es_network:
    driver: bridge
~~~

创建文件到自定义目录，在确保有 `docker-compose`脚本的前提下使用 `docker-compose up -d` 命令启动

~~~ shell
docker-compose up -d
~~~

命令行验证安装

~~~ shell
curl -X GET "http://localhost:9200/"
~~~

## 启动失败错误

~~~ shell
#通常表示Elasticsearch 容器因为内存不足而被强制终止
Exited (137) 2 seconds ago elasticsearch
~~~

修改 ymal 文件，添加启动指定的内存

~~~ yaml
environment:
  - ES_JAVA_OPTS=-Xms512m -Xmx512m
~~~

## 安装 Kibana

在 `dockers-compose` 文件当中添加

~~~ yaml
kibana:
  image: docker.elastic.co/kibana/kibana:7.15.2
  container_name: kibana
  ports:
    - "5601:5601"
  environment:
    - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
  depends_on:
    - elasticsearch
  networks:
    - es_network
~~~

