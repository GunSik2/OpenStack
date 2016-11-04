
## Overview
- Node & Service Deployment
  - Node1 - Controller : Horizon, Neutron, Keystone, Glance, Heat
  - Node2 - Compute01 : Nova, Swift, Cinder
  - Node3 - Compute02 
  - Node4 - Compute04
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


Reference
- http://docs.openstack.org/mitaka/install-guide-ubuntu/common/conventions.html
