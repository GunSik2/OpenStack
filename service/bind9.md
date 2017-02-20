

```
sudo apt-get update
sudo apt-get install bind9 bind9utils
```

```
vi /etc/bind/named.conf.options
options {
        directory "/var/cache/bind";
        recursion yes;                 # enables resursive queries
        #allow-recursion { trusted; };  # allows recursive queries from "trusted" clients
        listen-on { 192.168.10.3; };   # ns1 private IP address - listen on private network only
        allow-transfer { none; };      # disable zone transfers by default

        forwarders {
                8.8.8.8;
                8.8.4.4;
        };
};

```

```
vi /etc/bind/named.conf.local
zone "medical.paasxpert.com" {
    type master;
    file "/etc/bind/zones/medical.paasxpert.com"; # zone file path
    #allow-transfer { 10.128.20.12; };           # ns2 private IP address - secondary
};


sudo mkdir /etc/bind/zones
sudo cp /etc/bind/db.local /etc/bind/zones/medical.paasxpert.com

```

```
vi /etc/bind/zones/medical.paasxpert.com
$TTL    604800
@       IN      SOA     ns.medical.paasxpert.com. root.medical.paasxpert.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
        IN      NS      ns.medical.paasxpert.com.

ns.medical.paasxpert.com.  IN A  172.16.0.132
api.medical.paasxpert.com. IN A  172.16.0.131
*.medical.paasxpert.com.   IN A  172.16.0.131

```

```
 tail -f /var/log/syslog &
 sudo /etc/init.d/bind9 restart
 
  dig api.medical.paasxpert.com @192.168.10.3
```
