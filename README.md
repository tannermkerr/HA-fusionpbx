# HA-fusionpbx
##How to create a highly available fusionpbx cluster on Debian with keepalived postgresql and BDR

1. Create two machines with debian installs. Give both a public and private ip. Steps 1 - 6 need to be done on both fusion1 and fusion2.

2. Install fusionpbx on two separate machines with commands listed here:
https://fusionpbx.com/app/www/download.php
Before running the install script you will want to edit the `/usr/src/fusionpbx-install.sh/debian/resources/postgres.sh` installation file, so you don't have two separate postgres instances. You will want to uncomment the "Add PostgreSQL and BDR REPO" section and comment out the "postgres official repository". The BDR repositories and packages are needed for HA.

<!-- 3. Add the Debian BDR repository to your sources.list to download the required packages:
deb http://packages.2ndquadrant.com/bdr/apt/ wheezy-2ndquadrant main

4. import the repository key:
```
wget --quiet -O - http://packages.2ndquadrant.com/bdr/apt/AA7A6805.asc | sudo apt-key add -
sudo apt-get update
``` -->

3. Make sure the following packages are installed:
`sudo apt-get install php5-sqlite libuv-dev flex libtool php5-fpm ssl-cert nginx libjson0-dev php-db libexpat-dev php5-cli sqlite fail2ban lsb-release python-software-properties libpcap-dev ghostscript git-core libjpeg-dev subversion build-essential autoconf automake devscripts gawk g++ make libncurses5-dev python-dev pkg-config libtiff5-dev libldns-dev libperl-dev libgdbm-dev libdb-dev gettext libcurl4-openssl-dev libpcre3-dev libspeex-dev libspeexdsp-dev libsqlite3-dev libedit-dev libpq-dev screen htop bzip2 curl memcached ntp php5-curl php5-imap php5-mcrypt lame time bison libssl-dev unixodbc libmyodbc unixodbc-dev libtiff-tools libmemcached-dev libtool-bin yasm nasm liblua5.2-0 liblua5.2-dev libopus-dev libcodec2-dev libyuv-dev libsndfile-dev libvpx-dev libvpx2-dev postgresql-bdr-9.4 postgresql-bdr-9.4-bdr-plugin php5-pgsql tmux csync2 tree keepalived inotify-tools`

4. Add the following postgres conf files:
#/etc/postgresql/9.4/main/pg_hba.conf
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
host  replication     postgres        YOURSUBNET   
```

#/etc/postgresql/9.4/main/postgres.conf
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

5. reboot the machines

6. create the postgresql extensions on both databases:
```
su -l postgres
psql fusionpbx
create extension btree_gist;
create extension pgcrypto;
create extension bdr;
create extension uuid-ossp;
\connect freeswitch;
create extension btree_gist;
create extension pgcrypto;
create extension bdr;
```

7. On fusion1 create bdr group for fusionpbx and freeswitch databases:

8. On fusion2 join to bdr group

9. add freeswitch db tables

10. switch to pg
