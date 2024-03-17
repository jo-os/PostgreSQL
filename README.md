# PostgreSQL
PostgreSQL replication

Install Postgresql

On both
```
yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
yum install -y postgresql14-server postgresql14-contrib postgresql14
yum install -y repmgr_14
```
On master
```
/usr/pgsql-14/bin/postgresql-14-setup initdb
systemctl enable --now postgresql-14
```
On both
```
vi /var/lib/pgsql/14/data/postgresql.conf

listen_addresses = '*'
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
hot_standby = on
wal_log_hints = on
archive_mode = on
archive_command = '/bin/true'
wal_keep_size = 1GB
```
```
vi /var/lib/pgsql/14/data/pg_hba.conf

host all repmgr 192.168.1.155/32 scram-sha-256
host all repmgr 192.168.1.156/32 scram-sha-256

host replication repmgr 192.168.1.155/32 md5
host replication repmgr 192.168.1.156/32 md5
```
On master
```
su postgres
psql

CREATE USER repmgr WITH ENCRYPTED PASSWORD 'repl123!7a';
CREATE DATABASE repmgr OWNER repmgr;
ALTER USER repmgr WITH SUPERUSER;
ALTER USER repmgr SET search_path TO repmgr, "$user", public;
```
On both
```
sudo su - postgres
vi ~/.pgpass

# hostname:port:database:username:password
*:*:*:repmgr:repl123!7a

chmod 600 ~/.pgpass
```
On Slave
```
psql -X -U repmgr -h 192.168.1.155 -c "IDENTIFY_SYSTEM" replication=1
```
On Master
```
vi /etc/repmgr/14/repmgr.conf

node_id=1
node_name='d00ps0001'
conninfo='host=192.168.1.155 dbname=repmgr user=repmgr'
data_directory='/var/lib/pgsql/14/data/'
use_replication_slots=true
log_level='INFO'
log_file='/var/log/repmgr/repmgr-14.log'
pg_bindir='/usr/pgsql-14/bin/'

service_start_command = 'sudo systemctl start postgresql-14'
service_stop_command = 'sudo systemctl stop postgresql-14'
service_restart_command = 'sudo systemctl restart postgresql-14'
service_reload_command = 'sudo systemctl reload postgresql-14'

sudo -u postgres /usr/pgsql-14/bin/repmgr -f /etc/repmgr/14/repmgr.conf master register
sudo -u postgres /usr/pgsql-14/bin/repmgr -f /etc/repmgr/14/repmgr.conf cluster show
```
On Slave
```
vi /etc/repmgr/14/repmgr.conf

node_id=2
node_name='d00ps0002'
conninfo='host=192.168.1.156 dbname=repmgr user=repmgr'
data_directory='/var/lib/pgsql/14/data/'
use_replication_slots=true
log_level='INFO'
log_file='/var/log/repmgr/repmgr-14.log'
pg_bindir='/usr/pgsql-14/bin/'


service_start_command = 'sudo systemctl start postgresql-14'
service_stop_command = 'sudo systemctl stop postgresql-14'
service_restart_command = 'sudo systemctl restart postgresql-14'
service_reload_command = 'sudo systemctl reload postgresql-14'

sudo -u postgres /usr/pgsql-14/bin/repmgr -h 192.168.1.155 -U repmgr -d repmgr -f /etc/repmgr/14/repmgr.conf standby clone --dry-run
sudo -u postgres /usr/pgsql-14/bin/repmgr -h 192.168.1.155 -U repmgr -d repmgr -f /etc/repmgr/14/repmgr.conf standby clone

systemctl enable --now postgresql-14
sudo -u postgres /usr/pgsql-14/bin/repmgr -f /etc/repmgr/14/repmgr.conf standby register
```
Create database and check replication
```
CREATE DATABASE TEST1;
\l
\c test1;
\dt
```




switchover

On Both
```
Defaults:postgres !requiretty
%postgres ALL=(ALL)NOPASSWD:/bin/systemctl start postgresql-14,/bin/systemctl stop postgresql-14,/bin/systemctl restart postgresql-14,/bin/systemctl reload postgresql-14

visudo -c
sudo -c postgres
ssh-keygen -t rsa
mkdir -p ~/.ssh
cat /var/lib/pgsql/.ssh/id_rsa.pub > ~/.ssh/authorized_keys
chmod 0700 ~/.ssh
chmod 0600 ~/.ssh/authorized_keys
```
Check switchover

On Slave
```
sudo -u postgres /usr/pgsql-14/bin/repmgr -f /etc/repmgr/14/repmgr.conf standby switchover --siblings-follow --dry-run
sudo -u postgres /usr/pgsql-14/bin/repmgr -f /etc/repmgr/14/repmgr.conf cluster show
```
On Master
```
systemctl stop postgresql-14
```
On Slave
```
sudo -u postgres /usr/pgsql-14/bin/repmgr -f /etc/repmgr/14/repmgr.conf standby promote

CREATE DATABASE TEST2;
```
On Old Master
```
$ sudo -u postgres /usr/pgsql-14/bin/repmgr -f /etc/repmgr/14/repmgr.conf node rejoin --force-rewind -d 'host=192.168.1.156 dbname=repmgr user=repmgr'
```
Security
```
alter user postgres with password 'Pass12word!';
```
