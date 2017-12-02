## How to configure hidden PowerDNS primary on FreeBSD jail and pf rules:

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

#### FreeBSD sysctl and loader.conf

I use this /boot/loader.conf in most cases:

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

