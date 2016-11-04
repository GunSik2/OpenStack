
## Overview
- Node & Service Deployment
  - Node1 - Controller : Horizon, Neutron, Keystone, Glance, Heat
  - Node2 - Compute01 : Nova, Swift, Cinder
  - Node3 - Compute02 : Nova, Swift, Cinder
  - Node4 - Compute04 : Nova, Swift, Cinder
- Networking 
  - [Self-service networks](http://docs.openstack.org/mitaka/install-guide-ubuntu/overview.html)
    - provider network : public network
    - management network : private network
    
## Environment
- Security 
```
echo "ADMIN_PASS=$(openssl rand -hex 10)
CEILOMETER_DBPASS=$(openssl rand -hex 10)
CEILOMETER_PASS=$(openssl rand -hex 10)
CINDER_DBPASS=$(openssl rand -hex 10)
CINDER_PASS=$(openssl rand -hex 10)
DASH_DBPASS=$(openssl rand -hex 10)
DEMO_PASS=$(openssl rand -hex 10)
GLANCE_DBPASS=$(openssl rand -hex 10)
GLANCE_PASS=$(openssl rand -hex 10)
HEAT_DBPASS=$(openssl rand -hex 10)
HEAT_DOMAIN_PASS=$(openssl rand -hex 10)
HEAT_PASS=$(openssl rand -hex 10)
KEYSTONE_DBPASS=$(openssl rand -hex 10)
NEUTRON_DBPASS=$(openssl rand -hex 10)
NEUTRON_PASS=$(openssl rand -hex 10)
NOVA_DBPASS=$(openssl rand -hex 10)
NOVA_PASS=$(openssl rand -hex 10)
RABBIT_PASS=$(openssl rand -hex 10)
SWIFT_PASS=$(openssl rand -hex 10)" > password.sh
```
- Host networking
  - All nodes have two interfaces, provider & management interfaces.
    - Management on 10.0.0.0/24 with gateway 10.0.0.1
    - Provider on 203.0.113.0/24 with gateway 203.0.113.1
- NTP
  - Controller node
```
# apt-get install chrony
# /etc/chrony/chrony.conf
server NTP_SERVER iburst
# service chrony restart

# chronyc sources
```
  - Other nodes
```
# apt-get install chrony
# /etc/chrony/chrony.conf
server controller iburst
# service chrony restart

# chronyc sources
```
- OpenStack packages on all nodes
```
# apt-get install software-properties-common
# add-apt-repository cloud-archive:mitaka
# apt-get update && apt-get dist-upgrade
# apt-get install python-openstackclient
```
- SQL database on controller node
```
# apt-get install mariadb-server python-pymysql
# vi /etc/mysql/conf.d/openstack.cnf
[mysqld]
...
bind-address = 10.0.0.11   # Management interface ip
default-storage-engine = innodb
innodb_file_per_table
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8

# service mysql restart
# mysql_secure_installation
```
- NoSQL database on controller node (optional)
  - The installation of the NoSQL database server is only necessary when installing the Telemetry service.
```
# apt-get install mongodb-server mongodb-clients python-pymongo
# vi /etc/mongodb.conf
bind_ip = 10.0.0.11
smallfiles = true

# service mongodb stop
# rm /var/lib/mongodb/journal/prealloc.*
# service mongodb start
```
- Message queue on controller node
```
# apt-get install rabbitmq-server
# rabbitmqctl add_user openstack $RABBIT_PASS
# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```
- Memcached on controller node
  - The Identity service authentication mechanism for services uses Memcached to cache tokens.
```
# apt-get install memcached python-memcache
# vi /etc/memcached.conf
-l 10.0.0.11   # management inteface ip
# service memcached restart
```

## Identity Service
- Overview
  - The Identity service contains these components:
    - Server : A centralized server provides authentication and authorization services using a RESTful interface
    - Drivers : Drivers or a service back end  (for example, SQL databases or LDAP servers).
    - Modules : Modules intercept service requests, extract user credentials, and send them to the centralized server for authorization
- Install & Configure
  - Prerequisites
```
$ mysql -u root -p
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
  IDENTIFIED BY '$KEYSTONE_DBPASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
  IDENTIFIED BY '$KEYSTONE_DBPASS';
```
  - Install and configure components
````
# echo "manual" > /etc/init/keystone.override
# apt-get install keystone apache2 libapache2-mod-wsgi
# vi /etc/keystone/keystone.conf
[DEFAULT]
...
admin_token = $ADMIN_TOKEN

[database]
...
connection = mysql+pymysql://keystone:$KEYSTONE_DBPASS@controller/keystone

[token]
...
provider = fernet

# su -s /bin/sh -c "keystone-manage db_sync" keystone
# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
```
  - Configure the Apache HTTP server 
```
vi /etc/apache2/apache2.con
ServerName controller

vi /etc/apache2/sites-available/wsgi-keystone.conf
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined

    <Directory /usr/bin>
        Require all granted
    </Directory>
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined

    <Directory /usr/bin>
        Require all granted
    </Directory>
</VirtualHost>

# ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled
# service apache2 restart
# rm -f /var/lib/keystone/keystone.db  
```
- Create the service entity and API endpoints
  - Prerequisites
```
$ export OS_TOKEN=ADMIN_TOKEN
$ export OS_URL=http://controller:35357/v3
$ export OS_IDENTITY_API_VERSION=3
```
  - Create the service entity and API endpoints
```
$ openstack service create \
  --name keystone --description "OpenStack Identity" identity
$ openstack endpoint create --region RegionOne \
  identity public http://controller:5000/v3  
$ openstack endpoint create --region RegionOne \
  identity internal http://controller:5000/v3
$ openstack endpoint create --region RegionOne \
  identity admin http://controller:35357/v3  
```
- Create a domain, projects, users, and roles
```
$ openstack domain create --description "Default Domain" default
$ openstack project create --domain default \
  --description "Admin Project" admin

$ openstack user create --domain default \
  --password-prompt admin
$ openstack role create admin
$ openstack role add --project admin --user admin admin
$ openstack project create --domain default \
  --description "Service Project" service
  
$ openstack user create --domain default \
  --password-prompt demo
$ openstack role create user  
$ openstack role add --project demo --user demo user 
$ openstack project create --domain default \
  --description "Demo Project" demo
```
- Verify operation
```
$ unset OS_TOKEN OS_URL
$ openstack --os-auth-url http://controller:35357/v3 \
  --os-project-domain-name default --os-user-domain-name default \
  --os-project-name admin --os-username admin token issue
Password:
+------------+-----------------------------------------------------------------+
| Field      | Value                                                           |
+------------+-----------------------------------------------------------------+
| expires    | 2016-02-12T20:14:07.056119Z                                     |
| id         | gAAAAABWvi7_B8kKQD9wdXac8MoZiQldmjEO643d-e_j-XXq9AmIegIbA7UHGPv |
|            | atnN21qtOMjCFWX7BReJEQnVOAj3nclRQgAYRsfSU_MrsuWb4EDtnjU7HEpoBb4 |
|            | o6ozsA_NmFWEpLeKy0uNn_WeKbAhYygrsmQGA49dclHVnz-OMVLiyM9ws       |
| project_id | 343d245e850143a096806dfaefa9afdc                                |
| user_id    | ac3377633149401296f6c0d92d79dc16                                |
+------------+-----------------------------------------------------------------+ 
```
- Create OpenStack client environment scripts
  - Creating the scripts
```
# vi admin-openrc
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=$ADMIN_PASS
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2

# vi demo-openrc
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=$DEMO_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
  - Using the scripts
```
$ . admin-openrc
$ openstack token issue
```

## Image service
- Overview
  - The Image service includes the following components:
    - glance-api : Accepts Image API calls for image discovery, retrieval, and storage.
    - glance-registry : Stores, processes, and retrieves metadata about images. (A private internal service)
    - Database : Stores image metadata 
    - Storage repository for image files :  Supported including normal file systems, Object Storage, RADOS block devices, HTTP, and Amazon S3
    - Metadata definition service : A common API to define custom metadata
- Install and configure
  - Prerequisites
```
$ mysql -u root -p
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'GLANCE_DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'GLANCE_DBPASS';
$ . admin-openrc
$ openstack user create --domain default --password-prompt glance
$ openstack role add --project service --user glance admin
$ openstack service create --name glance \
  --description "OpenStack Image" image
$ openstack endpoint create --region RegionOne \
  image public http://controller:9292  
$ openstack endpoint create --region RegionOne \
  image internal http://controller:9292
$ openstack endpoint create --region RegionOne \
  image admin http://controller:9292  
```
  - Install and configure components : GLANCE_DBPASS, GLANCE_PASS
```
# apt-get install glance
# vi /etc/glance/glance-api.conf 
[database]
...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
...
flavor = keystone
[glance_store]
...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

# vi /etc/glance/glance-registry.conf
[database]
...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
...
flavor = keystone

# su -s /bin/sh -c "glance-manage db_sync" glance
# service glance-registry restart
# service glance-api restart
```
- Verify operation
```
$ . admin-openrc
$ wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
$ openstack image create "cirros" \
  --file cirros-0.3.4-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public
$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 38047887-61a7-41ea-9b49-27987d5e8bb9 | cirros | active |
+--------------------------------------+--------+--------+
```
## Compute service
- Overview
  - Use OpenStack Compute to host and manage cloud computing systems. 
  - OpenStack Compute interacts with 
    - OpenStack Identity for authentication; 
    - OpenStack Image service for disk and server images; and 
    - OpenStack dashboard for the user and administrative interface.
  - OpenStack Compute consists of the following areas and their components:
    - Controller node
      - nova-api service : Accepts and responds to end user compute API calls. 
      - nova-conductor module : Mediates interactions between the nova-compute service and the database. 
      - nova-consoleauth daemon : Authorizes tokens for users that console proxies (nova-novncproxy) provide. 
      - nova-novncproxy daemon : Provides a proxy for accessing running instances through a VNC connection. Supports browser-based novnc clients.
      - nova-scheduler service : Takes a virtual machine instance request from the queue and determines on which compute server host it runs.
      - The queue
      - SQL database
    - Compute nodes
      - nova-compute service : A worker daemon that creates and terminates virtual machine instances through hypervisor APIs.       
    - Etc  
      - nova-api-metadata service:  Used when you run in multi-host mode with nova-network installations.
      - nova-network worker daemon : Similar to the nova-compute service 
      - nova-cert module : A server daemon that serves the Nova Cert service for X509 certificates.
      - nova-spicehtml5proxy daemon : Provides a proxy for accessing running instances through a SPICE connection. Supports browser-based HTML5 client.
      - nova-xvpvncproxy daemon : Provides a proxy for accessing running instances through a VNC connection. Supports an OpenStack-specific Java client.
      - nova-cert daemon : x509 certificates.
      - nova client : Enables users to submit commands as a tenant administrator or end user.
-Install and configure controller node
  - Prerequisites : NOVA_DBPASS 
```
$ mysql -u root -p
CREATE DATABASE nova_api;
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY 'NOVA_DBPASS';
  
$ . admin-openrc  
$ openstack user create --domain default \
  --password-prompt nova
$ openstack role add --project service --user nova admin
$ openstack service create --name nova \
  --description "OpenStack Compute" compute
$ openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1/%\(tenant_id\)s
$ openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1/%\(tenant_id\)s  
$ openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1/%\(tenant_id\)s  
```
  - Install and configure components : NOVA_DBPASS, RABBIT_PASS
```
# apt-get install nova-api nova-conductor nova-consoleauth \
  nova-novncproxy nova-scheduler
# sudo vi /etc/nova/nova.conf
[DEFAULT]
enabled_apis = osapi_compute,metadata
rpc_backend = rabbit
auth_strategy = keystone
my_ip = 10.0.0.11   # management interface IP
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

[database]
connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = NOVA_PASS

[vnc]
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp


# su -s /bin/sh -c "nova-manage api_db sync" nova
# su -s /bin/sh -c "nova-manage db sync" nova
# service nova-api restart
# service nova-consoleauth restart
# service nova-scheduler restart
# service nova-conductor restart
# service nova-novncproxy restart
```
-Install and configure a compute node
  - Install and configure components : RABBIT_PASS, NOVA_PASS, MANAGEMENT_INTERFACE_IP_ADDRESS
```
# apt-get install nova-compute
# sudo vi /etc/nova/nova.conf
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone
my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[oslo_messaging_rabbit]
...
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = NOVA_PASS

[vnc]
...
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html

[glance]
...
api_servers = http://controller:9292

[oslo_concurrency]
...
lock_path = /var/lib/nova/tmp


$ egrep -c '(vmx|svm)' /proc/cpuinfo
## If this command returns a value of one or greater, your compute node supports hardware acceleration which typically requires no additional configuration.
## If this command returns a value of zero, your compute node does not support hardware acceleration and you must configure libvirt to use QEMU instead of KVM.  


# service nova-compute restart
```
- Verify operation 
```
$ . admin-openrc
$ openstack compute service list
+----+--------------------+------------+----------+---------+-------+----------------------------+
| Id | Binary             | Host       | Zone     | Status  | State | Updated At                 |
+----+--------------------+------------+----------+---------+-------+----------------------------+
|  1 | nova-consoleauth   | controller | internal | enabled | up    | 2016-02-09T23:11:15.000000 |
|  2 | nova-scheduler     | controller | internal | enabled | up    | 2016-02-09T23:11:15.000000 |
|  3 | nova-conductor     | controller | internal | enabled | up    | 2016-02-09T23:11:16.000000 |
|  4 | nova-compute       | compute1   | nova     | enabled | up    | 2016-02-09T23:11:20.000000 |
+----+--------------------+------------+----------+---------+-------+----------------------------+
```


## Networking service 
- Overview
- Install and configure
  - Prerequisites
  - Install and configure components
- Verify operation

## Dashboard
- Overview
- Install and configure
  - Prerequisites
  - Install and configure components
- Verify operation

## Block Storage service
- Overview
- Install and configure
  - Prerequisites
  - Install and configure components
- Verify operation

## Shared File Systems service
- Overview
- Install and configure
  - Prerequisites
  - Install and configure components
- Verify operation

## Object Storage service
- Overview
- Install and configure
  - Prerequisites
  - Install and configure components
- Verify operation

## Launch an instance
- Overview
- Install and configure
  - Prerequisites
  - Install and configure components
- Verify operation


Reference
- http://docs.openstack.org/mitaka/install-guide-ubuntu/common/conventions.html
