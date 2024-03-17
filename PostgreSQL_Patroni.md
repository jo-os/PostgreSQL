## PostgreSQL - Patroni

https://habr.com/ru/companies/otus/articles/753294/
```
/etc/hosts

127.0.0.1 localhost node1
192.168.10.1 node1
192.168.10.2 node2
192.168.10.3 node3
```
### etcd
```
sudo apt-get install etcd
```
node1
```
/etc/default/etcd

ETCD_NAME=node1
ETCD_INITIAL_CLUSTER="node1=http://192.168.10.1:2380"
ETCD_INITIAL_CLUSTER_TOKEN="devops_token"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.10.1:2380"
ETCD_DATA_DIR="/var/lib/etcd/postgresql"
ETCD_LISTEN_PEER_URLS="http://192.168.10.1:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.10.1:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.10.1:2379"
```
```
sudo systemctl restart etcd
```
node2
```
/etc/default/etcd

ETCD_NAME=node2
ETCD_INITIAL_CLUSTER="node1=http://192.168.10.1:2380,node2=http://192.168.10.2:2380"
ETCD_INITIAL_CLUSTER_TOKEN="devops_token"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.10.2:2380"
ETCD_DATA_DIR="/var/lib/etcd/postgresql"
ETCD_LISTEN_PEER_URLS="http://192.168.10.2:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.10.2:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.10.2:2379"
```
node1
```
sudo etcdctl member add node2 http://192.168.10.2:2380
```
node2
```
sudo systemctl restart etcd
```
node3
```
/etc/default/etcd

ETCD_NAME=node3
ETCD_INITIAL_CLUSTER="node1=http://192.168.10.1:2380,node2=http://192.168.10.2:2380,node3=http://192.168.10.3:2380"
ETCD_INITIAL_CLUSTER_TOKEN="devops_token"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.10.3:2380"
ETCD_DATA_DIR="/var/lib/etcd/postgresql"
ETCD_LISTEN_PEER_URLS="http://192.168.10.3:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.10.3:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.10.3:2379"
```
node1
```
sudo etcdctl member add node3 http://192.168.10.3:2380
```
node3
```
sudo systemctl restart etcd
```
Проверка
```
sudo etcdctl member list
```
### Watchdog
```
sudo sh -c 'echo "softdog" >> /etc/modules'
sudo sh -c 'echo "KERNEL=="watchdog", OWNER="postgres", GROUP="postgres"" >> /etc/udev/rules.d/61-watchdog.rules'
```
Проверяем и удаляем все что находим
```
grep blacklist /lib/modprobe.d/* /etc/modprobe.d/* |grep softdog
```
Перезапускаем ВМ и проверяем
```
lsmod | grep softdog
```
### PostgreSQL
```
sudo apt-get update -y; sudo apt-get install -y wget gnupg2 lsb-release curl
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo dpkg -i percona-release_latest.generic_all.deb
sudo apt-get update
sudo percona-release setup ppg-15
sudo apt-get install percona-postgresql-12
```
Запуск и инициализацию будет осуществлять через patroni, поэтому отключаем и удаляем
```
sudo systemctl disable postgresql
```
```
sudo systemctl stop postgresql
sudo rm -fr /var/lib/postgresql/15/main
```
### Patroni
```
sudo apt-get install percona-patroni
```
node1
```
/etc/patroni/patroni.yml
scope: stampede
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: node1:8008

etcd:
  host: node1:2379

bootstrap:
  # this section will be written into Etcd:/<namespace>/<scope>/config after initializing new cluster
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
#    master_start_timeout: 300
#    synchronous_mode: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        logging_collector: 'on'
        max_wal_senders: 5
        max_replication_slots: 5
        wal_log_hints: "on"
        #archive_mode: "on"
        #archive_timeout: 600
        #archive_command: "cp -f %p /home/postgres/archived/%f"
        #recovery_conf:
        #restore_command: cp /home/postgres/archived/%f %p

  # some desired options for 'initdb'
  initdb:  # Note: It needs to be a list (some options need values, others are switches)
  - encoding: UTF8
  - data-checksums

  pg_hba:  # Add following lines to pg_hba.conf after running 'initdb'
  - host replication replicator 192.168.10.1/24 md5
  - host replication replicator 127.0.0.1/32 trust
  - host all all 192.168.10.1/24 md5
  - host all all 0.0.0.0/0 md5
#  - hostssl all all 0.0.0.0/0 md5

  # Additional script to be launched after initial cluster creation (will be passed the connection URL as parameter)
# post_init: /usr/local/bin/setup_cluster.sh
  # Some additional users users which needs to be created after initializing new cluster
  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb

postgresql:
  listen: 0.0.0.0:5432
  connect_address: node1:5432
  data_dir: "/var/lib/postgresql/12/main"
  bin_dir: "/usr/lib/postgresql/12/bin"
#  config_dir:
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: vagrant
    superuser:
      username: postgres
      password: vagrant
  parameters:
    unix_socket_directories: '/var/run/postgresql'

watchdog:
  mode: required # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```
```
sudo systemctl restart patroni
```
Проверяем
```
psql -U postgres
```
Повторяем установку Patroni на node2 и node3 - меняем в конфиге node1 на соответствующие ноды.

Проверяем
```
sudo patronictl -c /etc/patroni/config.yml list
```
### HAProxy
```
sudo apt-get install haproxy
```
```
/etc/haproxy/haproxy.cfg
global
    maxconn 100

defaults
    log    global
    mode    tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen primary
    bind *:5000
    option httpchk OPTIONS /master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 node1:5432 maxconn 100 check port 8008
    server node2 node2:5432 maxconn 100 check port 8008
    server node3 node3:5432 maxconn 100 check port 8008

listen standbys
    balance roundrobin
    bind *:5001
    option httpchk OPTIONS /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 node1:5432 maxconn 100 check port 8008
    server node2 node2:5432 maxconn 100 check port 8008
    server node3 node3:5432 maxconn 100 check port 8008
```
Для тестов
```
echo "localhost:5000:postgres:postgres:vagrant" > ~/.pgpass
echo "localhost:5001:postgres:postgres:vagrant" >> ~/.pgpass
chmod 0600 ~/.pgpass
```




