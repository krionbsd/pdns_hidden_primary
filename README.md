## How to configure hidden PowerDNS primary on FreeBSD jail with pf rules:

When I start any server installation powered by FreeBSD, I usually
start with poudriere repo to fetch packages from my central
poudriere repo.  I prefer building from central repo and distribute
packages on another servers.  All my servers are dedicated Hetzner
servers running on FreeBSD with ZFS, I'll try to review ZFS in
another article to show you how easy it is to understand, to
maintain and backup your servers.

Additionally I'm using some tweaks to boost server
networking performance.  Let's start with them and kernel modules
first.

#### FreeBSD /etc/sysctl and /boot/loader.conf

I use this `/boot/loader.conf` in most cases:

```
zfs_load="YES"
autoboot_delay="3"
coretemp_load="YES"
aesni_load="YES"
aio_load="YES"
tmpfs_load="YES"
pf_load="YES"
pflog_load="YES"
cc_htcp_load="YES"
accf_http_load="YES"
accf_data_load="YES"
accf_dns_load="YES"
net.inet.tcp.hostcache.cachelimit="0"
net.link.ifqmaxlen="2048"
net.inet.tcp.soreceive_stream="1"
```

Here is my `/etc/sysctl.conf`

```
vfs.read_max="128"
net.inet.tcp.sendspace=262144  # (default 32768)
net.inet.tcp.recvspace=262144  # (default 65536)
net.inet.tcp.sendbuf_max=16777216
net.inet.tcp.recvbuf_max=16777216
net.inet.tcp.sendbuf_inc=32768  # (default 8192)
net.inet.tcp.recvbuf_inc=65536  # (default 16384)
net.local.stream.sendspace=16384  # default (8192)
net.local.stream.recvspace=16384  # default (8192)
net.inet.raw.maxdgram=16384
net.inet.raw.recvspace=16384
net.inet.tcp.abc_l_var=44          # (default 2)
net.inet.tcp.initcwnd_segments=44  # (default 10)
net.inet.tcp.mssdflt=1448  # (default 536)
net.inet.tcp.minmss=524  # (default 216)
net.inet.tcp.cc.algorithm=htcp
net.inet.tcp.cc.htcp.adaptive_backoff=1
net.inet.tcp.cc.htcp.rtt_scaling=1
net.inet.tcp.rfc6675_pipe=1
net.inet.tcp.syncookies=0
net.inet.tcp.nolocaltimewait=1
net.inet.tcp.tso=0
net.inet.ip.intr_queue_maxlen=2048  # (default 256)
net.route.netisr_maxqlen=2048       # (default 256)
```

Please note, I'm using only FreeBSD releases on production servers and update them only for security patches with

```
freebsd-update fetch
freebsd-update install

```
I always avoid building sources on production servers as well as CURRENT and STABLE, but I think everyone does the same.

#### Poudriere:

I really like using poudriere, and personally I use it on every
server, I've created environment with one central server which
builds packages every night and distribute them across another
servers. So let's configure it first, I'm describing here only
client part not a server one:

```
sudo mkdir -p /usr/local/etc/ssl/{keys,certs}
ssh krion@our_poudriere_server 'cat /usr/local/etc/ssl/certs/poudriere.cert' | sudo tee /usr/local/etc/ssl/certs/poudriere.cert
sudo mkdir -p /usr/local/etc/pkg/repos
sudo vi /usr/local/etc/pkg/repos/poudriere.conf

poudriere: {
    url: "http://our_poudriere_server/packages/11amd64-default/",
    mirror_type: "https",
    signature_type: "pubkey",
    pubkey: "/usr/local/etc/ssl/certs/poudriere.cert",
    enabled: yes,
}
```
Disable official FreeBSD pkg repo and use only ours:
```
sudo vi /usr/local/etc/pkg/repos/freebsd.conf

FreeBSD: {
    enabled: no
}
```
So we're done.  We can build the packages on central poudriere
server, fetch them and install them locally on our servers, eg:
```
pkg install vim
```

How to install and use central poudriere server is described here:

* https://www.freebsd.org/doc/handbook/ports-poudriere.html
* https://github.com/freebsd/poudriere/wiki

#### Installing jail (iocage) and pf rules configuration:



#### PowerDNS auth installation and configuration:

We've to build PowerDNS auth server first and choose OPTIONS which
are required for you.  I always enable sqlite3 (for DNSSEC) and
protobuf support (starting from powerdns-4.1.0 protobuf is enabled
by default in ports):
```
_FILE_COMPLETE_OPTIONS_LIST=GEOIP MYDNS MYSQL OPENDBX OPENLDAP
OPTALGO PGSQL PROTOBUF REMOTE SQLITE3 TINYDNS TOOLS
UNIXODBC LUA LUAJIT LUABACKEND ZEROMQ
OPTIONS_FILE_UNSET+=GEOIP
OPTIONS_FILE_UNSET+=MYDNS
OPTIONS_FILE_SET+=MYSQL
OPTIONS_FILE_UNSET+=OPENDBX
OPTIONS_FILE_UNSET+=OPENLDAP
OPTIONS_FILE_SET+=OPTALGO
OPTIONS_FILE_SET+=PGSQL
OPTIONS_FILE_UNSET+=PROTOBUF
OPTIONS_FILE_UNSET+=REMOTE
OPTIONS_FILE_SET+=SQLITE3
OPTIONS_FILE_UNSET+=TINYDNS
OPTIONS_FILE_SET+=TOOLS
OPTIONS_FILE_UNSET+=UNIXODBC
OPTIONS_FILE_SET+=LUA
OPTIONS_FILE_UNSET+=LUAJIT
OPTIONS_FILE_UNSET+=LUABACKEND
OPTIONS_FILE_UNSET+=ZEROMQ
```

Login into your jail `iocage console $jail_name` and install
PowerDNS auth which you built on poudriere central pkg server with
`pkg install pdns`.  Your configuration directory is located in 
`/usr/local/etc/pdns/`
PowerDNS auth configuration is really easy and straightforward, if
you have DNS experience, you will be surprised how intuitive and
simple configuration is.  One of the best PowerDNS features is its
ability to work with various backends like MySQL, PostgreSQL, SQLite
etc., but for old farts like me, nothing is better and intuitive
than old warm Bind zones configurations, so I'll use them in my
example. 

```
/usr/local/etc/pdns/pdns.conf:

allow-axfr-ips=$IP_of_your_official_auth_ipv4, $IP_of_your_official_auth_ipv6
disable-axfr=no
log-dns-details=on

bind-check-interval=300
launch=bind
bind-config=/usr/local/etc/pdns/bindbackend.conf
local-address=172.16.1.1 # Jail IP
master=yes
version-string=anonymous

bind-dnssec-db=/usr/local/etc/pdns/db/bind-dnssec-db.sqlite3
```

And that's it.  Too easy, isn't it?  Now we need to configure our bind
zones.
```
/usr/local/etc/pdns/bindbackend.conf:

zone "example1.com" { type master; file "/usr/local/etc/pdns/zones/example1.com.zone"; };
zone "example1_IP.in-addr.arpa" { type master; file "/usr/local/etc/pdns/zones/rev.example1.com"; };
zone "example2.com" { type master; file "/usr/local/etc/pdns/zones/example2.com.zone"; };
```
