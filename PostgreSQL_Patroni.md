## PostgreSQL - Patroni

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

```
sudo systemctl disable postgresql
```
```
sudo systemctl stop postgresql
sudo rm -fr /var/lib/postgresql/15/main
```



