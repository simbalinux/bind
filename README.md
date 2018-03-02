# BIND DNS Master/Slave Server w/ dynamic updating

This project can be automatically deployed using "git clone" and "vagrant up" at the following repo URL:

https://bitbucket.org/bit-ops/dns/src

## Setup

In this post, we are trying setup one master DNS Server with one Slave DNS Server, we will update records in the Slave DNS Server that will
dynamically update the Master DNS Server using "nsupdate".

DNS clients will Query Slave DNS Server for Name resolution and Master DNS Server is not reachable for Clients, this will verify that replication is
in fact working.

```
Master DNS Server -- ns1.bitfocus.local -- 172.31.1.
Slave DNS Server -- ns2.bitfocus.local -- 172.31.1.
Client Machine -- client.bitfocus.local -- 172.31.1.
```
We are using CentOS7 as DNS Master/Slave DNS Server.

## Installation

Installation of Bind packages on CentOS7 with below command.

```
yum install bind bind-utils
```
** Package installation on Master and Salve DNS servers are same, so above yum install command will work for both DNS Servers.

Below packages installed on my DNS machine.

```
[root@ns1 ~] rpm -qa |grep bind
bind-license-9.9.4-51.el7.noarch
bind-utils-9.9.4-51.el7.x86_
rpcbind-0.2.0-38.el7_3.1.x86_
bind-libs-9.9.4-51.el7.x86_
bind-devel-9.9.4-51.el7.x86_
bind-9.9.4-51.el7.x86_
bind-libs-lite-9.9.4-51.el7.x86_
```
### Configure Master DNS system


```
#add this code below to /etc/dhcp/dhclient.conf on MASTER, EL7 (network
manager) does not permit manipulation of /etc/resolv.conf directly
```
```
interface "eth0" {
supersede domain-name-servers 172.31.1.20, 172.31.1.21, 8.8.8.8;
}
```
```
## create ddns key and copy to /etc
ddns-confgen -r /dev/urandom -z bitfocus.local |grep -i algorithm -A 2
-B1 > /etc/ddns-key.txt
chmod 644 /etc/ddns-key.txt
```
```
## configure firewall rules
```
```
systemctl start firewalld
firewall-cmd --permanent --add-port=53/tcp
firewall-cmd --permanent --add-port=53/udp
firewall-cmd --reload
```
```
## Disable SELINUX
setenforce 0
sed -i 's/SELINUX=\(enforcing\|permissive\)/SELINUX=disabled/g'
/etc/selinux/config
```
```
## set permissions
chgrp named -R /var/named
chown -v root:named /etc/named.conf
restorecon -rv /var/named
restorecon /etc/named.conf
```
```
chown -R named:named /var/named
```
### Configure Master DNS

Master DNS Server. we need to edit named.conf file again with some other derivatives. Change where comments are located.


```
## /etc/named.conf
```
```
options {
listen-on port 53 { 127.0.0.1; 172.31.1.20; }; #IP of MASTER
directory "/var/named";
dump-file "/var/named/data/cache_dump.db";
statistics-file "/var/named/data/named_stats.txt";
memstatistics-file "/var/named/data/named_mem_stats.txt";
allow-query { 172.31.1.20; 172.31.1.21; 127.0.0.1; }; ##IPS of
MASTER - SLAVE - LOCALHOST
```
```
};
```
```
include "/etc/ddns-key.txt"; ##Inlcude path to ddns key
```
```
## These are the mentions of the zone files we are going to create
FORWARD & REV, add everything below this comment.
zone "bitfocus.local" IN {
type master;
file "forward.bitfocus.local";
allow-update { key "ddns-key.bitfocus.local"; }; # the name of this key
is derived from the ddns-key include listed above
};
zone "1.31.172.in-addr.arpa" IN {
type master;
file "reverse.bitfocus.local";
allow-update { key "ddns-key.bitfocus.local"; }; # same as above
ddns-key
};
```
listen-on port 53 — This derivatives used for every DNS server and important as it would mentioned on which Internet protocol address (IP
address) DNS service should listen on machine.

allow-query — Which host could allow to Query this DNS server, This derivative could used in every DNS machines. In Master DNS for security
purpose i only used localhost, own IP and Slave DNS server IP address. Any other then this can’t query Master DNS server. This way we can
isolate Master DNS server from any attack with LAN.

```
Forward lookup Zone
```

#### $TTL 86400

```
@ IN SOA ns1.bitfocus.local. root.bitfocus.local. (
2011071001 ;Serial
3600 ;Refresh
1800 ;Retry
604800 ;Expire
86400 ;Minimum TTL
)
@ IN NS ns1.bitfocus.local.
@ IN NS ns2.bitfocus.local.
@ IN A 172.31.1.
@ IN A 172.31.1.
@ IN A 172.31.1.
ns1 IN A 172.31.1.
ns2 IN A 172.31.1.
client IN A 172.31.1.
```
Reverse lookup zone

#### $TTL 86400

```
@ IN SOA ns1.bitfocus.local. root.bitfocus.local. (
2011071001 ;Serial
3600 ;Refresh
1800 ;Retry
604800 ;Expire
86400 ;Minimum TTL
)
@ IN NS ns1.bitfocus.local.
@ IN NS ns2.bitfocus.local.
@ IN PTR bitfocus.local.
ns1 IN A 172.31.1.
ns2 IN A 172.31.1.
client IN A 172.31.1.
20 IN PTR ns1.bitfocus.local.
21 IN PTR ns2.bitfocus.local.
22 IN PTR client.bitfocus.local.
```
Check ZONE files syntax is correct


```
named-checkzone bitfocus.local /var/named/forward.bitfocus.local
named-checkzone bitfocus.local /var/named/reverse.bitfocus.local
```
```
#if correct you will get no output
```
```
Enable NAMED && Reboot to turn off SELINUX
```
```
# enable named daemon and start
systemctl enable named
systemctl start named
```
```
# reboot machine to turn off SELINUX
```
```
reboot
```
Configuration of Master DNS server complete, let’s start configuring Slave DNS Server.

## Configure Slave DNS Server

Installation part of Slave DNS Server is same as of Master DNS Server. Packages required and installation method is same as of Master DNS
Server.

To configure Slave DNS Server, you will need to edit named.conf file of Slave DNS Server and start named service its should transfer zones file
automatically via dynamic update. Let’s start editing named.conf for Slave DNS Server. Below is named.conf of Slave DNS Server

```
options {
listen-on port 53 { 127.0.0.1; 172.31.1.21; }; #SLAVE DNS
IP
# listen-on-v6 port 53 { ::1; };
directory "/var/named";
dump-file "/var/named/data/cache_dump.db";
```
```
statistics-file "/var/named/data/named_stats.txt";
```
```
memstatistics-file "/var/named/data/named_mem_stats.txt";
```
```
allow-query { 172.31.1.0/24; }; ## IP RANGE ##
```

#### /*

- If you are building an AUTHORITATIVE DNS server, do NOT
enable recursion.
- If you are building a RECURSIVE (caching) DNS server, you
need to enable
recursion.
- If your recursive DNS server has a public IP address, you
MUST enable access
control to limit queries to your legitimate users. Failing to
do so will
cause your server to become part of large scale DNS
amplification
attacks. Implementing BCP38 within your network would greatly

reduce such attack surface
*/
recursion yes;

dnssec-enable yes;
dnssec-validation yes;

/* Path to ISC DLV key */
bindkeys-file "/etc/named.iscdlv.key";

managed-keys-directory "/var/named/dynamic";

pid-file "/run/named/named.pid";

session-keyfile "/run/named/session.key";

};

logging {
channel default_debug {
file "data/named.run";
severity dynamic;
};
};

zone "." IN {
type hint;
file "named.ca";
};

## These are the mentions of the zone files we are going to create
FORWARD & REV
zone "bitfocus.local" IN {


type slave;
file "slaves/bitfocus.fwd";
masters { 172.31.1.20; }; #Master IP
};
zone "1.31.172.in-addr.arpa" IN {
type slave;
file "slaves/bitfocus.rev";
masters { 172.31.1.20; }; #Master IP
};


```
include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```
In this named.conf, we have some different derivatives than Master DNS. Let study them below.

allow-query { 172.31.1.0/24; };

allow-query — This derivative used to query a complete subnet in comparison of Master where we only query for few Host for security purpose.

type slave;

type — This denote as slave as it used to mention slave zone file

masters {172.31.1.21;};

masters — This derivative is only relevant to Slave DNS as it defines Master DNS IP address of particular zone.

### Update DNS dynamically via nsupdate

*copy the ddns-key.txt you created on Master to the Slave as well as an NSUPDATE script.


```
# cp /etc/ddns-key.txt over to slave ~/
```
```
# nsupdate script
cat update_ns1.txt
```
```
server ns1.bitfocus.local
debug yes
zone bitfocus.local
update add test.bitfocus.local 3600 A 172.31.1.
show
send
```
```
#Now lets run nsupdate from NS2 aka SLAVE
```
```
nsupdate -k ddns-key.txt -v nsupdate_ns1.txt
```
```
#check logs on MASTER for the updated records. @NS
```
```
named[2209]: client 172.31.1.21/43630/key ddns-key.bitfocus.local:
signer "ddns-key.bitfocus.local" approved
named[2209]: client 172.31.1.21/43630/key ddns-key.bitfocus.local:
updating zone 'bitfocus.local/IN': adding an RR at 'test.bitfocus.local'
A
```
```
# Master will have created a temporary .jnl forward file in /var/named/
```
```
[root@ns1 ~] ls -al /var/named |grep jnl
-rw-r--r-- 1 named named 745 Nov 29 16:10 forward.bitfocus.local.jnl
```
```
#lets verify that MASTER was updated on MASTER
[root@ns1 ~] dig +short test.bitfocus.local
172.31.1.
```
```
#verify that NS1 is checked first
[root@ns1 ~] cat /etc/resolv.conf
# Generated by NetworkManager
search bitfocus.local
nameserver 172.31.1.
nameserver 172.31.1.
nameserver 8.8.8.
```
### Configuration of DNS Clients

For Linux DNS Client, we need to mention DNS Server IP address in /etc/resolv.conf, remember network manager is an issue so do not edit


/etc/resolv.conf directly!!

copy these contents into /etc/dhcdp/dhclient.conf on the client node ensuring to only reach NS2 not Master. Matching your interface name

```
interface "eth0" {
supersede domain-name-servers 172.31.1.21, 8.8.8.8;
}
```
Finally ensure that the client can see the MASTERS dynamically updated RR using our SLAVE query.

```
[vagrant@client ~]$ dig +short test.bitfocus.local && cat
/etc/resolv.conf
172.31.1.
# Generated by NetworkManager
search bitfocus.local
nameserver 172.31.1.
```
Final conclusion:
To update ZONE files simply ssh > SLAVE NS2, from there edit the update_ns1.txt file to you're liking. Then issue a "nsupdate -k ddns-key.txt -v
update_ns1.txt" and NS1/Master will be updated and the serial will be automatically updated and all HOSTS in the network will have updated
ZONE files for DNS.



