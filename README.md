ansible-role-netfilter
==========================

Role for deploy ipset and iptables (sets and rules).

Ansible versions
--------------------
Role is adapted for Ansible 2.0, should work on 1.9.

Requirements for usage
-----------------------------------

* GNU/Linux with netfilter support;
* iptables-restore with '--test' support;
* iptables/ipset - restore/destroy by init (flawless work with systemd);
* [python-netaddr](//docs.ansible.com/ansible/playbooks_filters_ipaddr.html)
library (on machine with Ansible);

About iptables
-------------------

This is very simple implementation. Yes, can create a dictionary and check the
validity of the ip/cidr/mac, but for large instances one iptables string grows
to ~10 key-value strings. Who can read iptables rules natively, like it. Before
saving to disk, syntax check is performed.

About ipset support
-----------------------

For now support sets with their 'special options':

* bitmap:ip
* bitmap:ip,mac
* bitmap:port
* hash:ip
* hash:mac
* hash:net
* hash:net,iface

Unfortunately, I don't know how to check ipset syntax (is there a way?). Before
deploy ip/cidr/mac will check by python-netaddr library.

Extra
-----------

You can enable/disable iptables or netfilter management by this role via vars:

* **netfilter_iptables_mgmt**, *default is true*
* **netfilter_ipset_mgmt**, *default is true*

Behavior of handlers rule by variables:

* **netfilter_iptables_enable**, *default is false*
* **netfilter_iptables_restart**, *default is false*
* **netfilter_ipset_enable**, *default is false*
* **netfilter_ipset_restart**, *default is false*

**data** field in entry of ip sets - this is field for any kind of ip/cidr/net
addresses.

Example configuration
-------------------------

This example provide all possible options for ipset. Also is describe how to
use iptables tables and chains.

```yaml
---
netfilter_iptables_mgmt: 'true'
netfilter_iptables_enable: 'false'
netfilter_iptables_restart: 'false'
netfilter_ipset_mgmt: 'true'
netfilter_ipset_enable: 'true'
netfilter_ipset_restart: 'true'

netfilter_iptables:
- table: 'mangle'
  chains:
  - name: 'PREROUTING'
    action: 'ACCEPT'
    rules:
    - "PREROUTING -i vlan666 -j MARK --set-xmark 0x1/0xffffffff"
  - name: 'INPUT'
    action: 'ACCEPT'
  - name: 'FORWARD'
    action: 'ACCEPT'
  - name: 'OUTPUT'
    action: 'ACCEPT'
  - name: 'POSTROUTING'
    action: 'ACCEPT'
    rules:
    - "POSTROUTING -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu"
- table: 'filter'
  chains:
  - name: 'INPUT'
    action: 'ACCEPT'
    rules:
    - "INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT"
    - "INPUT -i lo -j ACCEPT"
    - "INPUT -i vlan1998 -j ACCEPT"
    - "INPUT -i vlan1999 -j ACCEPT"
    - "INPUT -i tap+ -j ACCEPT"
    - "INPUT -p udp --dport 15947 -j ACCEPT -m comment --comment \"EXAMPLE COMMENT\""
  - name: 'FORWARD'
    action: 'DROP'
    rules:
    - "FORWARD -i lo -j ACCEPT"
    - "FORWARD -i tap+ -j ACCEPT"
    - "FORWARD -i vlan1998 -s 172.16.14.128/25 -m conntrack --ctstate NEW -j ACCEPT"
    - "FORWARD -i vlan1999 -s 172.16.14.0/25 -m conntrack --ctstate NEW -j ACCEPT"
  - name: 'OUTPUT'
    action: 'ACCEPT'
- table: 'nat'
  chains:
  - name: 'PREROUTING'
    action: 'ACCEPT'
    rules:
    - "PREROUTING -i vlan666 -m string --string \"www.apple.com/library/test/success.html\" --algo kmp --to 65535 -j ACCEPT"
    - "PREROUTING -i vlan666 -m set --match-set BITMAP_IP_MAC src,src -j ACCEPT"
  - name: 'OUTPUT'
    action: 'ACCEPT'
  - name: 'POSTROUTING'
    action: 'ACCEPT'
    rules:
    - "POSTROUTING -s 192.168.2.0/24 -d 172.16.201.0/26,172.16.202.0/26 -j SNAT --to-source 10.9.0.6"
    - "POSTROUTING -s 172.16.66.0/24 -o tap+ -j SNAT --to-source 192.168.2.1"
    - "POSTROUTING -s 172.16.14.0/24 -o vlan100 -j SNAT --to-source 192.168.2.1"

netfilter_ipset:
- set: 'BITMAP_IP'
  type: 'bitmap:ip'
  range: '198.18.0.0/16'
  entry:
  - data: '198.18.0.1/30'
  - data: '198.18.1.1/30'
- set: 'BITMAP_IP_MAC'
  type: 'bitmap:ip,mac'
  range: '198.18.0.0/16'
  timeout: '30'
  counters: 'true'
  comment: 'true'
  skbinfo: 'true'
  entry:
  - data: '198.18.0.5'
    macaddr: '3a:62:6d:97:ab:ee'
    timeout: '15'
    packets: '42'
    bytes: '1024'
    comment: 'Test comment'
    skbmark: '0x1111/0xff00ffff'
    skbprio: '1:10'
    skbqueue: '10'
  - data: '198.18.0.10'
    macaddr: '00:25:22:bf:86:9a'
- set: 'BITMAP_PORT'
  type: 'bitmap:port'
  range: '0-1024'
  entry:
  - port: '80'
  - port: '443'
- set: 'HASH_IP'
  type: 'hash:ip'
  range: '198.18.0.0/16'
  entry:
  - data: '198.18.0.1/24'
  - data: '10.10.10.10/32'
- set: 'HASH_MAC'
  type: 'hash:mac'
  entry:
  - macaddr: '3a:62:6d:97:ab:ee'
  - macaddr: '00:25:22:bf:86:9a'
- set: 'HASH_NET'
  type: 'hash:net'
  entry:
  - data: "240.0.0.0/4"
  - data: "203.0.113.0/24"
  - data: "0.0.0.0/8"
  - data: "127.0.0.0/8"
  - data: "169.254.0.0/16"
  - data: "192.0.2.0/24"
  - data: "198.51.100.0/24"
  - data: "192.168.0.0/16"
  - data: "172.16.0.0/12"
  - data: "10.0.0.0/8"
- set: 'HASH_NET_IFACE'
  type: 'hash:net,iface'
  entry:
  - data: '192.168.0/24'
    iface: 'eth0'
  - data: '10.1.0.0/16'
    iface: 'eth1'

```
