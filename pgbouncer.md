**Install pgbouncer**
```
sudo apt-get -y install pgbouncer
```
/etc/pgbouncer/pgbouncer.ini
```
[databases]

* = host=localhost port=5432 # change

[pgbouncer]

logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid

listen_addr = * # change
listen_port = 6432

unix_socket_dir = /var/run/postgresql

auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

max_client_conn = 1000 # change for test

default_pool_size = 20
```
```
su - postgres
ALTER SYSTEM SET max_connections = 500;
ALTER SYSTEM SET shared_buffers = '512MB';

createuser -P test
createdb test_db -O test

echo -n '<password>' | md5sum
```
/etc/pgbouncer/userlist.txt
```
“<username>” “md5<password>”
```
```
systemctl enable --now pgbouncer
systemctl restart pgbouncer
```
```
pgbench -p 5432 -i -U test -h localhost test_db

pgbench -p 5432 -c 100 -j 16 -T 60 -U test -h localhost test_db
ps aux | grep test | cat -b

pgbench -p 6432 -c 100 -j 16 -T 60 -U test -h localhost test_db
ps aux | grep test | cat -b
```
