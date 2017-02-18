# HA-fusionpbx
##How to create a highly available fusionpbx cluster on Debian with keepalived postgresql and BDR

1. Create two machines with debian installs. Give both a public and private ip.

###Fusion1
2. Install fusionpbx on one machine, we'll call this one fusion1. BUT, Before running the install script (`./install.sh`) you will want to edit the `/usr/src/fusionpbx-install.sh/debian/resources/postgres.sh` installation file, so you don't have two separate postgres instances. Follow the steps below:

```
apt-get update && apt-get upgrade -y --force-yes
apt-get install -y --force-yes git
cd /usr/src
git clone https://github.com/fusionpbx/fusionpbx-install.sh.git
chmod 755 -R /usr/src/fusionpbx-install.sh
cd /usr/src/fusionpbx-install.sh/debian
```
Now uncomment the "Add PostgreSQL and BDR REPO" section, and comment out the "postgres official repository" like so:
####/usr/src/fusionpbx-install.sh/debian/resources/postgres.sh
```
#!/bin/sh

#send a message
echo "Install PostgreSQL"

#generate a random password
password=$(dd if=/dev/urandom bs=1 count=20 2>/dev/null | base64)

#install message
echo "Install PostgreSQL and create the database and users\n"

#included in the distribution
#apt-get install -y --force-yes sudo postgresql

#postgres official repository
#echo 'deb http://apt.postgresql.org/pub/repos/apt/ jessie-pgdg main' >> /etc/apt/sources.list.d/pgdg.list
#wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -
#apt-get update && apt-get upgrade -y
#apt-get install -y --force-yes sudo postgresql

#Add PostgreSQL and BDR REPO
echo 'deb http://apt.postgresql.org/pub/repos/apt/ jessie-pgdg main'  >> /etc/apt/sources.list.d/postgresql.list
echo 'deb http://packages.2ndquadrant.com/bdr/apt/ jessie-2ndquadrant main' >> /etc/apt/sources.list.d/2ndquadrant.list
/usr/bin/wget --quiet -O - http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | apt-key add -
/usr/bin/wget --quiet -O - http://packages.2ndquadrant.com/bdr/apt/AA7A6805.asc | apt-key add -
apt-get update && apt-get upgrade -y
apt-get install -y --force-yes sudo postgresql-bdr-9.4 postgresql-bdr-9.4-bdr-plugin postgresql-bdr-contrib-9.4

#systemd
systemctl daemon-reload
systemctl restart postgresql

#init.d
#/usr/sbin/service postgresql restart

#move to /tmp to prevent a red herring error when running sudo with psql
cwd=$(pwd)
cd /tmp
#add the databases, users and grant permissions to them
sudo -u postgres psql -c "CREATE DATABASE fusionpbx";
sudo -u postgres psql -c "CREATE DATABASE freeswitch";
sudo -u postgres psql -c "CREATE ROLE fusionpbx WITH SUPERUSER LOGIN PASSWORD '$password';"
sudo -u postgres psql -c "CREATE ROLE freeswitch WITH SUPERUSER LOGIN PASSWORD '$password';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE fusionpbx to fusionpbx;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE freeswitch to fusionpbx;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE freeswitch to freeswitch;"
#ALTER USER fusionpbx WITH PASSWORD 'newpassword';
cd $cwd

#set the ip address
#server_address=$(hostname -I)
```
The BDR repositories and packages are needed for HA. Now do the install (/usr/src/fusionpbx-install.sh/debian/install.sh):

```
./install.sh
```
*Take note of the password you've been given in the output.*


3. Make sure the following packages are installed:
```
sudo apt-get install php5-sqlite libuv-dev flex libtool php5-fpm ssl-cert nginx libjson0-dev php-db libexpat-dev php5-cli sqlite fail2ban lsb-release python-software-properties libpcap-dev ghostscript git-core libjpeg-dev subversion build-essential autoconf automake devscripts gawk g++ make libncurses5-dev python-dev pkg-config libtiff5-dev libldns-dev libperl-dev libgdbm-dev libdb-dev gettext libcurl4-openssl-dev libpcre3-dev libspeex-dev libspeexdsp-dev libsqlite3-dev libedit-dev libpq-dev screen htop bzip2 curl memcached ntp php5-curl php5-imap php5-mcrypt lame time bison libssl-dev unixodbc libmyodbc unixodbc-dev libtiff-tools libmemcached-dev libtool-bin yasm nasm liblua5.2-0 liblua5.2-dev libopus-dev libcodec2-dev libyuv-dev libsndfile-dev libvpx-dev libvpx2-dev postgresql-bdr-9.4 postgresql-bdr-9.4-bdr-plugin postgresql-bdr-contrib-9.4  php5-pgsql tmux csync2 tree keepalived inotify-tools
```

4. Add the following postgres conf files:
###/etc/postgresql/9.4/main/pg_hba.conf
Replace YOURSUBNET with the cidr of your subnet...
```
local   all             postgres                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
#hostssl all             all             YOURSUBNET            trust # FOR SSL
host all             all             YOURSUBNET            trust
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                peer
host    replication     postgres        127.0.0.1/32            md5
#host    replication     postgres        ::1/128                 md5
#hostssl  replication     postgres        YOURSUBNET            trust # FOR SSL
host  replication     postgres        YOURSUBNET              trust
```

###/etc/postgresql/9.4/main/postgres.conf
```
data_directory = '/var/lib/postgresql/9.4/main'        # use data in another directory
                    # (change requires restart)
hba_file = '/etc/postgresql/9.4/main/pg_hba.conf'    # host-based authentication file
                    # (change requires restart)
ident_file = '/etc/postgresql/9.4/main/pg_ident.conf'    # ident configuration file
                    # (change requires restart)

external_pid_file = '/var/run/postgresql/9.4-main.pid'            # write an extra PID file
                    # (change requires restart)
listen_addresses = '*'        # what IP address(es) to listen on;
                    # comma-separated list of addresses;
                    # defaults to 'localhost'; use '*' for all
                    # (change requires restart)
port = 5432                # (change requires restart)
max_connections = 100            # (change requires restart)
unix_socket_directories = '/var/run/postgresql'    # comma-separated list of directories

#ssl = true                # (change requires restart)
                    # (change requires restart)
#ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'        # (change requires restart)
#ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil-postgres.key'        # (change requires restart)

shared_buffers = 128MB            # min 128kB
dynamic_shared_memory_type = posix    # the default is the first option
shared_preload_libraries = 'bdr'    # (change requires restart)

max_worker_processes = 20
wal_level = 'logical'            # minimal, archive, hot_standby, or logical
max_wal_senders = 10        # max number of walsender processes

max_replication_slots = 10    # max number of replication slots
track_commit_timestamp = on    # collect timestamp of transaction commit

log_error_verbosity = default
log_min_messages = warning
log_line_prefix = '%t [%p-%l] %q%u@%d '            # special values:

log_timezone = 'localtime'

stats_temp_directory = '/var/run/postgresql/9.4-main.pg_stat_tmp'

datestyle = 'iso, dmy'
timezone = 'localtime'
default_text_search_config = 'pg_catalog.english'
```

5. Reboot the machine

6. Create the postgresql extensions on both databases:
```
su -l postgres
psql fusionpbx
create extension btree_gist;
create extension pgcrypto;
create extension bdr;
\connect freeswitch;
create extension btree_gist;
create extension pgcrypto;
create extension bdr;
```

7. Create bdr group for fusionpbx and freeswitch databases:
```
su -l postgres
psql fusionpbx
SELECT bdr.bdr_group_create(local_node_name := 'fusion1', node_external_dsn := 'host=NODE1IP port=5432  dbname=fusionpbx connect_timeout=10 keepalives_idle=5 keepalives_interval=1');
\connect freeswitch
SELECT bdr.bdr_group_create(local_node_name := 'fusion1', node_external_dsn := 'host=NODE1IP port=5432  dbname=freeswitch connect_timeout=10 keepalives_idle=5 keepalives_interval=1');
```

###Fusion2

1. Add the necessary repos and keys:
```
echo 'deb http://apt.postgresql.org/pub/repos/apt/ jessie-pgdg main'  >> /etc/apt/sources.list
echo 'deb http://packages.2ndquadrant.com/bdr/apt/ jessie-2ndquadrant main' >> /etc/apt/sources.list
/usr/bin/wget --quiet -O - http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | apt-key add -
/usr/bin/wget --quiet -O - http://packages.2ndquadrant.com/bdr/apt/AA7A6805.asc | apt-key add -
apt-get update && apt-get upgrade -y
```
2. Install the following packages
```
sudo apt-get install postgresql-bdr-9.4 postgresql-bdr-9.4-bdr-plugin postgresql-bdr-contrib-9.4  
```

3. Create and setup the fusionpbx and freeswitch databases for BDR replication:
```
sudo -u postgres psql -c "CREATE DATABASE fusionpbx";
sudo -u postgres psql -c "CREATE DATABASE freeswitch";
sudo -u postgres psql -c "CREATE ROLE fusionpbx WITH SUPERUSER LOGIN PASSWORD 'NODE1DBPASSWORD';"
sudo -u postgres psql -c "CREATE ROLE freeswitch WITH SUPERUSER LOGIN PASSWORD 'NODE1DBPASSWORD';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE fusionpbx to fusionpbx;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE freeswitch to fusionpbx;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE freeswitch to freeswitch;"
```

4. Add the following config files replacing YOURSUBNET with your subnet in cidr notation (same as fusion1):
###/etc/postgresql/9.4/main/postgresql.conf
```
data_directory = '/var/lib/postgresql/9.4/main'        # use data in another directory
                    # (change requires restart)
hba_file = '/etc/postgresql/9.4/main/pg_hba.conf'    # host-based authentication file
                    # (change requires restart)
ident_file = '/etc/postgresql/9.4/main/pg_ident.conf'    # ident configuration file
                    # (change requires restart)

external_pid_file = '/var/run/postgresql/9.4-main.pid'            # write an extra PID file
                    # (change requires restart)
listen_addresses = '*'        # what IP address(es) to listen on;
                    # comma-separated list of addresses;
                    # defaults to 'localhost'; use '*' for all
                    # (change requires restart)
port = 5432                # (change requires restart)
max_connections = 100            # (change requires restart)
unix_socket_directories = '/var/run/postgresql'    # comma-separated list of directories

#ssl = true                # (change requires restart)
                    # (change requires restart)
#ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'        # (change requires restart)
#ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil-postgres.key'        # (change requires restart)

shared_buffers = 128MB            # min 128kB
dynamic_shared_memory_type = posix    # the default is the first option
shared_preload_libraries = 'bdr'    # (change requires restart)

max_worker_processes = 20
wal_level = 'logical'            # minimal, archive, hot_standby, or logical
max_wal_senders = 10        # max number of walsender processes

max_replication_slots = 10    # max number of replication slots
track_commit_timestamp = on    # collect timestamp of transaction commit

log_error_verbosity = default
log_min_messages = warning
log_line_prefix = '%t [%p-%l] %q%u@%d '            # special values:

log_timezone = 'localtime'

stats_temp_directory = '/var/run/postgresql/9.4-main.pg_stat_tmp'

datestyle = 'iso, dmy'
timezone = 'localtime'
default_text_search_config = 'pg_catalog.english'
```
###/etc/postgresql/9.4/main/pg_hba.conf
```
local   all             postgres                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
#hostssl all             all             YOURSUBNET            trust # FOR SSL
host all             all             YOURSUBNET            trust
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                peer
host    replication     postgres        127.0.0.1/32            md5
#host    replication     postgres        ::1/128                 md5
#hostssl  replication     postgres        YOURSUBNET            trust # FOR SSL
host  replication     postgres        YOURSUBNET              trust
```

5. restart postgresql `service postgresql restart`

6. create db extensions for fusionpbx and freeswitch databases:
```
su -l postgres
psql fusionpbx
create extension btree_gist;
create extension pgcrypto;
create extension bdr;
\connect freeswitch;
create extension btree_gist;
create extension pgcrypto;
create extension bdr;
```

6. On fusion2 join to the bdr group you've created on fusion1. NODE1IP is a management ip of node 1, preferrably a private ip
```
su -l postgres
psql fusionpbx
select bdr.bdr_group_join(local_node_name := 'fusion2', node_external_dsn := 'host=NODE2IP port=5432 dbname=fusionpbx connect_timeout=10 keepalives_idle=5 keepalives_interval=1', join_using_dsn := 'host=NODE1IP  port=5432 dbname=fusionpbx connect_timeout=10 keepalives_idle=5 keepalives_interval=1');
\connect freeswitch
select bdr.bdr_group_join(local_node_name := 'fusion2', node_external_dsn := 'host=NODE2IP port=5432 dbname=freeswitch connect_timeout=10 keepalives_idle=5 keepalives_interval=1', join_using_dsn := 'host=NODE1IP  port=5432 dbname=freeswitch connect_timeout=10 keepalives_idle=5 keepalives_interval=1');
```

You should see the fusionpx on fusion2 contains a number of tables after the join, as it has copied the tables from fusion1. If both joins succeeded replication is now happening between both nodes. *You can check to see that the join has succeeded by doing a `select bdr.bdr_node_join_wait_for_ready();` on each database*. If it worked it will return, if it hangs, something has gone wrong.
You can also see the status of the nodes in the group by doing: `select * from bdr.bdr_nodes;`
Active replicating nodes have node status 'r'.
Initializing nodes have node status 'i'.
Dead/killed nodes have node status 'k'.

To remove a node/start over/re-join a node see here TODO

7. Add the freeswitch tables so we can switch from sqlite to postgresql. This can be done on either fusion1 or fusion2 (but not both), as replication is now taking place.
```
cd /tmp
wget https://raw.githubusercontent.com/fusionpbx/fusionpbx/master/resources/install/sql/switch.sql
chmod 755 switch.sql
su -l postgres
psql -U postgres -d freeswitch -f /tmp/switch.sql -L sql.log
```
##TODO

. switch to pg (set switchname, nonlocal bind)

. keepalived
