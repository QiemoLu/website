####创建mysql redis etcd 挂在目录
```shell
mkdir -p /apps/mysql/{mydir,datadir,conf,source}
mkdir -p /apps/redis/{datadir,conf,logs} 
mkdir -p /apps/etcd/{etc, etcd-data}
```
#### 当前目录创建docker-compose.yml
``` yaml
version: '3'
services:
  mysql:
    restart: always
    image: mysql:5.7.18
    container_name: mysql-lable
    volumes:
      - /apps/mysql/mydir:/mydir
      - /apps/mysql/datadir:/var/lib/mysql
      - /apps/mysql/conf/my.cnf:/etc/my.cnf
      # 数据库还原目录 可将需要还原的sql文件放在这里
      - /apps/mysql/source:/docker-entrypoint-initdb.d
    environment:
      - "MYSQL_ROOT_PASSWORD=123456"
      - "TZ=Asia/Shanghai"
    ports:
      # 使用宿主机的3306端口映射到容器的3306端口
      # 宿主机：容器
      - 3306:3306


  redis:
    restart: always
    image: redis:5.0
    container_name: redis-label
    volumes:
      - /apps/redis/datadir:/data
      - /apps/redis/conf/redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
#   #  两个写入操作 只是为了解决启动后警告 可以去掉
#    /bin/bash -c "echo 511 > /proc/sys/net/core/somaxconn
#    && echo never > /sys/kernel/mm/transparent_hugepage/enabled
#    && redis-server /usr/local/etc/redis/redis.conf"
    ports:
# 使用宿主机的端口映射到容器的端口
# 宿主机：容器
      - 6379:6379



  etcd:
    image: quay.io/coreos/etcd:v3.4.14
    container_name: etcd-label
    command: /usr/local/bin/etcd
    restart: always
    #networks:
    #  - deng
    ports:
      - "2379:2379"
      - "2380:2380"
    volumes:
      - "/apps/etcd/etcd-data:/etcd-data"
      - "/apps/etcd/etc/localtime:/etc/localtime:ro"
    environment:
      # 指定版本
      ETCDCTL_API: 3
      # 日志类型
      ETCD_LOGGER: capnslog
      # 存储路径
      ETCD_DATA_DIR: /etcd-data

      # 节点名称
      ETCD_NAME: node1
      # 创建集群唯一TOKEN
      INITIAL_CLUSTER_TOKEN: etcd-test-cluster

      # 该节点同伴监听地址
      ETCD_INITIAL_ADVERTISE_PEER_URLS: http://172.17.10.71:2380
      # 和其他节点通信的地址
      ETCD_LISTEN_PEER_URLS: http://0.0.0.0:2380

      # 对外公告该节点客户端监听地址
      ETCD_ADVERTISE_CLIENT_URLS: http://172.17.10.71:2379
      # 对外提供服务的地址
      ETCD_LISTEN_CLIENT_URLS: http://0.0.0.0:2379

      # 初始化集群所有节点列表（逗号隔开）
      ETCD_INITIAL_CLUSTER: node1=http://172.17.10.71:2380
      # 集群初始化状态（新建集群时为new）
      ETCD_INITIAL_CLUSTER_STATE: new
      # 启用K-V键值自动压缩存盘
      ETCD_AUTO_COMPACTION_RETENTION: 1


#networks:
#  deng:
#    external: true


```
#### 创建my.cnf
```markdown
[mysqld]
user=mysql
default-storage-engine=INNODB
character-set-server=utf8
character-set-client-handshake=FALSE
collation-server=utf8_unicode_ci
init_connect='SET NAMES utf8'
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8

```
#### redis.conf配置一下访问和持久化
```shell
 bind 0.0.0.0
 appendonly yes
```

#### 启动查看
```markdown
dokcer-compose up -d
docker-compose ps
docker-compose logs $service_name
```
